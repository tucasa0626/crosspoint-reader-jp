# CrossPoint Reader JP

**XTEINK X4向け日本語版CrossPoint Readerファームウェア**

[CrossPoint Reader](https://github.com/crosspoint-reader/crosspoint-reader)（MIT License, © 2025 Dave Allie）をベースとした非公式の日本語ディストリビューションです。CrossPoint Readerとは無関係の独立プロジェクトです。

---

## このプロジェクトについて

CrossPoint Readerは素晴らしいオープンソースのE-Inkリーダーファームウェアですが、日本語フォントと日本語UIが未対応でした。このフォークは、XTEINK X4を日本語環境で快適に使うことを目的として開発しています。

また、このデバイスを電子書籍リーダーとしてだけでなく、**アイディアやスケッチ、ZINEなどを持ち歩いて人に見せるポータブル表示デバイス**として育てることも目指しています。

---

## 実装済みの機能（Phase 1）

- **日本語UI**：設定・メニュー・ダイアログを含む全264キーを日本語化
- **日本語フォントレンダリング**：ひらがな・カタカナ・UI頻出漢字195字をビットマップ化して内蔵
  - Ubuntu JP 10pt / 12pt（UIフォント）：Ubuntu + Noto Sans CJK JPのフォントスタック
  - Noto Sans JP 8pt（ボタンラベルフォント）
- **言語選択画面**に「日本語」として表示・選択可能

---

## 今後の予定（Phase 2以降）

- **SDカードからの動的フォントロード**：フルセットの漢字対応、ファイル名の日本語表示、フォント選択機能
- **TEXT / Markdownビューア**：書きかけのアイディアをそのまま放り込んで閲覧
- **画像ビューア＋スライドショー**：スケッチやZINEをSDカードに入れてプレゼン
  - WiFiアップロード時に前処理（グレースケール化・リサイズ・ディザリング）を実施
- **PDF対応**：PC側でページ画像に変換してSDに転送する方式

---

## ビルド方法

### 必要なもの

- Docker（macOSのlibexpatの問題を回避するため）
- esptool（`brew install esptool`）
- XTEINK X4本体

### ビルド

```bash
git clone --recursive https://github.com/tucasa0626/crosspoint-reader-jp.git
cd crosspoint-reader-jp

docker run --rm \
  -v $(pwd):/project \
  -w /project \
  python:3.12-slim \
  bash -c "apt-get update -qq && apt-get install -y -qq git && pip install platformio && pio run"
```

### 書き込み

```bash
# XTEINK X4をUSB接続した状態で
esptool --chip esp32c3 --port /dev/cu.usbmodem101 --baud 921600 write-flash \
  0x10000 .pio/build/default/firmware.bin

esptool --chip esp32c3 --port /dev/cu.usbmodem101 --baud 921600 write-flash \
  0x650000 .pio/build/default/firmware.bin
```

> ポート名は環境により異なります（`ls /dev/cu.*`で確認）

### 元のファームウェアに戻す場合

書き込み前にバックアップを取っておくことを強く推奨します：

```bash
esptool --chip esp32c3 --port /dev/cu.usbmodem101 read_flash \
  0x0 0x1000000 official_firmware_backup.bin
```

バックアップから復元：

```bash
esptool --chip esp32c3 --port /dev/cu.usbmodem101 --baud 921600 write-flash \
  0x0 official_firmware_backup.bin
```

---

## フォントについて

日本語フォントには以下を使用しています：

- [Noto Sans CJK JP](https://github.com/notofonts/noto-cjk)（SIL Open Font License 1.1）
- [Ubuntu Font](https://design.ubuntu.com/font)（Ubuntu Font Licence 1.0）

フォント変換には`lib/EpdFont/scripts/fontconvert.py`を使用しています。

---

## ハードウェア仕様（XTEINK X4）

| 項目 | 仕様 |
|---|---|
| MCU | ESP32-C3（RISC-V、RAM 400KB、PSRAM無し） |
| ディスプレイ | 4.26" E-paper 480×800px 220PPI |
| フラッシュ | 16MB（app0/app1各6.25MB） |
| ストレージ | microSD |
| 接続 | WiFi / USB-C |

---

## ライセンス

MITライセンス（元のCrossPoint Readerに準拠）

Original CrossPoint Reader: Copyright © 2025 Dave Allie  
Japanese distribution modifications: Copyright © 2025 Tsukasa Ishizawa

本プロジェクトはCrossPoint Reader公式プロジェクトとは無関係です。
