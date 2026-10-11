# Switch2ProBridge の仕組み

ソースコードは非公開ですが、どういう方針・構成で作っているかをここにまとめます。
手法だけを再現した最小サンプルは [switch2pro-macos-sample](https://github.com/kakignu/switch2pro-macos-sample) にあります。
プロトコルの詳細は [PROTOCOL.md](PROTOCOL.md)。

## 方針: ユーザー空間で完結させる

kext（カーネル拡張）も DriverKit（dext）も使っていません。普通の `.app` が IOKit / IOUSBHost / IOHIDManager / CoreBluetooth を使ってコントローラーと通信しています。

| 方式 | 必要なもの | 今回 |
|---|---|---|
| kext | SIP/セキュリティ設定の緩和、ユーザー承認 | 不採用（非推奨） |
| DriverKit (dext) | `com.apple.developer.driverkit.transport.usb` 等の entitlement（対象 VID を Apple に申請。他社 VID は通常不可） | 不採用 |
| ユーザー空間（IOUSBHost / IOHIDManager） | なし（**システムのドライバが掴んでいないインターフェース**に限る） | 採用 |

Switch 2 Pro Controller は USB インターフェースを 2 つ持ち、コマンド用の IF#1（Vendor class 0xFF）は macOS の標準ドライバが掴まないため、entitlement なしでアプリから直接開けます。

## 構成

```
[Switch 2 Pro] --USB--> IF#0 HID (macOS 標準 HID ドライバが掴む)
                   └--> IF#1 vendor bulk (EP 0x02 OUT / 0x82 IN)
                          ↑ IOUSBHostInterface で開いて初期化コマンド送信

初期化後: IF#0 から HID report 0x09 が流れる → macOS からは普通の HID Game Pad
          → IOHIDManager（非排他）で読む → レポート解析
          → キー/マウス割り当て（CGEvent）※アクセシビリティ許可が必要

[Switch 2 Pro] --BLE--> 任天堂独自 GATT（HOGP ではない）
          → CoreBluetooth で接続・初期化・Notify 購読 → 同じレポート解析
```

| モジュール | 役割 |
|---|---|
| Core（純 Swift） | コマンド生成、レポート解析、スティック較正。プラットフォーム非依存でユニットテスト対象 |
| USB 層（Objective-C, IOUSBHost） | デバイス監視、IF#1 の Bulk 送受信 |
| HID リーダー（IOHIDManager） | report 0x09 の受信（他アプリと共有できるよう非排他） |
| BLE マネージャー（CoreBluetooth） | 独自 GATT への接続・初期化 |
| 入力マッパー（CGEvent） | ボタン/スティック → キー・マウス |
| UI（SwiftUI MenuBarExtra） | 状態表示、入力テスター、割り当て設定 |

## 設計判断とハマりどころ

1. **入力レポートは 0x09 を使う**（SDL は 0x05）。0x09 は HID レポートディスクリプタ上「21 ボタン + X/Y/Rx/Rz（12bit）」の標準 Game Pad として記述されているので、macOS 標準 HID がそのまま解釈でき、他アプリからもゲームパッドとして見える。
2. **IOKit のマッチング辞書に数値プロパティを入れない**。`idVendor` 等を指定するとマッチしなかった（OSNumber のサイズ違いと推測）。`IOUSBHostDevice` をクラスだけでマッチし、VID/PID はコードで比較、子から `bInterfaceNumber == 1` を探す。インターフェースの登録はデバイスより遅れるのでリトライする。
3. **アクセシビリティ許可は署名とパスに紐づく**。ad-hoc 署名だとビルドのたびに許可が外れる。安定した署名（Apple Development 等）で署名し、`/Applications` の 1 か所に置く。
4. **SwiftPM の Objective-C ターゲットは ARC 無効がデフォルト** → `-fobjc-arc` が必要。
5. **Bluetooth は `.app` バンドルで起動する**。`NSBluetoothAlwaysUsageDescription` が無いとクラッシュする（`swift run` 直起動は不可）。
6. **IOHIDManager の入力コールバックは report ID を先頭に含む**（USB では 64B、先頭 0x09）。BLE の Notify には ID が無いので解析側で吸収する。
7. UI 更新は約 30Hz に間引き、入力→キー変換は間引かない。

## できないこと

- **Apple GameController.framework 対応ゲームで直接使う**: GameController は既知のコントローラーしか認識せず、Switch 2 Pro は未対応。仮想 HID デバイス（CoreHID `HIDVirtualDevice` / `IOHIDUserDevice`）でなりすますには Apple 承認制の `com.apple.developer.hid.virtual.device` entitlement が必要で、OS 側も仮想デバイスを無視する仕組みがある。
- そのため現実解として **キー/マウス割り当て**を提供している。

## 参考

- [ndeadly/switch2_controller_research](https://github.com/ndeadly/switch2_controller_research)
- SDL3 [`SDL_hidapi_switch2.c`](https://github.com/libsdl-org/SDL/blob/main/src/joystick/hidapi/SDL_hidapi_switch2.c)
- Apple Developer Forums: [Virtual Controllers on Mac](https://developer.apple.com/forums/thread/812774)
