# 検討結果: 設計・構成の決定と代替案

記録日: 2026-11-22

## 決定一覧

| 項目 | 採用 | 却下した代替案と理由 |
|---|---|---|
| NCS版 | **v3.4.2 (LTS)** | v3.3.x: LTSでない。main: 破壊的変更の可能性 |
| Zephyr Rust モジュール | **zephyr-lang-rust @ dd73abc** | `main` 最新: API名が流動的 (`export_kconfig_bool_options` 等)。dd73abcはNCS側 submanifests/optional.yaml が同タグでピンする「整合した」版 |
| マニフェスト方式 | **T2型 `west.yml` (リポジトリ自身)** | (a) NCSワークスペースの `name-allowlist` 編集: SDK配布物を改変、再現性なし。(b) submanifest追加も sdk-zephyr 配下は import の name-allowlist で遮断。(c) `ZEPHYR_EXTRA_MODULES`: 動くがマニフェスト管理外。→リポジトリ自身を manifest repository にし、`sdk-nrf` を `import: true` + zephyr-lang-rust をプロジェクト追加する方式が最も再現性が高い |
| アプリ構成 | **hello_world 相当 (ログ出力)** | blinky: LED可視は魅力的だが初期の動作確認としてはコンソールログが最小かつ twister 準拠の成否判定になる。次ステップ候補として記録 |
| Rustツール | rustup stable (1.100.0) | 特定バージョンへのピン留めは不要 (モジュールにtoolchainピン無し) |
| bindgen 環境 | ホストlibclang + `BINDGEN_EXTRA_CLANG_ARGS=-imacros .../zephyr_stdint.h` | zephyr-sys パッチ: 外部モジュールの改変で保守負担。環境変数注入が最小侵襲 |

## 残タスク・次候補

- 実機検証: `west flash` + UART `Hello world from Rust on nrf54l15dk` 確認
- LED blinky化、ボタン、スレッド/ワークキュー、Embassy統合
- `CONFIG_FP_HARDABI` で `thumbv8m.main-none-eabihf` 切替確認
- `west build -t rustdoc` でボード固有APIドキュメント生成
- `cpuapp/ns` (TF-M) 経路でのRust可否調査
