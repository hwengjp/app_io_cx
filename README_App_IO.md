# App_IO_CX.c 修正内容ドキュメント

> **注意**: 本ドキュメントは、App_IOからの修正内容を詳細に記述したものです。  
> 仕様とコンパイル手順については、[README.md](README.md)を参照してください。

## 概要

本ドキュメントは、`App_IO_CX.c`（旧`App_IO.c`）に対して実施した修正内容をまとめたものです。
専用プログラム化のため、以下の変更を行いました：

1. アプリケーション名の変更（`App_IO` → `App_IO_CX`）
2. モード設定の簡素化（M1のみ使用、親機・連射子機のみ対応）
3. 不要なモード処理の削除
4. 中継処理の削除
5. シリアルポート処理の整理（インタラクティブモード用に一部残存）
6. 入出力設定のオプションビット削除（固定設定化）
7. 未使用パケットタイプの削除（IO設定要求パケット）
8. MULTINONE統合フレームワーク関連コードの削除
9. ADC関連コメントの削除
10. 低レイテンシモード以外のコードの削除
11. スリープモード処理の削除（`vProcessEvCoreSlp`関数）
12. 標準ライブラリのデフォルト値をローカルコードに取り込み
13. 未使用ライブラリフォルダの削除（`libApp_IO`、`App_IO_CX_BAK`）

---

## 1. モード設定の簡素化

### 1.1 変更内容

- **変更前**: M1, M2, M3の3ピンで8通りのモード設定
- **変更後**: M1のみで2通りのモード設定

### 1.2 対応モード

| M1 | モード | 説明 |
|----|--------|------|
| O (0) | `E_IO_MODE_CHILD_CONT_TX` | 連射子機 |
| G (1) | `E_IO_MODE_PARNET` | 親機 |

### 1.3 削除されたモード

以下のモードは削除されました：

- `E_IO_MODE_CHILD` (通常子機)
- `E_IO_MODE_ROUTER` (中継機)
- `E_IO_MODE_CHILD_SLP_1SEC` (1秒スリープ子機)
- `E_IO_MODE_CHILD_SLP_10SEC` (10秒スリープ子機)

### 1.4 修正箇所

#### モード読み取り（App_IO_CX.c）

```c
// モード設定（M1のみ使用：M1=O→CHILD_CONT_TX, M1=G→PARNET）
if (!f_warm_start) {
    vPortAsInput(PORT_CONF1);
    sAppData.u8Mode = bPortRead(PORT_CONF1) ? E_IO_MODE_PARNET : E_IO_MODE_CHILD_CONT_TX;
}
```

#### モード依存の初期値設定（App_IO_CX.c）

```c
// 各モード依存の初期値の設定など
switch(sAppData.u8Mode) {
case E_IO_MODE_PARNET:
    sAppData.u8AppLogicalId = LOGICAL_ID_PARENT;
    break;

case E_IO_MODE_CHILD_CONT_TX:
    sAppData.u8AppLogicalId = LOGICAL_ID_CHILDREN;
    break;

default: // 未定義機能なので、SILENT モードにする。
    sAppData.u8AppLogicalId = 255;
    sAppStg.u8role = E_APPCONF_ROLE_SILENT;
    break;
}
```

#### 状態遷移マシンの登録（App_IO_CX.c）

`E_IO_MODE_PARNET`と`E_IO_MODE_CHILD_CONT_TX`のみに対応するように簡素化。

---

## 2. 中継処理の削除

### 2.1 変更内容

中継機（ROUTER）モードが削除されたため、パケットの中継処理を削除しました。

### 2.2 修正箇所

#### 受信関数からの削除

以下の関数から中継処理を削除：

- `vReceiveIoData()` (App_IO_CX.c)

削除されたコード例：

```c
// 削除前
if (u8TxFlag == 0 && sAppData.u8Mode == E_IO_MODE_ROUTER) {
    // 中継処理
    i16TransmitRepeat(pRx);
}

// 削除後
(void)u8TxFlag; // 未使用変数の警告回避
```

#### 状態遷移からの削除

`vProcessEvCorePwr()`関数から`E_STATE_APP_RUNNING_ROUTER`への遷移を削除。

---

## 3. シリアルポート処理の整理

### 3.1 変更内容

