# CrossPoint Reader JP 開発ノート

> 後からトレース・再現するための技術的経緯メモ。

---

## プロジェクト概要

- **目的**：XTEINK X4（ESP32-C3ベースE-Inkリーダー）向けに、CrossPoint ReaderのOSSファームウェアをforkして日本語化・機能拡張する
- **リポジトリ**：https://github.com/tucasa0626/crosspoint-reader-jp
- **upstream**：https://github.com/crosspoint-reader/crosspoint-reader（MIT License）
- **開発環境**：Mac Mini（Apple Silicon、macOS 26.2 Tahoe beta）＋ MacBook Air（開発端末）＋ VS Code Remote SSH

---

## ハードウェアメモ

### XTEINK X4の重要スペック

- MCU: ESP32-C3（QFN32）、RISC-Vシングルコア、RAM 400KB、**PSRAMなし**
- Flash: Puya PY25Q128HA 16MB
- Display: Good Display GDEQ0426T82、SSD1677コントローラ、480×800px 220PPI
- SDカードとディスプレイが**同じSPIバスを共有**（時分割アクセス）
- ボタン6個が**抵抗ラダー方式**でGPIO1/GPIO2の2本に集約（同時押し検出不可）

### フラッシュパーティション構成

```
nvs       0x9000   20KB
otadata   0xE000    8KB
app0      0x10000   6.25MB（OTA_0）
app1      0x650000  6.25MB（OTA_1）← 0x610000ではなく0x650000が正しい
spiffs    0xC90000  3.375MB
coredump  0xFF0000  64KB
```

> ⚠️ app1の開始アドレスは0x650000。0x610000と間違えると書き込み失敗・フリーズの原因になる。

---

## 開発環境構築の経緯

### macOSのlibexpat問題

macOS 26.2（Tahoe beta）ではHomebrewのPython（3.12/3.13/3.14）がシステムの`libexpat.1.dylib`と不一致を起こし、`pip install platformio`が失敗する。

**解決策**：**Docker（python:3.12-slim）でビルドする**。これでmacOSのlibexpat問題を完全回避。

```bash
docker run --rm \
  -v ~/projects/xteink/crosspoint-reader-jp:/project \
  -w /project \
  python:3.12-slim \
  bash -c "apt-get update -qq && apt-get install -y -qq git && pip install platformio && pio run"
```

### SSH環境

- Mac MiniにTailscale App Store版を入れたが、サンドボックス制限でCLIが使えない
- 解決：Homebrewで`brew install tailscale`してPATHを設定
- macOS Remote Loginを有効化してSSH接続（Tailscale SSHではなく標準sshd）
- MacBook Air → VS Code Remote SSH → Mac Mini、という開発フローが基本

---

## ファームウェア書き込みの手順と注意事項

### 正しい書き込み手順

```bash
# app0（通常起動）
esptool --chip esp32c3 --port /dev/cu.usbmodem101 --baud 921600 \
  write-flash 0x10000 .pio/build/default/firmware.bin

# app1（OTAロールバック用）
esptool --chip esp32c3 --port /dev/cu.usbmodem101 --baud 921600 \
  write-flash 0x650000 .pio/build/default/firmware.bin
```

### ハマりポイント

1. **app1アドレスを0x610000にするとフリーズ**。partitions.csvを見て0x650000と確認。
2. **書き込み後に元のOSが起動する場合**：otadataが古いアプリを指している。両パーティションに書くことで解決。
3. **純正ファームが起動している場合の文鎮化対策**：事前に`read_flash 0x0 0x1000000`でフルバックアップ必須。

---

## 日本語化の実装

### i18nシステムの仕組み

CrossPointのUIは`lib/I18n/translations/*.yaml`で管理されている。

```bash
# YAMLを追加してC++ヘッダを自動生成
python3 scripts/gen_i18n.py lib/I18n/translations lib/I18n/
```

`japanese.yaml`を作成し、264キーを全翻訳。言語コードは`JA`、order=23。

### フォントシステムの構造

フォントは`lib/EpdFont/scripts/fontconvert.py`でTTF/OTF→C++ヘッダに変換する。

```bash
python3 fontconvert.py <name> <size> <font.ttf> [fallback.ttf] \
  --2bit --compress \
  --additional-intervals <min_codepoint>,<max_codepoint>
```

**フォントスタック**：複数フォントファイルを指定すると、先のフォントにないグリフは次のフォントから補完される。→ Ubuntu＋Noto Sans CJKのスタックで英字はUbuntu、日本語はNoto Sans CJKを使うUIフォントを実現。

