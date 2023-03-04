<div align="center">

# `🐉bevy-rust-gpu`

[![Documentation](https://img.shields.io/badge/docs-API-blue)](https://bevy-rust-gpu.github.io/bevy-rust-gpu/bevy_rust_gpu/)

A bevy plugin supporting the use of [`rust-gpu`](https://github.com/EmbarkStudios/rust-gpu) shader crates.
Part of the [Bevy Rust-GPU](https://github.com/Bevy-Rust-GPU) suite.

Features include hot-reloading with metadata-based entrypoint validation, and hot-recompiling via runtime export of active entrypoints.

![Hot-rebuild workflow](https://github.com/Bevy-Rust-GPU/bevy-rust-gpu/blob/static-assets/hot-rebuild-workflow.gif?raw=true)

</div>

## Current Status 🚧

`bevy-rust-gpu` relies on [`rust-gpu`](https://github.com/EmbarkStudios/rust-gpu), which is in active development.

As such, its use implies all the caveats of the above, plus the following:

* [Using SPIR-V in a bevy `Material` requires a custom `bevy` fork](https://github.com/Bevy-Rust-GPU/bevy-rust-gpu/issues/12)
* [Storage buffers are currently unsupported](https://github.com/Bevy-Rust-GPU/bevy-rust-gpu/issues/13)

Beyond that, `bevy-rust-gpu` is also in active development, but has a relatively small user-facing API footprint.
Major changes will be driven by development in the upstream `bevy` and `rust-gpu` crates.

In practical terms, it's able to support the hot-rebuild workflow depicted above,
and allows for relatively complex shader implementations (storage buffer issues notwithstanding), such as [a Rust reimplementation of `bevy_pbr`](https://github.com/Bevy-Rust-GPU/bevy-pbr-rust).

Currently, none of the [Bevy Rust-GPU](https://github.com/Bevy-Rust-GPU) crates are published on crates.io;
this may change as and when the major caveats are solved, but in the meantime will be hosted on github and versioned by tag.

## Usage

First, add `bevy-rust-gpu` to your `Cargo.toml`:

```toml
[dependencies]
bevy-rust-gpu = { git = "https://github.com/Bevy-Rust-GPU/bevy-rust-gpu", tag = "v0.3.0" }
```

Next, implement a `Material` type to describe your material's bind group layout and pipeline specialization:

```rust
#[derive(Debug, Default, Copy, Clone, AsBindGroup, TypeUuid)]
#[uuid = "786779ff-e3ac-4b36-ae96-f4844f8e3064"]
struct MyRustGpuMaterial {
    #[uniform(0)]
    color: Vec4,
}

// The vertex and fragment shaders specified here can be used as a fallback
// when entrypoints are unavailable (see the documentation of bevy_rust_gpu::prelude::RustGpuSettings),
// but are otherwise deferred to ShaderRef::Default, so can be left unimplemented.
impl Material for MyRustGpuMaterial {}
```

Then, implement `RustGpuMaterial` for your `Material` type.

```rust
// First, implement some marker structs to represent our shader entry points

pub enum MyVertex {}

impl EntryPoint for MyVertex {
    const NAME: EntryPointName = "vertex";
    const PARAMETERS: EntryPointParameters = &[];
}

pub enum MyFragment {}

impl EntryPoint for MyFragment {
    const NAME: EntryPointName = "fragment";
    const PARAMETERS: EntryPointParameters = &[];
}

// Then, impl RustGpuMaterial for our material to tie them together

impl RustGpuMaterial for MyRustGpuMaterial {
    type Vertex = MyVertex;
    type Fragment = MyFragment;
}
```

(See [`bevy_pbr_rust.rs`](https://github.com/Bevy-Rust-GPU/bevy-rust-gpu/blob/master/src/bevy_pbr_rust.rs) for the [`bevy-pbr-rust`](https://github.com/Bevy-Rust-GPU/bevy-pbr-rust)-backed `StandardMaterial` reference implementation.)

Next, add `RustGpuPlugin` to your bevy app to configure the backend.

```rust
    let mut app = App::default();
    app.add_plugin(RustGpuPlugin); // Must be before RenderPlugin, i.e. before DefaultPlugins
    app.add_plugins(DefaultPlugins);
```

For each `RustGpuMaterial` implementor, add a `RustGpuMaterialPlugin::<M>` to your app to setup rendering machinery and hot-reload / hot-rebuild support if the respective features are enabled (see below.)

```rust
    app.add_plugin(RustGpuMaterialPlugin::<MyRustGpuMaterial>::default());

```

Finally, load your shader, and add it to a material:

```rust
fn setup(materials: ResMut<Assets<RustGpu<MyRustGpuMaterial>>>) {
    // Extension method provided by the LoadRustGpuShader trait
    // Returns a RustGpuShader, which is akin to Handle<Shader> with some extra hot-reloading machinery.
    let shader = asset_server.load_rust_gpu_shader(SHADER_PATH);

    // Add it to a RustGpu material, which can be used with bevy's MaterialMeshBundle
    let material = materials.add(RustGpu {
        vertex_shader = Some(shader),
        fragment_shader = Some(shader),
        ..default()
    });
    
    // Use material as per any other
    ...
}
```

## Feature Flags

### `hot-reload`

Enables hot-reloading support.

Automatically loads the `.spv.json` metadata generated by `rust-gpu` alongside its associated `.spv` file,
and uses it to validate entry points at material specialization time.

This prevents bevy from panicking when an invalid entrypoint is requested, falls back to the default shader,
and re-specializes the material if it becomes available after a reload.

Note: AssetServer gives up on trying to load an asset if it does not exist,
so the `.spv` file must be compiled at least once prior to app startup in order to hot-reload successfully.

### `hot-rebuild`

Adds the `EntryPointExport` resource, which can be used to retrieve an `ExportHandle` corresponding to a JSON output file.
When passed to a `RustGpu` material, active entrypoints to be aggregated and exported to the corresponding file on change.

This can be used in concert with the `hot-reload` feature, [`rust-gpu-builder`](https://github.com/Bevy-Rust-GPU/rust-gpu-builder)'s file watching functionality,
and [`permutate-macro`](https://github.com/Bevy-Rust-GPU/permutate-macro)'s static permutation generation to drive a hot-rebuild workflow:

* The bevy app loads a `RustGpu` material, tries to specialize it, and exports the set of required entry points to `entry_points.json`
* [`rust-gpu-builder`](https://github.com/Bevy-Rust-GPU/rust-gpu-builder) picks up the change to `entry_points.json` and triggers a recompile
* [`permutate-macro`](https://github.com/Bevy-Rust-GPU/permutate-macro) attributes in the target shader crates read `entry_points.json`, and conditionally generate the required entry points
* `rust-gpu` compiles the generated code, outputting `shader.spv` and `shader.spv.json`
* The bevy app picks up the changes to `shader.spv` and `shader.spv.json`, hot-reloads them, and respecializes the material with the now-available entry points
* Repeat as new `RustGpu` materials are loaded by the bevy app

### `bevy-pbr-rust`

Implements `RustGpu` for `StandardMaterial` via the `MeshVertex` and `PbrFragment` markers,
which corresponding to entry points defined in [`bevy-pbr-rust`](https://github.com/Bevy-Rust-GPU/bevy-pbr-rust).