シリアルポート（UART）関連の処理を整理しました。インタラクティブモードで使用されるため、完全には削除されていませんが、不要な機能を削除しました。

### 3.2 削除された機能

- 起動時メッセージ表示（`vSerInitMessage()`関数）
- IO設定要求のシリアルコマンド処理（`vProcessSerialCmd()`関数内の`SERCMD_ID_REQUEST_IO_DATA`ケース）

### 3.3 残されている機能

以下の機能はインタラクティブモードで使用されるため、残されています：

- UART初期化処理（`vSerialInit()`）
- シリアル入力処理（`vHandleSerialInput()`）
- シリアルコマンド処理（`vProcessSerialCmd()` - シリアルメッセージ送信のみ）
- シリアルメッセージパケットの送受信（`i16TransmitSerMsg()`、`vReceiveSerialMsg()`）

### 3.4 修正箇所

#### シリアルコマンド処理からの削除（App_IO_CX.c `vProcessSerialCmd()`関数）

```c
// 削除前
if (u8cmd == SERCMD_ID_REQUEST_IO_DATA) {
    // IO設定要求の処理
    tsIOSetReq sIOreq;
    // ... 処理 ...
    i16TransmitIoSettingRequest(u8addr, &sIOreq);
}

// 削除後
// IO設定要求処理は削除され、シリアルメッセージ送信のみ残る
```

---

## 4. その他の修正

### 4.1 ボタン監視マスクの調整

ウォッチドッグ出力機能が有効な場合、`PORT_EI2`を監視対象から除外。

```c:App_IO_CX/App_IO_CX.c
// ボタン監視の有効化
if (IS_APPCONF_OPT_WATCHDOG_OUTPUT()) {
    sAppData.sBTM_Config.bmPortMask = gc_u32_PORT_INPUT_MASK | (1UL << PORT_EI1);
} else {
    sAppData.sBTM_Config.bmPortMask = gc_u32_PORT_INPUT_MASK | (1UL << PORT_EI1) | (1UL << PORT_EI2);
}
```

### 4.2 チャネル設定の制限

ウォッチドッグ出力機能が有効な場合、`PORT_EI2`をチャネル設定から除外。

```c:App_IO_CX/App_IO_CX.c
if (IS_APPCONF_OPT_WATCHDOG_OUTPUT()) {
    sAppData.u8ChCfg &= 1; // ２ビット目をマスクしておく
}
```

### 4.3 入出力設定の固定化

#### 4.3.1 変更内容

入出力ポート設定を決定するオプションビット（`PORT_TBL1`、`PORT_TBL2`）を削除し、固定設定に変更しました。

- **変更前**: オプションビット `PORT_TBL1` (0x1000) と `PORT_TBL2` (0x2000) の組み合わせで4通りのポートテーブルから選択
- **変更後**: 常にSET3（テーブル番号2）を使用：子機入力６・出力６（親機出力６・入力６）

#### 4.3.2 固定設定の詳細

| 設定 | 子機 | 親機 |
|------|------|------|
| 入力ポート数 | 6 | 6 |
| 出力ポート数 | 6 | 6 |
| 入力ポート | INPUT1-4, EO1, EO2 | INPUT1-4, I2C_CLK, I2C_DAT |
| 出力ポート | OUT1-4, I2C_CLK, I2C_DAT | OUT1-4, EO1, EO2 |

#### 4.3.3 修正箇所

ポートテーブル決定処理（App_IO_CX.c）

```c
// 変更前
// ポートの入出力を決定
sAppData.u8IoTbl = IS_APPCONF_OPT_PORT_TBL1() ? 1 : 0;
sAppData.u8IoTbl += IS_APPCONF_OPT_PORT_TBL2() ? 2 : 0;
MWPFX(bPortTblInit)(sAppData.u8IoTbl, IS_LOGICAL_ID_PARENT(sAppData.u8AppLogicalId));

// 変更後
// ポートの入出力を決定（固定設定：子機入力６・出力６、親機出力６・入力６）
sAppData.u8IoTbl = 2; // SET3 (6:6) に固定
MWPFX(bPortTblInit)(sAppData.u8IoTbl, IS_LOGICAL_ID_PARENT(sAppData.u8AppLogicalId));
```

#### 4.3.4 影響

