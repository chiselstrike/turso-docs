---
description: Setup guide for configuring Turso for deployment using Fermyon Spin and Rust.
keywords:
  - turso
  - tutorial
  - guide
  - fermyon
  - spin
---

# Fermyon Spin with Rust setup guide

In this setup guide, you will deploy a simple route component to Fermyon Spin
that queries a [Turso] database. The deployment is configured with variables
whose values are obtained from the [Turso CLI].

## Prerequisites

- A [Fermyon Cloud account]
- The Turso CLI installed on your machine ([installation instructions])
- Rust toolchain installed [using rustup] (Homebrew installation will not work)
- Spin CLI installed
- Familiarity with Spin is suggested ([follow the Spin quickstart])

Check that the turso CLI is installed:

```
$ turso --version
```

Check that rust is installed using rustup:

```
$ rustup --version
rustup 1.26.0 (5af9b9484 2023-04-05)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.70.0 (90c541806 2023-05-31)`
```

You must have the wasm32-wasi toolchain support installed:

```
$ rustup target add wasm32-wasi

$ rustup target list | grep wasm32-wasi
wasm32-wasi (installed)
```

Check that the Spin CLI is installed:

```
$ spin --version
spin 1.3.0 (9fb8256 2023-06-12)
```

Make sure you are already logged in to the CLI using `spin login`.


## 1. Create a Turso database

Run the following Turso CLI command to create a new database named "turso-fermyon":

```bash
$ turso db create turso-fermyon
```

This tutorial does not cover populating or querying tables in the database.
Instead, you will simply run a query that requires no tables.

## 2. Deploy a new spin app using the Rust template

### 2a. Create a new app using a template for Rust

In a shell, change to a directory where you want to create a project. The Spin
CLI will create another directory under it.

Run the following Spin CLI command to create a new project using a template:

```bash
$ spin new
```

1. Select the "http-rust" template.
1. When prompted, give the new application the name "turso-fermyon".
1. Take all the other defaults by pressing enter at each prompt.

```
Pick a template to start your application with: http-rust (HTTP request handler using Rust)
Enter a name for your new application: turso-fermyon
Description:
HTTP base: /
HTTP path: /...
```

Spin creates a new directory with the in the current directory with the same
name that you gave the application.

Open `src/lib.rs` in a code editor. This contains minimal code to respond to an
HTTP request at the route specified in `spin.toml`. It responds with the text
"Hello, Fermyon".

```rust
#[http_component]
fn handle_turso_fermyon(req: Request) -> Result<Response> {
    println!("{:?}", req.headers());
    Ok(http::Response::builder()
        .status(200)
        .header("foo", "bar")
        .body(Some("Hello, Fermyon".into()))?)
}
```

### 2b. Build the template app

Build the template application as-is without any modifications to verify that
it works.

```bash
$ cd turso-fermyon
$ spin build
...
Finished building all Spin components
```

The build process creates a file
`target/wasm32-wasi/release/turso_fermyon.wasm`. This file is referenced in the
project `spin.toml` as the component source to deploy.

### 2c. Run the template app locally

Run the template app:

```bash
$ spin up
```
```
Logging component stdio to ".spin/logs/"

Serving http://127.0.0.1:3000
Available Routes:
  turso-fermyon: http://127.0.0.1:3000 (wildcard)
```

The output contains a local URL you can use to invoke the route. Copy the URL
and use curl to access it:

```bash
$ curl http://127.0.0.1:3000
```
```
Hello, Fermyon
```

It outputs the string "Hello, Fermyon".

Stop the local Spin server with ctrl-C.

### 2c. Deploy the template app to Fermyon Cloud

Deploy the template app:

```bash
$ spin deploy
```

```
Uploading turso-fermyon version 0.1.0+rb01010f5...
Deploying...
Waiting for application to become ready...... ready
Available Routes:
  turso-fermyon: https://turso-fermyon-[RANDOM-CHARS].fermyon.app (wildcard)
