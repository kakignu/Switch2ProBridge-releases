# Switch2ProBridge

Nintendo Switch 2 Pro Controller を macOS で使うための **ユーザー空間ドライバ**（メニューバー常駐アプリ）の配布ページです。
ソースコードは非公開です。

## ダウンロード

[Releases](https://github.com/kakignu/Switch2ProBridge-releases/releases) から最新版の `Switch2ProBridge.zip` を取得してください。

## ドキュメント

- 📖 **[使い方ガイド（スクリーンショット付き）](docs/USAGE.md)** — インストール、初回起動の許可、キー割り当て、Bluetooth、トラブルシュート
- 🔧 [仕組み](docs/HOW_IT_WORKS.md) — kext / DriverKit を使わないユーザー空間ドライバの構成と設計判断
- 📡 [プロトコルメモ](docs/PROTOCOL.md) — USB / BLE のコマンド・レポート形式
- 🧪 手法を再現した最小サンプル: [switch2pro-macos-sample](https://github.com/kakignu/switch2pro-macos-sample)

## できること

| 機能 | 状態 |
|---|---|
| USB 接続時の初期化 → 入力読み取り（ボタン21個・スティック2本・バッテリー） | ✅ |
| スティック較正値の読み出し | ✅ |
| キー/マウス割り当て（GUI で編集可） | ✅ |
| Bluetooth LE 接続 | ⚠️ 実験的 |
| Apple GameController 対応ゲームで直接使う | ❌ 不可（Apple の制限） |

## 必要環境

- macOS 13 以降（Apple Silicon で確認）

## クイックスタート

> [!IMPORTANT]
> このアプリは Apple の Developer ID で署名・公証していません（ad-hoc 署名）。初回起動時とアップデート時に追加の許可操作が必要です。詳しくは[使い方ガイド](docs/USAGE.md#0-はじめに-このアプリは公証されていないアプリです)を参照してください。

1. zip を展開し `Switch2ProBridge.app` を `/Applications` へ移動
2. 起動 → 「開いていません」と出たら「完了」→ **システム設定 → プライバシーとセキュリティ → 「このまま開く」**
3. コントローラーを USB-C で接続 → メニューバー 🎮 の「USB」が「初期化済み」になる
4. キー割り当てを使う場合: **システム設定 → プライバシーとセキュリティ → アクセシビリティ** で ON → アプリ再起動
5. 「割り当て設定…」で編集、「割り当てを有効にする」を ON（HOME ボタンでも切替）

## 不具合報告

[Issues](https://github.com/kakignu/Switch2ProBridge-releases/issues) へ。メニューの「テスター/ログ」→「コピー」で取得したログを添えてください。

## 謝辞

- プロトコル情報: [ndeadly/switch2_controller_research](https://github.com/ndeadly/switch2_controller_research)
- USB 初期化シーケンスの参考: SDL3 `SDL_hidapi_switch2.c`（zlib License）

Nintendo Switch は任天堂の商標です。本プロジェクトは任天堂とは無関係の非公式ツールです。
