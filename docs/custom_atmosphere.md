# Building Horizon OC against a custom Atmosphère

`build.sh` builds two components from an Atmosphère source tree:

- `dist/atmosphere/kips/hoc.kip`: Atmosphère's loader plus the Horizon OC overrides from `Source/Atmosphere/stratosphere/loader`.
- `dist/atmosphere/exosphere.bin`: Atmosphère's secure monitor plus the overrides from `Source/Atmosphere-Patches`. fusee loads `sdmc:/atmosphere/exosphere.bin` instead of the exosphere inside `package3`, and hekate users point `secmon=` at it.

Because this `exosphere.bin` replaces the one shipped with Atmosphère, it must be built from the same Atmosphère the console runs. Otherwise any secure-monitor change in that Atmosphère is lost.

For example, the [postkevone/Atmosphere](https://github.com/postkevone/Atmosphere) fork (branch `emmc-emummc`) adds emuMMC type `3` (raw partition on the internal eMMC). An `exosphere.bin` built from upstream Atmosphère does not know type `3`: its `smcAmsGetEmummcConfig` reaches an unreachable case and the console crashes at boot.

By default `build.sh` clones upstream Atmosphère and checks out its latest tag. Two optional environment variables change that.

## Options

| Variable | Default | Meaning |
|---|---|---|
| `ATMOSPHERE_URL` | `https://github.com/Atmosphere-NX/Atmosphere.git` | Atmosphère repository to clone into `build/atmosphere` |
| `ATMOSPHERE_REF` | latest tag of that repository | Tag, commit or branch name to build. A branch name is resolved as `origin/<branch>`. |

When neither is set, `build.sh` behaves exactly like upstream.

```sh
ATMOSPHERE_URL=https://github.com/postkevone/Atmosphere.git \
ATMOSPHERE_REF=emmc-emummc \
bash build.sh
```

`build.sh` only clones when `build/atmosphere` doesn't exist yet. Delete `build/atmosphere` whenever you change `ATMOSPHERE_URL` or `ATMOSPHERE_REF`, or want to pick up new commits.

## Toolchain

- Atmosphère 1.12.1 and newer (including the fork above) need **libnx from git master**, because they use `FsStorageForBatchRead`, which libnx 4.12.0 doesn't have.
- The repository's CI pins Atmosphère 1.11.2 with libnx 4.12.0 and doesn't build exosphere, so it is not affected.

Docker recipe used for the fork build, based on Atmosphère's own `Dockerfile`:

```sh
docker build -t ams-build <path to Atmosphere checkout>

cat > Dockerfile.libnx <<'EOF'
FROM ams-build
USER root
RUN git clone https://github.com/switchbrew/libnx.git /tmp/libnx && \
    cd /tmp/libnx && git checkout b156b47 && \
    make -C nx -j"$(nproc)" && make -C nx install && \
    rm -rf /tmp/libnx
USER atmosphere
EOF
docker build -t ams-build-libnx -f Dockerfile.libnx .

docker run --rm --user $(id -u):$(id -g) -e HOME=/tmp \
  -e DEVKITPRO=/opt/devkitpro -e DEVKITA64=/opt/devkitpro/devkitA64 \
  -e ATMOSPHERE_URL=https://github.com/postkevone/Atmosphere.git \
  -e ATMOSPHERE_REF=emmc-emummc \
  -v "$PWD":/src -w /src ams-build-libnx bash build.sh
```

The result is `dist/dist.zip`.

## Compatibility of the overrides

`build.sh` replaces whole files in the Atmosphère tree, so check them whenever the Atmosphère source changes. For the `emmc-emummc` fork, based on Atmosphère 1.12.1, the overrides apply cleanly:

- `secmon_smc_handler.cpp`, `secmon_smc_register_access.cpp` and `secmon_memory_layout.hpp` are identical in Atmosphère 1.11.2, 1.12.1 and the fork. The fork's changes are in other files: `secmon_smc_info.cpp` and `secmon_emummc_context.hpp`.
- `ldr_process_creation.cpp` from `Source/Atmosphere/stratosphere/loader` is based on current Atmosphère and only adds the Horizon OC hooks.

If an overridden file differs between your Atmosphère and upstream, merge the change into the Horizon OC copy instead of overwriting it.

## Extensions and hekate

Don't build with `--ext` when you use your own hekate (for example [postkevone/hekate](https://github.com/postkevone/hekate), branch `emmc-emummc`). `--ext` builds Horizon OC's custom hekate and ships it as `bootloader/update.bin` and `payload.bin`. hekate updates itself from `bootloader/update.bin`, so your hekate would be replaced.

## SD card setup with the emuMMC-on-eMMC forks

Extract in this order:

1. the Atmosphère fork release;
2. the hekate fork release, if you boot through hekate (its `bootloader/sys/emummc.kipm` is built from the Atmosphère fork);
3. this `dist.zip`.

hekate boot entry (`kip1` and `secmon` must come after `pkg3`, so they override what hekate loads from `package3`):

```ini
pkg3=atmosphere/package3
kip1=atmosphere/kips/hoc.kip
secmon=atmosphere/exosphere.bin
```