- オプションビット `E_APPCONF_OPT_PORT_TBL1` (0x1000) と `E_APPCONF_OPT_PORT_TBL2` (0x2000) は無視されます
- フラッシュ設定にこれらのオプションビットが設定されていても、動作には影響しません
- 常に「子機入力６・出力６（親機出力６・入力６）」の設定で動作します

---

## 5. 未使用パケットタイプの削除

### 5.1 変更内容

使用されていないパケットタイプとその関連コードを削除しました。

### 5.2 削除されたパケットタイプ

#### 5.2.1 IO設定要求パケット (`TOCONET_PACKET_CMD_APP_USER_IO_DATA_EXT`)

- **削除理由**: 送信関数の呼び出し箇所がなく、実質的に使用されていない
- **削除内容**:
  - 送信関数: `i16TransmitIoSettingRequest()` 関数全体（約68行）
  - 受信関数: `vReceiveIoSettingRequest()` 関数全体（約63行）
  - 関数プロトタイプ宣言（2箇所）
  - 受信コールバックでのディスパッチ処理
  - `vProcessSerialCmd()`関数内のIO設定要求処理（`SERCMD_ID_REQUEST_IO_DATA`ケース）
  - データ構造体: `tsIOSetReq` 構造体定義（`App_IO_CX.h`）
  - パケットタイプ定義: `TOCONET_PACKET_CMD_APP_USER_IO_DATA_EXT`（`Common/common.h`）
  - コメント内の参照

### 5.3 修正箇所

#### 関数プロトタイプ宣言の削除（App_IO_CX.c）

```c
// 削除前
static void vReceiveIoData(tsRxDataApp *pRx);
static void vReceiveIoSettingRequest(tsRxDataApp *pRx);

static int16 i16TransmitIoData(uint8 u8Quick);
static int16 i16TransmitIoSettingRequest(uint8 u8DstAddr, tsIOSetReq *pReq);

// 削除後
static void vReceiveIoData(tsRxDataApp *pRx);

static int16 i16TransmitIoData(uint8 u8Quick);
```

#### 受信コールバックでのディスパッチ処理の削除（App_IO_CX.c）

```c
// 削除前
switch (psRx->u8Cmd) {
case TOCONET_PACKET_CMD_APP_USER_IO_DATA: // IO状態の伝送
    vReceiveIoData(psRx);
    break;
case TOCONET_PACKET_CMD_APP_USER_IO_DATA_EXT: // IO状態の伝送(UART経由)
    vReceiveIoSettingRequest(psRx);
    break;
}

// 削除後
switch (psRx->u8Cmd) {
case TOCONET_PACKET_CMD_APP_USER_IO_DATA: // IO状態の伝送
    vReceiveIoData(psRx);
    break;
}
```

#### シリアルコマンド処理からの削除（App_IO_CX.c `vProcessSerialCmd()`関数）

```c
// 削除前
if (u8cmd == SERCMD_ID_REQUEST_IO_DATA) {
    // IO設定要求の処理
    tsIOSetReq sIOreq;
    // ... 処理 ...
    i16TransmitIoSettingRequest(u8addr, &sIOreq);
}

// 削除後
// IO設定要求処理は削除され、シリアルメッセージ送信のみ残る
```

#### データ構造体の削除（App_IO_CX.h）

```c
// 削除前
/** @ingroup MASTER
 * IO 設定要求
 */
typedef struct {
    uint16 u16IOports;          //!< 出力IOの状態 (1=Lo, 0=Hi)
    uint16 u16IOports_use_mask; //!< 設定を行うポートなら TRUE
} tsIOSetReq;

// 削除後
（構造体定義を削除）
```

### 5.4 影響

- 削除されたパケットタイプは受信されても処理されません（パケット種別の不一致で無視されます）
- コードサイズが削減されました（約270行のコード削除）
- 保守性が向上しました（未使用コードの削除により、コードベースが簡潔になりました）

**注意**: `TOCONET_PACKET_CMD_APP_USER_SERIAL_MSG`（シリアルメッセージパケット）は削除されていません。インタラクティブモードで使用されているため、`i16TransmitSerMsg()`関数と`vReceiveSerialMsg()`関数は残されています。

---

## 6. MULTINONE統合フレームワーク関連コードの削除

### 6.1 変更内容

複数のアプリケーションを1つのファームウェアに統合するためのMULTINONEフレームワーク関連コードを削除しました。本アプリケーションは単一アプリケーションとして動作するため、これらのコードは不要です。

