# nRF54L15-DK / nRF54L15 SoC

記録日: 2026-11-22

## ハードウェア

- nRF54L15 DK は PCA10156。
- nRF54L15 SoC の構成は次のとおり。
  - アプリケーションコアは Arm Cortex-M33 @ 128 MHz
    (`ARMV8_M_DSP`, `CPU_HAS_ARM_MPU`, `CPU_HAS_ARM_SAU`, `CPU_HAS_FPU`。
    sdk-zephyr `soc/nordic/nrf54l/Kconfig` で確認)
  - 補助コアは RISC-V VPR (FLPR)
  - 不揮発メモリ 1.5 MB / RAM 256 KB
    (ビルドのリンカ領域表示で FLASH 1524 KB, RAM 256 KB と確認)

## ボードターゲット (NCS 3.4.x)

| ターゲット | 用途 |
|---|---|
| `nrf54l15dk/nrf54l15/cpuapp` | アプリケーションコア単一実行環境 (今回の対象) |
| `nrf54l15dk/nrf54l15/cpuapp/ns` | TF-M (CMSEセキュア分離) |
| `nrf54l15dk/nrf54l15/cpuflpr` | VPR (FLPR) コア |
| `nrf54l15dk/nrf54l15/cpuflpr/xip` | FLPRのXIP構成 |

## Rustとの相性

- `cpuapp` は `CPU_CORTEX_M` → `RUST_SUPPORTED` OK。
- `cpuflpr` は RV32E系 → `RUST_SUPPORTED` 対象外。Rust対象は cpuapp のみ。
- FPU自体は搭載されているが、ZephyrはデフォルトでソフトフロートABIを選択
  (`CONFIG_FP_HARDABI` がデフォルト無効) ため、Rustターゲットは
  **`thumbv8m.main-none-eabi`**。`CONFIG_FP_HARDABI`/`FPU_SHARING` を有効化すれば
  `thumbv8m.main-none-eabihf` に切り替わる想定。
- ログ出力: オンボード J-Link OB の USB CDC ACM (VCP) 経由UART。115200bps。
- 書き込み: `west flash` (J-Link) または `nrfutil device`。
- ユーザー評価可能な可視確認: `led0` (blinky サンプルの `dt_cfgs` +
  `zephyr::devicetree::aliases::led0` で扱える)。