変換後のヘッダは`lib/EpdFont/builtinFonts/all.h`にincludeし、`src/main.cpp`でオブジェクトを宣言して`renderer.insertFont(FONT_ID, fontFamily)`で登録する。

**フォントIDはSHA256ハッシュから生成**：

```bash
ruby -rdigest -e 'puts [
  "./font_file.h"
].map{|f| Digest::SHA256.hexdigest(File.read(f)).to_i(16) }.sum % (2 ** 32) - (2 ** 31)'
```

このIDを`src/fontIds.h`の`UI_10_FONT_ID`等に設定する。

### EpdFontData.hの型拡張（重要）

日本語フォントはカーニングクラス数が256を超えるため、元のコードがコンパイルエラーになる。以下の型を`uint8_t`→`uint16_t`に変更した：

```cpp
// lib/EpdFont/EpdFontData.h
typedef struct {
  uint16_t width;    // uint8_t → uint16_t
  uint16_t height;   // uint8_t → uint16_t
  ...
} EpdGlyph;

typedef struct {
  uint16_t classId;  // uint8_t → uint16_t（EpdKernClassEntry）
  ...
};

// EpdFontData内
uint16_t kernLeftClassCount;   // uint8_t → uint16_t
uint16_t kernRightClassCount;  // uint8_t → uint16_t
```

### 日本語フォントの構成（Phase 1）

| フォント変数名 | サイズ | 用途 | グリフ範囲 |
|---|---|---|---|
| `ubuntu_jp_10_regular/bold` | 10pt | UIフォント（UI_10_FONT_ID） | ひらがな・カタカナ・漢字195字・全角記号 |
| `ubuntu_jp_12_regular/bold` | 12pt | UIフォント（UI_12_FONT_ID） | 同上 |
| `notosansjp_8_regular` | 8pt | ボタンラベル（SMALL_FONT_ID） | ひらがな・カタカナ・漢字195字 |

**漢字195字の抽出方法**：

```python
# japanese.yamlから漢字コードポイントを抽出
python3 -c "
with open('lib/I18n/translations/japanese.yaml', 'r') as f:
    text = f.read()
kanji = sorted(set(c for c in text if '\u4e00' <= c <= '\u9fff'))
print(' '.join([f'--additional-intervals 0x{ord(c):04X},0x{ord(c):04X}' for c in kanji]))
"
```

---

## 今後の実装方針

### Phase 2：SDカードからの動的フォントロード

- `FontCacheManager`にSDフォントのロードロジックを追加
- SDに`/fonts/NotoSansCJKjp.bin`（fontconvert済みバイナリ）を配置
- グリフ検索時にFlashになければSDから該当グループだけ展開してRAMにキャッシュ
- LRUキャッシュで古いグリフを追い出す

### Phase 3：コンテンツビューア

**TEXT/Markdown**：
- CrossPointのEPUBレンダラー（HTML処理パイプライン）を流用
- Markdown → HTML変換に[md4c](https://github.com/mity/md4c)（C実装の軽量MDパーサ）を検討

**画像ビューア**：
- WiFiアップロード時にエッジ側で前処理（グレースケール化・480×800リサイズ・ディザリング）
- フォルダ単位でスライドショー表示

**PDF**：
- PC側でpdftoppm/PyMuPDFでページ画像に変換してSDに転送
- X4側は連番画像フォルダとして扱う（画像ビューアで代替）

---

## ファイル構成のポイント

```
crosspoint-reader-jp/
├── lib/
│   ├── EpdFont/
│   │   ├── EpdFontData.h          ← 型をuint16_tに拡張（重要）
│   │   ├── builtinFonts/
│   │   │   ├── all.h              ← フォントヘッダのinclude一覧
│   │   │   ├── ubuntu_jp_*.h      ← 日本語UIフォント（生成物）
│   │   │   ├── notosansjp_*.h     ← 日本語ボタンフォント（生成物）
│   │   │   └── source/
│   │   │       └── NotoSansJP/    ← NotoSansCJKjp-Regular.otf を配置
│   │   └── scripts/
│   │       └── fontconvert.py     ← フォント変換スクリプト
│   └── I18n/
│       └── translations/
│           └── japanese.yaml      ← 日本語翻訳（264キー）
├── src/
│   ├── fontIds.h                  ← フォントID定数（SHA256ハッシュ由来）
│   └── main.cpp                   ← フォントオブジェクト宣言・登録
└── DEVNOTES.md                    ← このファイル
```