```

The output contains a remote URL you can use to invoke the route. It has random
characters that are unique to your deployment. Copy the URL and use curl to
access it:

```
$ curl https://turso-fermyon-[RANDOM-CHARS].fermyon.app
```
```
Hello, Fermyon
```

## 3. Add the libSQL Rust SDK

Now that you've verified that you can run and deploy the application, the next
step is to change the code add the libSQL client SDK for Rust used to query your
Turso database.

The [libsql-client crate] requires the feature `spin_backend` at installation,
and no default features. Run the following command:

```bash
$ cargo add \
    --git https://github.com/libsql/libsql-client-rs \
    --no-default-features -F spin_backend
```

This adds the following line to your `Cargo.toml` in the `[dependencies]` table:

```toml
libsql-client = { git = "https://github.com/libsql/libsql-client-rs", version = "0.31.4", default-features = false, features = ["spin_backend"] }
```

:::note

You must install the client using the GitHub URL. The Spin SDK doesn't support
installation of crates from crates.io as you would normally with a Rust program.

:::

### 3a. Add configuration for Turso in spin.toml

You can use [Spin Application Variables and Secrets] to configure values in a
Spin app.

Add the following lines to `spin.toml` **above** the `[[component]]` table to define
two variables whose values will be used later in code:

```toml
# The [variables] table must occur above the [[component]] table
[variables]
turso_url = { default = "libsql://your-turso-database.turso.io" }
turso_auth_token = { required = true, secret = true }
```

:::note

Above, `turso_auth_token` is defined as a secret variable that will eventually
contain a database auth token that you will create later. You should not share
this token with anyone you don't trust with full access to your database.

:::

Add the following lines to `spin.toml` **below** the `[[component]]` table to add
the variables to the component config:

```toml
# The [component.config] table must occur below the [[component]] table
[component.config]
turso_url = "{{ turso_url }}"
turso_auth_token = "{{ turso_auth_token }}"
```

Modify the value for `allowed_http_hosts` within the `[[component]]` table:

```toml
# The following line must be modified under the [[component]] table
allowed_http_hosts = ["your-turso-database.turso.io"]
```

:::note

By default, Spin does not allow any outgoing network connections. In order to
query your Turso database, `allowed_http_hosts` must contain the hostname of
your database in order for Spin to [allow network access] to it.

:::

In the above configurations, "your-turso-database.turso.io" must be replaced by
your Turso database hostname.

Run the following Turso CLI command to get the libSQL URL for your database:

```bash
$ turso db show turso-fermyon --url
```

Copy the hostname part of the URL and paste it to the two places where you see
"your-turso-database.turso.io" in `spin.toml` (both the `turso_url` variable and
`allowed_http_hosts`).

:::note

- The `turso_url` variable must have the full URL of the database, including
  "libsql://".
- The `allowed_http_hosts` array just requires the hostname part of the URL.

:::

The entire spin.toml file will look something like this, with your database
hostname substituted in the highlighted lines:

```toml title=spin.toml
spin_manifest_version = "1"
authors = ["your-name <your-email>"]
description = ""
name = "turso-fermyon"
trigger = { type = "http", base = "/" }
version = "0.1.0"

[variables]
# highlight-next-line
turso_url = { default = "libsql://turso-fermyon-[your-github].turso.io" }
turso_auth_token = { required = true, secret = true }

[[component]]
id = "turso-fermyon"
source = "target/wasm32-wasi/release/turso_fermyon.wasm"
# highlight-next-line
allowed_http_hosts = ["turso-fermyon-[your-github].turso.io"]
[component.trigger]
route = "/..."
[component.build]
command = "cargo build --target wasm32-wasi --release"
watch = ["src/**/*.rs", "Cargo.toml"]
[component.config]
turso_url = "{{ turso_url }}"
turso_auth_token = "{{ turso_auth_token }}"
```

### 3b. Add code to create a client object and query Turso in `src/lib.rs`

Open `src/lib.rs`, delete all of the template code, and replace it with the following:

```rust
use anyhow::Result;
// Use the libsql_client module
use libsql_client::{Config, SyncClient};
use spin_sdk::{
    http::{Request, Response},
    http_component, config,
};

