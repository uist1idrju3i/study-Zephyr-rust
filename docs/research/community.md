# コミュニティ情報 (Zephyr Rust × NCS × nRF54L15)

記録日: 2026-11-22

## zephyrproject-rtos/zephyr-lang-rust issue #126 (2025-08)

- ユーザーがnRF54L15-DK + NCSでRustを試行した。
  - 上流手順 (`manifest.project-filter +zephyr-lang-rust`) が NCSでは機能しない
    (`name-allowlist` で輸入されるプロジェクトが制限されるため)。
  - 手動で `modules/lang/rust` にclone + `CONFIG_RUST=y` →
    `RUST has no prompt` エラー (Kconfig.rust のフォールバック定義側が解決される問題)。
  - `west.yml` の `name-allowlist` に `zephyr-lang-rust` を追加して解決したと報告。
- 示唆: NCSではマニフェスト側の明示的な宣言が必須。issueはクローズ済み。
- 補足: 現在は `sdk-zephyr/submanifests/optional.yaml` に同モジュールがピン
  入りしているが、`west list` で確認する限り NCSワークスペースには入ってこない。

## コミュニティ実装例

- [`ciniml/zephyr_rust_examples`](https://github.com/ciniml/zephyr_rust_examples) は
  NCS上でZephyr Rustを動かす実例集。nRF54L15-DKでの動作実績があるとREADME記載
  (本リポジトリではその手順は試していない)。
- zephyr-lang-rust自体のCI (`.github/workflows/build.yml`): Ubuntu 22.04上で
  twister + clippy を回し、対象ターゲットとして
  `thumbv6m/v7m/v7em/v8m.main-none-eabi`, `riscv32i/64imac`, `x86_64`, `aarch64`
  を `rustup target add` している。stable rustup を前提。

## 周辺ツール・評価 (Gemini Deep Research 由来、照合済み)

- Derek Molloy 氏「The State of Rust for Embedded Development in Mid-2026」
  (2026-07-15): zephyr-lang-rust はアクセス可能なAPI範囲が小さく製品出荷段階
  ではないと評価。無線 (BLE/Thread/Matter) では標準Zephyrまたは Rust+C
  スタック併用、非無線では Embassy/RTIC が現実的な選択肢と整理。
- Ac6「Workbench for Zephyr」(VS Code拡張): Rust ツールチェイン導入と
  zephyr-lang-rust の有効化、STM32F4でのRustサンプル構築までGUIで支援。
- OpenDevicePartnership (ODP): 独自フォークで defmt 対応を試行するなど
  先行開発あり。上流への maintainer 追加の動きも報告されている (伝聞)。
- `probe-rs` は Zephyr 4.0 以降の公式 west ランナー (書き込み・デバッグ
  の代替ツール。アプリのRust対応とは別件)。
- DigiKeyフォーラム投稿: nRF54LM20 DK で `west.yml` 手動編集による最小
  ビルド実績 (Flash 54,700 B / RAM 11,184 B と報告。伝聞)。
- 国内動向: Zephyr Meetup Tokyo 2025-12-08 でKurokawa氏による実機デモ、
  技術書典18でのNCS+Rust手順書などの報告あり (伝聞)。

## 古い情報・よくある誤解

- `tylerwhall/zephyr-rust` (2020年頃頻出) は公式以前の非公式プロジェクトで
  `std` 対応を目指したもの。公式 `zephyr-lang-rust` は `no_std`
  (+`CONFIG_RUST_ALLOC` で `alloc`) のみ。
- `west config manifest.project-filter +zephyr-lang-rust` の手順は
  Zephyr main (次期4.6) の外部モジュール化で無効。`west.yml` の
  `manifest.projects` に明示するのが現行手順 (本リポジトリの方式と一致)。
- 「某マイコンがZephyrとRustに対応」という記事は、C版Zephyr対応と
  Embassy等のベアメタルRust対応を並記したものが多く、zephyr-lang-rustでの
  動作実績を意味しない。
- defmt は上流 Issue #60 が未完了。標準は `log` クレート + printk /
  Zephyr logging 経由。

## 共通認識 (公式・コミュニティ双方)

- バインディングは「まだ最小限」であり、unsafe 領域が残ることは公式も認めている。
- Cargo.toml の `name = "rustapp"` 固定・一部API名の変更
  (例: `export_bool_kconfig` → `export_kconfig_bool_options`) 等、
  インタフェースは流動的。revision固定が重要。

## 見つけた落とし穴 (本リポジトリ実検証由来)

- `CONFIG_ENFORCE_ZEPHYR_STDINT` 構成で bindgen が `__UINT32_C` 未定義となる問題。
  コミュニティ記事・issueでは未整備の新規知見と見られる
  (→ `../verification/build-2026-11-22.md` 参照)。