### 6.2 削除されたコード

#### 6.2.1 App_IO_CX/App_IO_CX.c

- **MultInOne.h のインクルード**
- **TWENET_REG_CBS() 関数** - コールバック関数の登録とメモリ割り当て
- **関数プロトタイプのマクロ定義** - コールバック関数名のリネーム
- **グローバル変数の条件分岐**

#### 6.2.2 App_IO_CX/App_IO_CX.h

- **MultInOne.h のインクルード**（24-26行目）
- **sAppData の条件分岐**（152-156行目）- グローバル変数宣言とマクロ定義の分岐
- **sAppDataExt の条件分岐**（233-237行目）- グローバル変数宣言とマクロ定義の分岐

#### 6.2.3 Common/config.h

- **設定マクロの条件分岐**（27-43行目）- MULTINONE用の動的設定と通常ビルド用の固定設定
- **NVMEM設定の条件分岐**（45-50行目）- MULTINONE用のセクター設定

#### 6.2.4 Common/twenet_defs.h

- **ToCoNetモジュール選択の条件分岐**（5-9行目）- MULTINONE用キューと通常キュー

#### 6.2.5 Common/Interactive.c

- **関数名マクロ定義**（54-59行目）- コールバック関数名のリネーム
- **tsFinal構造体宣言の条件分岐**（276-281行目）
- **tsFinal構造体初期化の条件分岐**（379-386行目）

### 6.3 修正箇所

#### グローバル変数の宣言（App_IO_CX.c）

```c
// 削除前
#ifndef MWLIB_MULTINONE
tsAppData sAppData; //!< アプリケーションデータ  @ingroup MASTER
tsAppDataExt sAppDataExt; //!< アプリケーションデータ、その他  @ingroup MASTER
#endif

// 削除後
tsAppData sAppData; //!< アプリケーションデータ  @ingroup MASTER
tsAppDataExt sAppDataExt; //!< アプリケーションデータ、その他  @ingroup MASTER
```

#### ハードウェア初期化（Common/config.h 27-43行目）

```c
// 削除前
#ifdef MWLIB_MULTINONE
extern uint32 _MULTINONE_U32_APP_CONFIG;
# define FW_CONF_ID() ((_MULTINONE_U32_APP_CONFIG>>16)&0xFF)
// ... その他のマクロ定義 ...
#else
# define IS_OPT_INTRCT_ON_BOOT() (0)
# define IS_CONF_NORMAL() (1)
// ... その他のマクロ定義 ...
#endif

// 削除後
# define IS_OPT_INTRCT_ON_BOOT() (0)
# define IS_CONF_NORMAL() (1)
# define FW_CONF_ID() (1)
# define USE_LANG_INTERACTIVE_DEFAULT 2
# ifndef USE_LANG_INTERACTIVE
#  define USE_LANG_INTERACTIVE USE_LANG_INTERACTIVE_DEFAULT
# endif
# define FW_CONF_LANG() USE_LANG_INTERACTIVE
```

### 6.4 影響

- コードが単一アプリケーション専用に簡素化されました
- グローバル変数は常に静的に宣言されます（動的メモリ割り当てなし）
- 設定はコンパイル時に固定値として決定されます
- コードサイズが削減されました（約100行以上のコード削除）

---

## 7. ADC関連コメントの削除

### 7.1 変更内容

本アプリケーションはデジタル入出力のみを使用するため、ADC（アナログ-デジタル変換器）関連のコメントを削除しました。

### 7.2 削除されたコメント

#### 7.2.1 App_IO_CX/App_IO_CX.c

- **関数コメントの修正**
  - `ADC/DIの入力状態のチェック` → `DIの入力状態のチェック`
- **コメント行の削除**
  - `- ADC の完了確認` の行を削除
- **ADCイベントハンドラの削除**
  - `case E_AHI_DEVICE_ANALOGUE:` ケース全体を削除
- **割り込みハンドラのコメント修正**
  - `- ADCの実行管理` の行を削除
- **ハードウェア初期化コメントの修正**（2箇所）
  - `- ADC3/4 のプルアップ停止` の行を削除

#### 7.2.2 Common/app_event.h

- **未使用イベントの削除**（17行目）
  - `E_EVENT_APP_ADC_COMPLETE` イベント定義を削除
