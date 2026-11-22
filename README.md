# study-Zephyr-rust

Trying Zephyr's Rust language support on the **nRF54L15-DK**
(`nrf54l15dk/nrf54l15/cpuapp`) with **nRF Connect SDK v3.4.2**
(the first NCS LTS release line; Zephyr 4.4.2).

Rust support in Zephyr is provided by the external module
[`zephyr-lang-rust`](https://github.com/zephyrproject-rtos/zephyr-lang-rust)
(status: *experimental*). The NCS manifest does not include it by default;
this repo pins the same revision as `sdk-zephyr/submanifests/optional.yaml`
in ncs-v3.4.2 (`dd73abc`).

## Layout

- `docs/` — 調査・検討・検証記録 (Japanese). See
  [docs/README.md](docs/README.md)

- `west.yml` — standalone (T2) west manifest: imports `sdk-nrf` v3.4.2 and
  adds `zephyr-lang-rust` at `modules/lang/rust`
- `app/` — minimal `no_std` Rust application (`hello_world`-style):
  logs `Hello world from Rust on nrf54l15dk` via the Zephyr logging
  subsystem (UART console)

## Prerequisites

- nRF Connect SDK **v3.4.2** toolchain
  (`nrfutil sdk-manager toolchain install --ncs-version v3.4.2`)
- Rust via [rustup](https://rustup.rs/), plus the ARM target:

  ```sh
  rustup target add thumbv8m.main-none-eabi
  ```

  The build selects the Rust target automatically from the CPU and FPU
  configuration; for this board it is `thumbv8m.main-none-eabi`
  (soft-float ABI — `CONFIG_FP_HARDABI` is off by default).
- **libclang** for bindgen (e.g. `sudo apt install libclang-dev clang`)

## Build

### Option A — dedicated workspace from this repo's manifest

```sh
west init -l <path-to-this-repo>
cd <workspace>
west update   # fetches sdk-nrf, zephyr and zephyr-lang-rust → modules/lang/rust

west build -b nrf54l15dk/nrf54l15/cpuapp -s study-Zephyr-rust/app
```

`west update` clones the full NCS tree (several GB).

### Option B — reuse an existing NCS v3.4.2 workspace

```sh
# fetch the Rust module at the pinned revision
git clone https://github.com/zephyrproject-rtos/zephyr-lang-rust
cd zephyr-lang-rust && git checkout dd73abc242e995784da62352fe8c70d9a6c7ac2e && cd ..

nrfutil sdk-manager toolchain launch --ncs-version v3.4.2 -- bash -c \
  'ZEPHYR_EXTRA_MODULES=<path>/zephyr-lang-rust \
   west build -b nrf54l15dk/nrf54l15/cpuapp -s <this-repo>/app'
```

### bindgen note (required for picolibc builds)

With `CONFIG_ENFORCE_ZEPHYR_STDINT=y` (default for picolibc), integer
constant macros such as `__UINT32_C` reach C code through a compiler
`-imacros` flag that bindgen never sees, so binding generation fails in
`zephyr/kernel/internal/mm.h`. Work around it by pointing bindgen's clang
at the same header:

```sh
export BINDGEN_EXTRA_CLANG_ARGS="-imacros ${ZEPHYR_BASE}/include/zephyr/toolchain/zephyr_stdint.h"
```

## Expected result

`west build` produces `build/app/zephyr/zephyr.hex`. After flashing
(`west flash` or `nrfutil device`), the UART console (J-Link OB USB CDC,
115200 baud) prints:

```
Hello world from Rust on nrf54l15dk
```

Not verified on hardware in this repository; verified up to `west build`.
