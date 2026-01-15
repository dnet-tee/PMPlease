## Building with Make + Docker

Uses a Makefile wrapper around a multi-stage Docker build. Each Make target corresponds to a Docker build stage and produces artifacts in a shared output directory.

### Available build targets

The following Make targets are supported:

- `debian-kernel`
- `linuxboot-kernel`
- `opensbi`
- `u-root`
- `zsbl`
- `pack`
- `sdcard-image`

For example to build only the zsbl image:

```bash
make zsbl
```

### Building all targets

```bash
make
```

Produced files are found under `sg2042-bootloader/out/`

### Note

The Makefile does not yet compile the keystone framework. If you have made changes to the keystone source code, just copy the produced `fw_dynamic.bin` to the `sg2042-bootloader/` directory before executing `make`