- **状態名コメントの修正**（28行目）
  - `最初のADCやDIの状態確定を待つ` → `最初のDIの状態確定を待つ`

### 7.3 修正箇所

#### イベント定義の削除（app_event.h 13-20行目）

```c
// 削除前
typedef enum
{
    E_EVENT_APP_BASE = ToCoNet_EVENT_APP_BASE,
    E_EVENT_APP_TICK_A,                    //!< 64FPS のタイマーイベント
    E_EVENT_APP_ADC_COMPLETE,              //!< ADC完了
    E_EVENT_APP_TX_COMPLETE,                //!< TX完了
    E_EVENT_APP_RX_COMPLETE                //!< TX完了
} teEventApp;

// 削除後
typedef enum
{
    E_EVENT_APP_BASE = ToCoNet_EVENT_APP_BASE,
    E_EVENT_APP_TICK_A,                    //!< 64FPS のタイマーイベント
    E_EVENT_APP_TX_COMPLETE,                //!< TX完了
    E_EVENT_APP_RX_COMPLETE                //!< RX完了
} teEventApp;
```

### 7.4 影響

- コードがデジタル入出力専用であることが明確になりました
- 誤解を招くコメントが削除され、保守性が向上しました

---

## 8. 低レイテンシモード以外のコードの削除

### 8.1 変更内容

本アプリケーションは低レイテンシモードのみを使用するため、低レイテンシモードでない場合の処理コードを削除し、常に低レイテンシモードで動作するように簡素化しました。

### 8.2 削除されたコード

#### 8.2.1 ハードウェア初期化（割り込み設定）

**場所**: `vInitHardware_IOs()` 関数

```c
// 削除前
if (sAppStg.u32Opt & E_APPCONF_OPT_LOW_LATENCY_INPUT) {
    // 割り込みを有効にする
    vAHI_DioInterruptEnable(gc_u32_PORT_INPUT_MASK, 0);
    // ... 割り込みエッジ設定 ...
} else {
    vAHI_DioInterruptEnable(0, gc_u32_PORT_INPUT_MASK); // 割り込みを無効化
}

// 削除後
// 低レイテンシで入力を行う処理（割り込みを有効にする）
vAHI_DioInterruptEnable(gc_u32_PORT_INPUT_MASK, 0);
// ... 割り込みエッジ設定 ...
```

#### 8.2.2 IO変化による送信処理

**場所**: `vProcessEvCorePwr()` 関数

```c
// 削除前
if (IS_APPCONF_OPT_ON_PRESS_TRANSMIT() && IS_APPCONF_OPT_LOW_LATENCY_INPUT()) {
    // 低レイテンシとリモコンモードでの通常ボタンは割り込み検出のみで送信する
} else {
    // IO変化あり
    bTxCond |= sAppData.sIOData_now.u32BtmChanged ? TRUE : FALSE;
}

// 削除後
if (IS_APPCONF_OPT_ON_PRESS_TRANSMIT()) {
    // リモコンモードでの通常ボタンは割り込み検出のみで送信する
} else {
    // IO変化あり
    bTxCond |= sAppData.sIOData_now.u32BtmChanged ? TRUE : FALSE;
}
```

#### 8.2.3 送信速度の判定

**場所**: `vProcessEvCorePwr()` 関数

```c
// 削除前
bool_t bQuick = FALSE;
if (sAppData.sIOData_now.u32BtmChanged && (sAppStg.u32Opt & E_APPCONF_OPT_LOW_LATENCY_INPUT)) {
    bQuick = TRUE;
}

// 削除後
bool_t bQuick = sAppData.sIOData_now.u32BtmChanged ? TRUE : FALSE;
```

#### 8.2.4 タイマー周波数の設定

**場所**: `cbAppColdStart()` 関数

```c
// 削除前
if (sAppStg.u32Opt & E_APPCONF_OPT_LOW_LATENCY_INPUT) {
    sToCoNet_AppContext.u16TickHz = HZ_LOW_LATENCY; // 低レイテンシモードでは 1KHz 動作
    sAppData.u16ToCoNetTickDelta_ms = 1000/HZ_LOW_LATENCY;
}
sAppData.u16ToCoNetTickDelta_ms = 1000 / sToCoNet_AppContext.u16TickHz;

// 削除後
sToCoNet_AppContext.u16TickHz = HZ_LOW_LATENCY; // 低レイテンシモードでは 1KHz 動作
sAppData.u16ToCoNetTickDelta_ms = 1000/HZ_LOW_LATENCY;
```

