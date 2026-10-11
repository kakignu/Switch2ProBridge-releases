# Switch 2 Pro Controller プロトコルメモ（本実装で使っている部分）

出典: ndeadly/switch2_controller_research、SDL3 SDL_hidapi_switch2.c、および本プロジェクトの実機ログ。

## USB

- VID `0x057E` / PID `0x2069`、bcdUSB 2.0、構成1・インターフェース2
  - IF#0: HID（Interrupt IN/OUT 64B）— Game Pad。macOS 標準ドライバが掴む
  - IF#1: Vendor (class 0xFF)、Bulk OUT `0x02` / Bulk IN `0x82`（64B）— コマンド用
- 電源投入直後は HID 入力レポートが流れない。IF#1 にコマンド `0x03/0x0D`（Initialise USB）を送ると流れ始める
- Bulk IN の読み出しは 64B 単位（短パケットで終端）

## コマンド形式

```
[0] cmd  [1] 0x91(host→dev) / 0x01(応答)  [2] transport 0x00=USB 0x01=BLE
[3] subcmd  [4] 0x00  [5] data長(要求) / ACK(応答, 0xF8 等)  [6..7] 0x00
[8..] data
```

## 送っている USB 初期化シーケンス（すべて実機で ACK 確認）

| cmd/sub | data | 意味 |
|---|---|---|
| 02/01 ×5 | `00 00 00 00` + addr(LE) | フラッシュ読み出し 0x40B（0x13000 シリアル, 0x13080/0x130C0 工場較正L/R, 0x1FC040/0x1FC080 ユーザー較正L/R） |
| 07/01 | — | 不明（SDL 準拠） |
| 0C/02 | `27 00 00 00` | feature mask 設定 |
| 11/01 | — | 不明 |
| 0A/08 | 20B | 振動データ初期値 |
| 0C/04 | `27 00 00 00` | feature 有効化 |
| 01/0C, 01/01 | — | 不明 |
| 03/0A | `09 00 00 00` | 入力レポートを 0x09 に選択 |
| 03/0D | `01 00 FF FF FF FF FF FF` | USB 初期化（ホストアドレスはダミー可） |
| 09/07 | `mask 00×7` | プレイヤーLED（bit0-3） |

フラッシュ読み出し応答: header 8B + `40 00 00 00` + addr(4B) + data 0x40B（data は応答先頭から offset 16）。

## 較正データ

- 工場較正: ブロック内 offset `0x28` から 9B
- ユーザー較正: 先頭 `B2 A1` のとき offset 2 から 9B
- 9B = neutral(x,y) / max(x,y) / min(x,y) を 12bit×2 で 3B ずつパック
  - `x = b0 | (b1 & 0x0F) << 8`, `y = (b1 >> 4) | b2 << 4`
- 正規化: `(raw - neutral) / (raw < neutral ? min : max)`、上方向が +Y

## 入力レポート 0x09（Pro Controller 2）

USB では先頭に report ID `0x09`（全64B）、BLE では ID なし（63B）。以下は ID を除いたオフセット。

| off | 内容 |
|---|---|
| 0 | カウンタ |
| 1 | 電源: bit0 外部電源, bit1 充電中, bit2-5 残量(0-9) |
| 2-4 | ボタン（下表） |
| 5-7 | 左スティック 12bit×2 |
| 8-10 | 右スティック 12bit×2 |
| 0xE | モーションデータ長, 0xF〜 モーション（未使用） |

| byte | 0x01 | 0x02 | 0x04 | 0x08 | 0x10 | 0x20 | 0x40 | 0x80 |
|---|---|---|---|---|---|---|---|---|
| 0 | B | A | Y | X | R | ZR | + | RS |
| 1 | ↓ | → | ← | ↑ | L | ZL | − | LS |
| 2 | HOME | Capture | GR | GL | C | – | – | – |

実機ログ例: `09 b0 23 02 00 00 …` → A 押下、`09 00 23 20 00 00 …` → ZR。

## Bluetooth LE

- 広告: Manufacturer Data `53 05 01 00 03 7E 05 69 20 …`（会社ID 0x0553、VID/PID 埋め込み）。ペアリングモード（SYNC長押し）で広告
- HOGP ではなく独自 GATT。標準 SMP ペアリングを始めると切断される
- 主なキャラクタリスティック:
  - 入力 0x09: `7492866c-ec3e-4619-8258-32755ffcc0f8`（Notify, CCC に 0x0001）
  - コマンド（Pro, rumble+command）: `3dacbc7e-6955-40b5-8eaf-6f9809e8b379`（WriteWithoutResponse, 先頭に 0x00×0x21 を付けてからコマンド）
  - コマンド（basic）: `649d4ac9-8eb7-4e6c-af44-1ea54fe5f005`
  - 応答（Pro）: `506d9f7d-4278-4e95-a549-326ba77657e0`、応答#1: `c765a961-d9d8-4d36-a20a-5315b111836a`（先頭に 0 埋めあり→ヘッダ探索で解析）
  - 不明 WRITE: `00c5af5d-1964-4e30-8f51-1956f96bd282` に `01 00`
  - 入力キャラクタリスティックの記述子 `679d5510-…` に `85 00`（レポートレート設定？）
- BLE 初期化（ペアリング 0x15 系は省略）: 07/01, 16/01, 0A/02(接続振動), 09/07(LED), 0C/02 `2F`, 11/03, 0C/04 `2F`, 02/04 で較正読み出し → 入力 Notify 有効化
- 本実装の BLE は**未検証**
