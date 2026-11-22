# nRF Connect SDK 最新LTS分析

記録日: 2026-11-22

## LTSの位置づけ

- NCS v3.4.x がNCS初のLTSライン (Nordic公式ブログ 2026-07-02)。
  5年間のパッチ保証: バグ修正・重要セキュリティ修正のみ、
  上流Zephyr由来の破壊的変更は入らない。
- Nordic は「`Active` / `Maintenance` デバイスで `Supported` 機能を使う
  新規プロジェクトのデフォルトは LTS」と明言している。
- 採用版: v3.4.2 (2026-11-04リリース。記録時点の最新パッチ)。

## v3.4.x 技術構成

| 項目 | 値 |
|---|---|
| Zephyr | 4.4.2 (sdk-zephyr `ncs-v3.4.2`) |
| ツールチェーン | Zephyr SDK v1.0.1 (GCC推奨、GCC 14.3.0) |
| Mbed TLS | 4.1.0 |
| TF-M | 2.3 |
| West | manifest version 0.13, `group-filter` で `+optional +testing` |

- nRF52系はv3.4.xで最後 (feature complete。mainでは今後削除予定)。
- Partition Manager は非推奨 → DTSパーティション方式へ移行
  (2026年末までにmainから削除予定)。単一イメージビルドでは直接影響は小さい。

## Rustモジュールの取り込み経路 (実機確認済み)

- NCS `west.yml` の `name-allowlist` に `zephyr-lang-rust` は含まれない。
- ただし `sdk-zephyr/submanifests/optional.yaml` に
  `zephyr-lang-rust @ dd73abc242e995784da62352fe8c70d9a6c7ac2e`
  (`modules/lang/rust`, group `optional`) が宣言されている。
- 実際に `nrfutil sdk-manager` で入れたワークスペースでは
  `west list zephyr-lang-rust` が unknown project だった。自動では入らない。
  配線は用意されているがNCS側importが遮断している状態と読み取れる。
- → 利用には「アプリ側マニフェストで明示追加」または
  `ZEPHYR_EXTRA_MODULES` が必要 (詳細は `../plan/design-decisions.md`)。

## 本件との関連

- LTSである点はRust機能には直接関係しないが、Nordicが推奨する新規開発の
  基盤であるため採用した。
- LTSライン上のZephyr 4.4.2と、同タグでピンされている zephyr-lang-rust
  `dd73abc` の組み合わせが「Nordicが配線した」最新の整合性ある組み合わせ。