#### 8.2.5 TICK_TIMERイベントでの送信処理

**場所**: `cbToCoNet_vHwEvent()` 関数

```c
// 削除前
if (gc_bLowLatencyTxCond == 0xFF) {
    if (sAppData.u8IOFixState == E_IO_FIX_STATE_READY) {
        if ((sAppStg.u32Opt & E_APPCONF_OPT_LOW_LATENCY_INPUT)
            && !IS_APPCONF_OPT_ON_PRESS_TRANSMIT())
        ) {
            gc_bLowLatencyTxCond = TRUE;
        } else {
            gc_bLowLatencyTxCond = FALSE;
        }
    }
}

// 削除後
if (gc_bLowLatencyTxCond == 0xFF) {
    if (sAppData.u8IOFixState == E_IO_FIX_STATE_READY) {
        if (!IS_APPCONF_OPT_ON_PRESS_TRANSMIT()) {
            gc_bLowLatencyTxCond = TRUE;
        } else {
            gc_bLowLatencyTxCond = FALSE;
        }
    }
}
```

### 8.3 低レイテンシモードの動作

削除後、アプリケーションは常に以下の動作を行います：

- **DI割り込み**: 常に有効（割り込み駆動方式）
- **タイマー周波数**: 常に1000Hz（1ms周期）
- **送信タイミング**: 割り込み検出時またはIO変化検出時
- **送信速度**: IO変化時は常に高速送信（`bQuick = TRUE`）

### 8.4 影響

- コードが低レイテンシモード専用に簡素化されました
- 条件分岐が削減され、コードの可読性が向上しました
- オプションビット `E_APPCONF_OPT_LOW_LATENCY_INPUT` のチェックが不要になりました（常に有効として動作）

---

## 9. スリープモード処理の削除

### 9.1 変更内容

スリープモード（1秒/10秒間欠子機）が削除されたため、スリープモード用の状態遷移マシン関数を削除しました。

### 9.2 削除された関数

- **`vProcessEvCoreSlp()` 関数** - スリープ稼動モード用の状態遷移マシン（約300行）
  - プロトタイプ宣言も削除

### 9.3 影響

- コードサイズが削減されました（約300行のコード削除）
- 常時通電モード（`vProcessEvCorePwr()`）のみが使用されます

---

## 10. 標準ライブラリのデフォルト値をローカルコードに取り込み

### 10.1 変更内容

インタラクティブモードで表示されるデフォルト値（特に送信出力と再送回数 `x: 0x03`）を標準ライブラリからローカルコードに取り込み、カスタマイズ可能にしました。

### 10.2 取り込んだファイル

以下のファイルを標準ライブラリから`Common/`ディレクトリにコピー：

- `twesettings_std_defsets.h` - ヘッダファイル
- `twesettings_std_defsets.c` - 実装ファイル（デフォルト値定義）

### 10.3 修正箇所

#### Makefileへの追加（App_IO_CX/build/Makefile）

```makefile
APPSRC += twesettings_std_defsets.c
```

#### デフォルト値の定義（Common/twesettings_std_defsets.c）

- **送信出力と再送回数**: 59-62行目
  - デフォルト値: `0x03`（再送: デフォルト2回, 出力: レベル3=最大）
- **UARTボーレート**: 72-75行目
  - デフォルト値: `1152`（115200 bps）

### 10.4 デフォルト値の変更方法

`Common/twesettings_std_defsets.c`の該当箇所を編集することで、デフォルト値を変更できます。

### 10.5 影響

- デフォルト値をローカルでカスタマイズ可能になりました
- 標準ライブラリへの依存を減らし、保守性が向上しました

---

## 11. 未使用ライブラリフォルダの削除

### 11.1 削除されたフォルダ

- **`libApp_IO/`** - MULTINONE統合フレームワーク用のライブラリフォルダ
  - 単一アプリケーションとして使用するため不要
- **`App_IO_CX_BAK/`** - バックアップフォルダ
  - 修正完了後、不要になったため削除

### 11.2 影響

- プロジェクト構造が簡潔になりました
- 不要なファイルが削除され、保守性が向上しました

