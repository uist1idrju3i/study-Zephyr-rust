# コミュニティ情報 (Zephyr Rust × NCS × nRF54L15)

記録日: 2026-11-22

## zephyrproject-rtos/zephyr-lang-rust issue #126 (2025-08)

- ユーザーがnRF54L15-DK + NCSでRustを試行:
  - 上流手順 (`manifest.project-filter +zephyr-lang-rust`) が NCSでは機能しない
    (`name-allowlist` で輸入されるプロジェクトが制限されるため)。
  - 手動で `modules/lang/rust` にclone + `CONFIG_RUST=y` →
    `RUST has no prompt` エラー (Kconfig.rust のフォールバック定義側が解決される問題)。
  - `west.yml` の `name-allowlist` に `zephyr-lang-rust` を追加して解決したと報告。
- 示唆: NCSではマニフェスト側の明示的な宣言が必須。issueはクローズ済み。
- 補足: 現在は `sdk-zephyr/submanifests/optional.yaml` に同モジュールがピン
  入りしているが、`west list` で確認する限り NCSワークスペースには入ってこない。

## コミュニティ実装例

- [`ciniml/zephyr_rust_examples`](https://github.com/ciniml/zephyr_rust_examples):
  NCS上でZephyr Rustを動かす実例集。nRF54L15-DKでの動作実績があるとREADME記載
  (本リポジトリではその手順は試していない)。
- zephyr-lang-rust自体のCI (`.github/workflows/build.yml`): Ubuntu 22.04上で
  twister + clippy を回し、対象ターゲットとして
  `thumbv6m/v7m/v7em/v8m.main-none-eabi`, `riscv32i/64imac`, `x86_64`, `aarch64`
  を `rustup target add` している。stable rustup を前提。

## 共通認識 (公式・コミュニティ双方)

- バインディングは「まだ最小限」であり、unsafe 領域が残ることは公式も認めている。
- Cargo.toml の `name = "rustapp"` 固定・一部API名の変更
  (例: `export_bool_kconfig` → `export_kconfig_bool_options`) 等、
  インタフェースは流動的。revision固定が重要。

## 見つけた落とし穴 (本リポジトリ実検証由来)

- `CONFIG_ENFORCE_ZEPHYR_STDINT` 構成での bindgen `__UINT32_C` 未定義:
  コミュニティ記事・issueでは未整備の新規知見と見られる
  (→ `../verification/build-2026-11-22.md` 参照)。
