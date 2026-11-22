# study-Zephyr-rust

nRF54L15-DK (`nrf54l15dk/nrf54l15/cpuapp`) で、Zephyr の Rust 言語サポートを試すリポジトリです。基盤は nRF Connect SDK v3.4.2 です (NCS 初の LTS 系列、Zephyr 4.4.2)。

Zephyr の Rust サポートは外部モジュール [zephyr-lang-rust](https://github.com/zephyrproject-rtos/zephyr-lang-rust) として提供され、状態は experimental です。NCS のマニフェストには既定で含まれないため、このリポジトリは ncs-v3.4.2 の `sdk-zephyr/submanifests/optional.yaml` がピンする版 (`dd73abc`) を指定しています。

## 構成

- `west.yml`: T2 型の west マニフェストです。`sdk-nrf` v3.4.2 を import し、`zephyr-lang-rust` を `modules/lang/rust` に追加します。
- `app/`: `no_std` の最小 Rust アプリケーションです (hello_world 相当)。Zephyr の logging 経由で `Hello world from Rust on nrf54l15dk` を UART コンソールに出します。
- `docs/`: 調査・検討・検証の記録です。索引は [docs/README.md](docs/README.md) にあります。

## 必要なもの

- nRF Connect SDK v3.4.2 ツールチェーン
  (`nrfutil sdk-manager toolchain install --ncs-version v3.4.2`)
- [rustup](https://rustup.rs/) で入れた Rust と、次の ARM 用ターゲット。

  ```sh
  rustup target add thumbv8m.main-none-eabi
  ```

  Rust ターゲットは CPU と FPU の構成から自動で決まります。このボードでは `thumbv8m.main-none-eabi` です (`CONFIG_FP_HARDABI` が既定で無効なため soft-float ABI)。
- bindgen が使う libclang (例: `sudo apt install libclang-dev clang`)

## ビルド

### 方法 A: このリポジトリのマニフェストで専用ワークスペースを作る

```sh
west init -l <このリポジトリのパス>
cd <ワークスペース>
west update   # sdk-nrf・zephyr・zephyr-lang-rust (modules/lang/rust) を取得

west build -b nrf54l15dk/nrf54l15/cpuapp -s study-Zephyr-rust/app
```

`west update` は NCS 全体を clone します (数 GB)。

### 方法 B: 既存の NCS v3.4.2 ワークスペースを使う

```sh
# ピンされた版で Rust モジュールを取得
git clone https://github.com/zephyrproject-rtos/zephyr-lang-rust
cd zephyr-lang-rust && git checkout dd73abc242e995784da62352fe8c70d9a6c7ac2e && cd ..

nrfutil sdk-manager toolchain launch --ncs-version v3.4.2 -- bash -c \
  'ZEPHYR_EXTRA_MODULES=<path>/zephyr-lang-rust \
   west build -b nrf54l15dk/nrf54l15/cpuapp -s <このリポジトリ>/app'
```

### bindgen の注意点 (picolibc 構成では必須)

既定の `CONFIG_ENFORCE_ZEPHYR_STDINT=y` では、`__UINT32_C` などの整数定数マクロがコンパイラの `-imacros` フラグ経由でだけ有効になり、bindgen には届きません。そのままだと `zephyr/kernel/internal/mm.h` でバインディング生成が失敗します。bindgen の clang に同じヘッダを指すことで回避できます。

```sh
export BINDGEN_EXTRA_CLANG_ARGS="-imacros ${ZEPHYR_BASE}/include/zephyr/toolchain/zephyr_stdint.h"
```

## 期待する結果

`west build` で `build/app/zephyr/zephyr.hex` ができます。書き込み (`west flash` または `nrfutil device`) のあと、J-Link OB の USB シリアル (115200bps) を開くと次の出力が出ます。

```
Hello world from Rust on nrf54l15dk
```

本リポジトリでは `west build` までを検証しました。実機への書き込みと UART 出力は試していません。