---

## 12. 動作仕様

### 12.1 モード切り替え

- **M1=O (0)**: 連射子機モード（`E_IO_MODE_CHILD_CONT_TX`）
- **M1=G (1)**: 親機モード（`E_IO_MODE_PARNET`）

### 12.2 チャネル設定

| EI1 | EI2 | チャネル設定値 | 使用チャネル |
|-----|-----|--------------|------------|
| O (0) | O (0) | 0 | デフォルト/フラッシュ値 |
| G (1) | O (0) | 1 | チャネル12 |
| O (0) | G (1) | 2 | チャネル21 |
| G (1) | G (1) | 3 | チャネル25 |

**注意**: ウォッチドッグ出力機能が有効な場合、`PORT_EI2`は使用されず、`PORT_EI1`のみで制御（0または1の2通り）。

### 12.3 削除された機能

- シリアルポート通信
- 中継機能
- スリープモード（1秒/10秒間欠子機）
- 通常子機モード

---

## 13. コンパイル・動作確認

### 10.1 リンターエラー

修正後、リンターエラーはありません。

### 10.2 動作確認項目

以下の動作を確認してください：

1. **モード切り替え**
   - M1=O で連射子機として動作
   - M1=G で親機として動作

2. **チャネル設定**
   - EI1, EI2の組み合わせでチャネルが切り替わる
   - ウォッチドッグ出力有効時はEI1のみで制御

3. **無線通信**
   - 親機と子機間で正常に通信できる
   - 中継機能が動作しないことを確認

4. **ウォッチドッグ出力**
   - オプションビット有効時、PORT_EI2からパルスが出力される

---

## 14. 注意事項

### 11.1 互換性

- 本修正により、以前のバージョンとの互換性は失われます
- 中継機として動作していたデバイスは使用できません
- シリアルポート経由の設定・デバッグはできません

### 14.2 今後の拡張

必要に応じて、以下の機能を追加できます：

- 新しいモードの追加
- その他の専用機能

**注意**: シリアルポート機能はインタラクティブモードで使用されているため、既に実装されています。

---

## 15. 関連ファイル

### 修正されたファイル

- `App_IO_CX/App_IO_CX.c` - メインアプリケーションファイル（旧`App_IO/App_IO.c`）
- `App_IO_CX/App_IO_CX.h` - ヘッダファイル（旧`App_IO/App_IO.h`）
- `Common/twesettings_std_defsets.c` - 標準ライブラリからコピーしたデフォルト値定義
- `Common/twesettings_std_defsets.h` - 標準ライブラリからコピーしたヘッダファイル

### 参照ファイル

- `Common/common.h` - モード定義、チャネルマスク定義
- `Common/common.c` - チャネルマスクプリセットテーブル
- `Common/Interactive.h` - オプションビット定義
- `Common/config.h` - チャネル設定値

---

## 16. 変更履歴

| 日付 | 変更内容 |
|------|---------|
| - | アプリケーション名を`App_IO`から`App_IO_CX`に変更（ファイル名、フォルダ名、設定値など） |
| - | モード設定をM1のみに簡素化 |
| - | 不要なモード処理を削除 |
| - | 中継処理を削除 |
| - | シリアルポート処理を整理（インタラクティブモード用に一部残存） |
| - | 入出力設定のオプションビットを削除し、固定設定（SET3: 6:6）に変更 |
| - | 未使用パケットタイプ（IO設定要求パケット）を削除 |
| - | MULTINONE統合フレームワーク関連コードを削除 |
| - | ADC関連コメントを削除 |
| - | 低レイテンシモード以外のコードを削除 |
| - | スリープモード処理を削除（`vProcessEvCoreSlp`関数） |
| - | 標準ライブラリのデフォルト値をローカルコードに取り込み（`twesettings_std_defsets.c/h`をコピー） |
| - | 未使用ライブラリフォルダを削除（`libApp_IO`、`App_IO_CX_BAK`） |

---

## 17. 参考情報

### 17.1 チャネル設定入力

詳細は、チャネル設定入力機能のドキュメントを参照してください。

### 17.2 ウォッチドッグ出力機能

ウォッチドッグ出力機能の詳細については、関連ドキュメントを参照してください。

---

**作成日**: 2024年
**最終更新**: 2024年


