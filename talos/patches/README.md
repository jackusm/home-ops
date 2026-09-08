# Talos Patching

This directory contains patches that are added to the talstomize configuration file.

<https://www.talos.dev/v1.7/talos-guides/configuration/patching/>

## Patch Directories

Under this `patches` directory, there are several sub-directories that can contain patches that are added to the talstomize configuration file.
Each directory is optional and therefore might not created by default.

- `global/`: patches that are applied to both the controller and worker configurations
- `controller/`: patches that are applied to the controller configurations
- `worker/`: patches that are applied to the worker configurations

The `talstomize.yaml` file maps these to the `patches`, `controlplanePatches`, and `workerPatches` keys; per-node overrides go under `nodes.<name>.patches`.