/// A simple Spin HTTP component.
#[http_component]
fn handle_turso_fermyon(_req: Request) -> Result<Response> {
    // Get component configuration for the Turso database
    let turso_url = config::get("turso_url")?;
    let turso_auth_token = config::get("turso_auth_token")?;

    // Create a SyncClient object for querying Turso
    let libsql_config = Config::new(turso_url.as_str())?
        .with_auth_token(turso_auth_token);
    let libsql_client = SyncClient::from_config(libsql_config)?;

    // Make a query that returns a ResultSet with one row with one column containing a string
    let rs = libsql_client.execute("select 'Hello, Turso'")?;
    let message = rs.rows[0].values[0].to_string();

    Ok(http::Response::builder()
        .status(200)
        .body(Some(message.into()))?)
}
```

The value of the `turso_auth_token` config variable will be specified when running
and deploying the component.

:::note

This code uses a version of the client `SyncClient` that offers a fully
synchronous API. The libSQL Rust SDK also offers a client with an asynchronous
API, but async functions are not yet supported by Spin.

:::

:::note

In order to keep this code sample small, it does no error checking at all. You
may want to perform your own error checking wherever you see the `?` operator.

:::

## 4. Run the component locally with `spin up`

Before running, you will need an auth token string for the `turso_auth_token`
variable in the configuration and code you added in the prior step. Run the
following Turso CLI command to get a database auth token:

```bash
$ turso db tokens create turso-fermyon
```

:::caution

Do not share this token with anyone that you do not trust with full access to
your database.

:::

Copy the output and paste it into the following command to export a new shell
environment variable:

```bash
$ export SPIN_CONFIG_TURSO_AUTH_TOKEN="paste-your-token-here"
```

:::note

Spin environment variables always start with `SPIN_CONFIG_`, and the remainder
of the env var name must match the variable defined in `spin.toml`. In this
case, the shell environment variable `SPIN_CONFIG_TURSO_AUTH_TOKEN` contains the
value for the `turso_auth_token` secret variable.

:::

Build and run the component locally using the Spin CLI:

```bash
$ spin build && spin up
```

The Spin CLI outputs the local URL you can use to invoke the component. Copy the
URL and paste it on the curl command line:

```bash
$ curl http://127.0.0.1:3000
```
```
Hello, Turso
```

It outputs the message "Hello, Turso" generated by the select SQL statement in
the code.

Stop the local Spin server with ctrl-C.

## 5. Deploy and run the component on Fermyon Cloud

Deploy the updated component using the Spin CLI, specifying the value of the
`turso_auth_token` value using the shell environment variable defined in the
prior step:

```bash
$ spin deploy --variable turso_auth_token="$SPIN_CONFIG_TURSO_AUTH_TOKEN"
```

The CLI outputs the same remote URL that you got previously in step 2. Copy the
URL and use curl to access it:

```
$ curl https://turso-fermyon-[RANDOM-CHARS].fermyon.app
```
```
Hello, Turso
```

Congratulations, you have deployed a Spin application to Fermyon Cloud that
queries Turso with the libSQL Rust SDK!

## 6. More resources

Now that you know how to deploy a Fermyon Spin application using Turso, use the
following resources to learn more about how Turso and the libSQL Rust SDK work.

- [Turso documentation]
- [Turso CLI reference]
- [libSQL Rust SDK reference]
- [libsql-client crate]


[Turso]: https://turso.tech
[Turso CLI]: /reference/turso-cli
[Fermyon Cloud account]: https://cloud.fermyon.com/login
[installation instructions]: /reference/turso-cli#installation
[using rustup]: https://www.rust-lang.org/tools/install
[follow the Spin quickstart]: https://developer.fermyon.com/spin/quickstart
[libsql-client crate]: https://crates.io/crates/libsql-client
[Spin Application Variables and Secrets]: https://developer.fermyon.com/cloud/variables
[allow network access]: https://developer.fermyon.com/spin/writing-apps#granting-networking-permissions-to-components
[Turso documentation]: https://docs.turso.tech
[Turso CLI reference]: https://docs.turso.tech/reference/turso-cli
[libSQL Rust SDK reference]: https://docs.turso.tech/reference/client-access/rust-sdk
