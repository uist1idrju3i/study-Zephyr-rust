# Zephyr の Rust 言語サポート

対象: 公式ドキュメント + `zephyrproject-rtos/zephyr-lang-rust` (解析時点の実コード確認)。
記録日: 2026-11-22

## 概要

- Rustサポートは Zephyr 本体のマニフェストに含まれない外部モジュール
  [`zephyr-lang-rust`](https://github.com/zephyrproject-rtos/zephyr-lang-rust) として提供される。
- 開始は2024-09、主導はLinaro (d3zd3z)。ライセンスは Apache-2.0。
- `CONFIG_RUST` が `select EXPERIMENTAL` するため、公式位置づけは experimental。

## 有効化手順 (公式)

1. west マニフェストにモジュールを追加する
   - 上流Zephyr (4.5以前): `west config manifest.project-filter +zephyr-lang-rust` または
     `zephyr/submanifests/*.yaml` にプロジェクト定義を追加し `west update`。
     配置先は `modules/lang/rust`。
   - 上流Zephyr main (次期4.6): 2026-11-05 のコミット `289d75c`
     `manifest: move zephyr-lang-rust to be an external module` で
     optional.yaml から削除され外部モジュール化。アプリ側 `west.yml` の
     `manifest.projects` への明示追加が必要 (`manifest.project-filter` は無効)。
     本リポジトリの T2型 `west.yml` はこの新運用と同じ形式。
   - NCS: `name-allowlist` に含まれないため上流の方法はそのまま使えない
     (→ `community.md`, `design-decisions.md` 参照)。
2. `prj.conf` に `CONFIG_RUST=y`。
3. `CMakeLists.txt`: `find_package(Zephyr REQUIRED ...)` の後に `rust_cargo_application()`。
4. `Cargo.toml`: パッケージ名は現状 `rustapp` 固定、`[lib] crate-type = ["staticlib"]`、
   依存に `zephyr = "0.1.0"`。
5. `src/lib.rs`: `#![no_std]` + `#[no_mangle] extern "C" fn rust_main()`。
   C側 `main()` から `rust_main` が呼ばれる (モジュール内 `main.c`)。

## Kconfig

- `RUST_SUPPORTED`: `CPU_CORTEX_M` / `RISCV` (RV32E, RV128I 除く) / `ARCH_POSIX && 64BIT`
  かつ `!TIMER_READS_ITS_FREQUENCY_AT_RUNTIME` のとき y。
- `CONFIG_RUST_ALLOC=y` で Zephyr の malloc/free を使う `alloc` クレートが有効化。

## Rustターゲットの選定 (モジュール内 CMakeLists.txt)

CPU種とFPU構成から自動決定される。主な対応は次のとおり。

| Zephyr構成 | Rustターゲット |
|---|---|
| Cortex-M0/M0+ | `thumbv6m-none-eabi` |
| Cortex-M3 | `thumbv7m-none-eabi` |
| Cortex-M4/M7 FPUなし | `thumbv7em-none-eabi` |
| Cortex-M4/M7 FPUあり | `thumbv7em-none-eabihf` |
| Cortex-M23 (Armv8-M Baseline) | `thumbv8m.base-none-eabi` |
| Cortex-M33 FPUなし / soft-float ABI | `thumbv8m.main-none-eabi` |
| Cortex-M33 FPU + hard-float ABI | `thumbv8m.main-none-eabihf` |
| Cortex-M55 | 未対応 (Rustに `thumbv8.1m.main` が無い) |
| RV32 / RV64 (IMAC) | `riscv32i-unknown-none-elf` / `riscv64imac-unknown-none-elf` |
| posix64 | `x86_64-unknown-none`, `aarch64-unknown-none` |

ツールチェーンは Zephyr SDK に含まれず、rustup で別途導入する。
バージョンはピンされていない (rust-toolchain.toml 無し、stable想定)。

## バインディング生成 (zephyr-sys)

- `bindgen` 0.70.1 (experimental feature) でビルド時にZephyrヘッダから生成。
  ホストに libclang が必須。
- `wrapper.h` がinclude起点。allowlistパターンは `k_.*`, `gpio_.*`, `flash_.*`,
  `zr_.*`, `GPIO_.*`, `FLASH_.*` 等 (revisionにより変動)。
- 生成物は `zephyr::raw` 経由で参照可能。Kconfig値は `zephyr::kconfig::CONFIG_*`
  定数として取得できる。
- `west build -t rustdoc` でボード/設定固有の Rust API ドキュメントを生成可能。
- `build.rs` で `zephyr-build` クレートを呼ぶと `#[cfg]` 向けの条件コンパイルが使える
  (revision dd73abc: `export_bool_kconfig()` / `dt_cfgs()`、
  main: `export_kconfig_bool_options()`。API名が変わっている点に注意)。
- `log` クレートが標準の出力経路。`defmt` は上流 Issue #60 が未完了。

## `zephyr` クレートの提供機能 (revision dd73abc 実コード)

| モジュール | 内容 |
|---|---|
| `logging` | `set_logger()` + `log` クレート統合 (printk / zlog2 バックエンド) |
| `printk` | `printk` 直接呼び出し |
| `kconfig` | Kconfig値定数参照 |
| `devicetree` | devicetree由来の型安全デバイスアクセス (trait + `dt-rust.yaml` augment) |
| `device::{gpio, flash}` | GPIO / flash の安全ラッパ |
| `sync` | Mutex, SpinMutex, channel (Zephyrオブジェクトベース) |
| `thread` | スレッド生成・属性 |
| `time`, `timer` | uptime, sleep, 周期タイマー |
| `work` | ワークキュー |
| `embassy` | Embassy executor 統合 (サンプル `embassy` / `async-philosophers`) |
| `simpletls` | 簡易TLS |
| `object`, `error`, `align`, `alloc_impl` | 内部基盤 |

## ピン dd73abc と main の差分 (2026-11-22 時点)

本リポジトリは NCS v3.4.2 がピンする `dd73abc` を採用しているが、main は
約99コミット先行している。主な差分は次のとおり。

- `zephyr-panic` クレートへのパニックハンドラ分離 (#169)
- `RUST_BLOCKING_POOL` Kconfig + `zephyr::blocking::run` (ワーカースレッドで
  同期 Zephyr API を await 可能に)
- I2C コントローラ/ターゲットのバインディング (#150)、Hexagon 対応 (#159)
- Mutex の再入禁止・`MutexGuard: !Send` など安全性修正
- 進行中PR: UART (blocking/async), RTC, PWM, センサ, watchdog,
  edtlib 経由の devicetree バインディング生成 など
- 実機側の未解決Issue: ESP32-C3 #134, ESP32-C6 #141, RISCV32 SMP #62,
  alloc の atomic 制約 #131, SDK libclang 自動検出 #144, 実機検証文書化 #158

## 制約・注意点 (コード・ドキュメントから)

- unsafe境界は残る: `set_logger`, GPIO configure 等の一部APIは `unsafe fn`。
  「unsafe無しでZephyrを使い切る」は目標であって現状は未達。
- Cargo.toml の `name` は `rustapp` 固定 (`librustapp.a` をCMake側が期待)。
- `CONFIG_ENFORCE_ZEPHYR_STDINT` (picolibcデフォルト) 構成では
  `-imacros zephyr_stdint.h` がbindgenに伝わらず `__UINT32_C` 未定義エラーになる
  (→ `community.md`, `verification/build-2026-11-22.md`)。
- サンプル: `hello_world`, `blinky`, `button`, `philosophers`,
  `async-philosophers`, `blocking`, `bench`, `embassy` の8種。
- `sample.yaml` の `platform_allow` にNordic実機は `nrf52840dk/nrf52840` のみ
  (nRF54系は未記載だが `RUST_SUPPORTED` 上はビルド可能)。
