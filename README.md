# App_IO_CX

App_IO_CXは、TWELITE（モノワイヤレス製）向けのデジタルIO伝送アプリケーションです。標準アプリ（App_Twelite）のデジタルIO伝送機能に特化し、最大12ポートまで利用できるように拡張されています。

## 概要

本アプリケーションは、App_IOをベースに専用プログラム化したものです。詳細な修正内容については、[README_App_IO.md](README_App_IO.md)を参照してください。

## 仕様

### 動作モード

M1ピンで動作モードを切り替えます：

| M1 | モード | 説明 |
|----|--------|------|
| O (0) | `E_IO_MODE_CHILD_CONT_TX` | 連射子機 |
| G (1) | `E_IO_MODE_PARNET` | 親機 |

### チャネル設定

EI1とEI2ピンの組み合わせでチャネルを設定します：

| EI1 | EI2 | チャネル設定値 | 使用チャネル |
|-----|-----|--------------|------------|
| O (0) | O (0) | 0 | デフォルト/フラッシュ値 |
| G (1) | O (0) | 1 | チャネル12 |
| O (0) | G (1) | 2 | チャネル21 |
| G (1) | G (1) | 3 | チャネル25 |

**注意**: ウォッチドッグ出力機能が有効な場合、`PORT_EI2`は使用されず、`PORT_EI1`のみで制御（0または1の2通り）されます。

### ポート設定

入出力ポートは固定設定（SET3）で動作します：

| 設定 | 子機 | 親機 |
|------|------|------|
| 入力ポート数 | 6 | 6 |
| 出力ポート数 | 6 | 6 |
| 入力ポート | INPUT1-4, EO1, EO2 | INPUT1-4, I2C_CLK, I2C_DAT |
| 出力ポート | OUT1-4, I2C_CLK, I2C_DAT | OUT1-4, EO1, EO2 |

### 主な機能

- **低レイテンシモード**: 常時有効（1ms周期、割り込み駆動）
- **デジタルIO伝送**: 最大12ポート（入力6ポート、出力6ポート）
- **インタラクティブモード**: シリアルポート経由での設定・デバッグ
- **ウォッチドッグ出力**: オプション機能（有効時、PORT_EI2からパルス出力）

### 削除された機能

以下の機能は削除されています：

- 中継機能（ROUTERモード）
- スリープモード（1秒/10秒間欠子機）
- 通常子機モード
- ADC（アナログ-デジタル変換器）機能
- IO設定要求パケット

## コンパイル手順

### 必要な環境

- **MWSDK**: Mono Wireless SDK（TWELITE SDK）
- **Make**: GNU Make
- **GCC**: ARM用のGCCコンパイラ（MWSDKに含まれる）

### ビルド手順

1. **MWSDKのパスを設定**

   環境変数`MWSDK_ROOT`にMWSDKのインストールパスを設定します：

   ```bash
   export MWSDK_ROOT=/path/to/MWSDK
   ```

2. **ビルドディレクトリに移動**

   ```bash
   cd App_IO_CX/build
   ```

3. **コンパイル実行**

   ```bash
   make
   ```

   または、特定のTWELITEモデルを指定する場合：

   ```bash
   make TWELITE=BLUE
   ```

   サポートされているアーキテクチャ：
   - `BLUE`（デフォルト）
   - `RED`
   - `GOLD`

4. **ビルド成果物**

   ビルドが成功すると、以下のファイルが生成されます：

   - `App_IO_CX.bin`: バイナリファイル
   - `App_IO_CX.hex`: Intel HEX形式ファイル

   生成先ディレクトリは、`App_IO_CX/build/`配下のビルド構成に応じたディレクトリです。

### ビルドオプション

#### 言語設定

デフォルトは日本語（JP）です。英語に変更する場合：

```bash
make TWE_LANG_PREF=EN
```

サポートされている言語：
- `JP`（日本語）
- `EN`（英語）

#### デバッグ出力

デバッグ出力はデフォルトで有効です（`-DDEBUG_OUTPUT`）。無効にする場合は、`App_IO_CX/build/Makefile`の以下の行をコメントアウト：

```makefile
# CFLAGS += -DDEBUG_OUTPUT
```

#### 開発キット002L対応

開発キット002Lを使用する場合：

```bash
make TWE_DEVKIT=002L
```

### クリーンビルド

オブジェクトファイルを削除して再ビルドする場合：

```bash
make clean
make
```

### トラブルシューティング

#### エラー: MWSDK_ROOT not set

環境変数`MWSDK_ROOT`が設定されていません。MWSDKのパスを設定してください。

#### エラー: cannot find mw.mk

MWSDKのパスが正しく設定されていないか、MWSDKが正しくインストールされていません。

#### コンパイルエラー

- GCCのバージョンを確認してください（MWSDKに含まれるGCCを使用）
- ソースファイルのパスが正しいか確認してください
- `make clean`を実行してから再ビルドしてください

## ファイル構成

```
App_IO_CX/
├── README.md              # 本ファイル（仕様とコンパイル手順）
├── README_App_IO.md       # App_IOからの修正内容ドキュメント
├── Version.mk             # バージョン情報
├── makefile.defs          # Makefile定義
├── App_IO_CX/             # アプリケーションソース
│   ├── App_IO_CX.c        # メインアプリケーションファイル
│   ├── App_IO_CX.h        # ヘッダファイル
│   └── build/             # ビルド設定
│       └── Makefile       # ビルド用Makefile
└── Common/                # 共通ソース
    ├── Interactive.c      # インタラクティブモード処理
    ├── common.c           # 共通処理
    ├── Pairing.c          # ペアリング処理
    ├── duplicate_checker.c # 重複チェック処理
    └── twesettings_std_defsets.c # デフォルト値定義
```

## バージョン情報

- **バージョン**: 1.0.1（Version.mk参照）

## ライセンス

本ソフトウェアは、MW-SLA-*J,*E（MONO WIRELESS SOFTWARE LICENSE AGREEMENT）の下でリリースされています。

詳細は以下のファイルを参照してください：
- `MW-SLA-1J.txt`（日本語）
- `MW-SLA-1E.txt`（英語）

## 参考情報

- [TWELITE公式サイト](https://mono-wireless.com/)
- [App_IO公式ページ](https://mono-wireless.com/jp/products/TWE-APPS/App_IO/)
- [README_App_IO.md](README_App_IO.md) - App_IOからの修正内容の詳細

