# Switch2ProBridge

Nintendo Switch 2 Pro Controller を macOS で使うための **ユーザー空間ドライバ**（メニューバー常駐アプリ）の配布ページです。
ソースコードは非公開です。

## ダウンロード

[Releases](../../releases) から最新版の `Switch2ProBridge.zip` を取得してください。

## できること

| 機能 | 状態 |
|---|---|
| USB 接続時の初期化 → 入力読み取り（ボタン21個・スティック2本・バッテリー） | ✅ |
| スティック較正値の読み出し | ✅ |
| キー/マウス割り当て（GUI で編集可） | ✅ |
| Bluetooth LE 接続 | ⚠️ 未検証 |
| Apple GameController 対応ゲームで直接使う | ❌ 不可（Apple の制限） |

## 必要環境

- macOS 13 以降（Apple Silicon で確認）

## インストール

1. zip を展開し `Switch2ProBridge.app` を `/Applications` へ移動
2. 初回は Finder で右クリック →「開く」（公証していないため）。開けない場合:
   `xattr -dr com.apple.quarantine /Applications/Switch2ProBridge.app`
3. キー割り当てを使う場合: システム設定 → プライバシーとセキュリティ → **アクセシビリティ** で ON → アプリ再起動

## 使い方

1. コントローラーを USB-C で接続 → メニューバー 🎮 の「USB」が「初期化済み」になる
2. 「割り当て設定…」で割り当てを編集、「割り当てを有効にする」を ON（HOME ボタンでも切替）
3. Bluetooth: 「Bluetoothで接続」→ コントローラー上部の SYNC ボタンを長押し（毎回必要）

設定ファイル: `~/Library/Application Support/Switch2ProBridge/profile.json`

## 不具合報告

[Issues](../../issues) へ。

## 謝辞

- プロトコル情報: [ndeadly/switch2_controller_research](https://github.com/ndeadly/switch2_controller_research)
- USB 初期化シーケンスの参考: SDL3 `SDL_hidapi_switch2.c`（zlib License）

Nintendo Switch は任天堂の商標です。本プロジェクトは任天堂とは無関係の非公式ツールです。
