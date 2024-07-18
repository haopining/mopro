# mopro

Mopro is a toolkit for ZK app development on mobile. Mopro makes client-side proving on mobile simple.

The example below shows how to bind **Circom** circuits to **iOS** and **Android**. For other adapters as well as deployment targets, please refer to the documentation at [zkmopro](https://zkmopro.org/docs/intro).

## Getting started

Make sure you've installed the [prerequisites](https://zkmopro.org/docs/prerequisites).

After that, you can clone the repo and run `cargo build` or `cargo test`.

## Using a pre-built app library

There are pre-built examples projects for iOS and Android [here](https://github.com/vimwitch/mopro-app).

## Building your own app library

Mopro works by providing a static library and an interface for your app to build proofs. Before you start this tutorial you should have a `zkey` and `wasm` file generated by circom.

To get started we'll make a new rust project that builds this library. Run the following commands in your terminal:

```sh
mkdir mopro-example
cd mopro-example
cargo init --lib
```

This will create a new rust project in the current directory. Now we'll add some dependencies to this project. Edit your `Cargo.toml` so that it looks like the following:

```toml
[package]
name = "mopro-example"
version = "0.1.0"
edition = "2021"

# We're going to build a static library named mopro_bindings
# This library name should not be changed
[lib]
crate-type = ["lib", "cdylib", "staticlib"]
name = "mopro_bindings"

# This is a script used to build the iOS static library
[[bin]]
name = "ios"

# In this example we're going to build support only for circom proofs
[features]
default = ["mopro-ffi/circom"]

[dependencies]
mopro-ffi = { git = "https://github.com/zkmopro/mopro.git" }
uniffi = { version = "0.28", features = ["cli"] }
rust-witness = { git = "https://github.com/vimwitch/rust-witness.git" }
num-bigint = "0.4.0"

[build-dependencies]
rust-witness = { git = "https://github.com/vimwitch/rust-witness.git" }
mopro-ffi = { git = "https://github.com/zkmopro/mopro.git" }
uniffi = { version = "0.28", features = ["build"] }
```

Now you should copy your wasm and zkey files somewhere in the project folder. For this tutorial we'll assume you placed them in `test-vectors/circom`.

Now we need to add 4 rust files. First we'll add `build.rs` in the main project folder. This file should contain the following:

```rust
fn main() {
    // We're going to transpile the wasm witness generators to C
    // Change this to where you put your zkeys and wasm files
    rust_witness::transpile::transpile_wasm("./test-vectors/circom".to_string());
    // This is writing the UDL file which defines the functions exposed 
    // to your app. We have pre-generated this file for you
    // This file must be written to ./src
    std::fs::write("./src/mopro.udl", mopro_ffi::app_config::UDL).expect("Failed to write UDL");
    // Finally initialize uniffi and build the scaffolding into the
    // rust binary
    uniffi::generate_scaffolding("./src/mopro.udl").unwrap();
}
```

Second we'll change the file at `./src/lib.rs` to look like the following:

```rust
// Here we're generating the C witness generator functions
// for a circuit named `multiplier2`.
// Your circuit name will be the name of the wasm file all lowercase
// with spaces, dashes and underscores removed
//
// e.g.
// multiplier2 -> multiplier2
// keccak_256_256_main -> keccak256256main
// aadhaar-verifier -> aadhaarverifier
rust_witness::witness!(multiplier2);

// Here we're calling a macro exported by uniffi. This macro will
// write some functions and bind them to the uniffi UDL file. These
// functions will invoke the `get_circom_wtns_fn` generated below.
mopro_ffi::app!();

// This macro is used to set up the circom circuits to be used in the example.
// You can pass multiple comma seperated `(zkey_filename, witness_function)` pairs to it.
// One way to create the witness function is to use the `rust_witness!` above.
// Please read in the `adapters/circom` doc section on how you can manually configure this.
mopro_ffi::set_circom_circuits! {
    ("multiplier2_final.zkey", multiplier2_witness),
}
```

Finally, we'll add a new file at `src/bin/ios.rs`:

```rust
fn main() {
    // A simple wrapper around a build command provided by mopro.
    // In the future this will likely be published in the mopro crate itself.
    mopro_ffi::app_config::ios::build();
}
```

and another at `src/bin/android.rs`:

```rust
fn main() {
    // A simple wrapper around a build command provided by mopro.
    // In the future this will likely be published in the mopro crate itself.
    mopro_ffi::app_config::android::build();
}
```

Now you're ready to build your static library! You should be able to run either `cargo run --bin ios` or `cargo run --bin android` to build the corresponding static library.

## Using the library

To read how to add the resulting static library to your iOS or Android project, please refer to the documentation at [zkmopro](https://zkmopro.org/docs/intro).