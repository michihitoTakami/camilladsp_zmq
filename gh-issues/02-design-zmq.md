## Summary

ZeroMQ 制御インターフェースの設計方針を決定した。`libzmq` 連携の `zmq` crate を採用し、サーバは ROUTER、クライアントは DEALER で多クライアント／非同期に対応する。エンドポイント指定は単一フラグ `--zmq-endpoint`（`tcp://` / `ipc://`）とし、メッセージは WebSocket と同じ JSON `Command` / `Reply` を使う。

## Why this matters

- 依存関係: native `libzmq` に乗るが、成熟度と API 安定性を優先
- 同時接続: ROUTER/DEALER で複数クライアントを公平に捌ける
- 互換性: WebSocket と同じコマンド／返信を再利用し、実装重複を避ける

## Decisions

### 1) Rust crate（採用: `zmq` + system `libzmq`）
- 理由: 最も成熟したバインディングで ZMTP 4.x を広くカバーし、ROUTER/DEALER/PUB/SUB など必要パターンが揃う。
- ビルド: `zmq` は `libzmq` にリンクするため、`zmq-server` feature を **optional** にし、デフォルト OFF（EPIC 方針どおり）。CI/ドキュメントで `libzmq` インストール手順を明示する（例: `apt-get install libzmq3-dev`）。
- 代替検討: Pure Rust 系（例 `zeromq`）は API 安定度と機能網羅性が未成熟のため見送り。

### 2) Socket pattern（採用: ROUTER <-> DEALER）
- サーバ: ROUTER で bind。複数クライアントの同時要求を identity 付きで扱える。
- クライアント: DEALER で connect。送信順序に縛られず再送・並列送信が容易。
- フレーミング: `[identity][empty delimiter][UTF-8 JSON payload]`。返信も同じ identity で返送。
- 将来拡張: 状態通知/メータ更新は別ソケットで PUB/SUB を追加検討（v1 では実装しない）。
- v1 スレッドモデル: 1 スレッドで `zmq::Socket` を poll し、受信ごとに共通 `handle_command` を呼び出すシンプル構成。性能が問題になればワーカー分散を追加。

### 3) Endpoint configuration（採用: 単一フラグ）
- CLI: `--zmq-endpoint <ENDPOINT>` を追加（例: `tcp://127.0.0.1:5555` / `ipc:///tmp/camilladsp.sock`）。
- デフォルト例: 明示的に渡すことを推奨し、未指定なら `tcp://127.0.0.1:5555` を用意しても良い（CLI 実装側で決定）。
- 複数フラグ（address/port 分割）は採用しない。IPC/TCP を一貫して指定できるため。

### 4) Message contract（採用: WebSocket と同じ JSON 形）
- ペイロード: 空フレームを挟んだ末尾フレームを UTF-8 JSON 文字列とし、`WsCommand`/`WsReply`（Refactor で共通化後の `Command`/`Reply`）をシリアライズする。
- エラー: JSON デコード失敗や未知コマンドは WebSocket 同様 `Invalid { error: String }` を返却。`handle_command` が `None` を返すケースは返信なし。
- 互換性: 既存 JSON 形を変えない。プロトコルバージョンは当面導入せず、将来必要になれば envelope フィールドを追加する方針。

## Follow-up tasks（他 issue への入力）
- Control プロトコル共通化（`WsCommand`/`WsReply`/`handle_command` を独立モジュール化）[#3]
- ZMQ サーバ実装（ROUTER bind + DEALER クライアント前提のループ）[#4]
- CLI: `--zmq-endpoint` 追加（`zmq-server` feature 時のみ表示）[#5]
- Docs: `zmq-server` の有効化手順・サンプルクライアント（pyzmq/Rust）[#6]
- Tests/CI: `cargo build --no-default-features` / `--features zmq-server`、`libzmq` インストール手順、最小リクエスト/レスポンステスト[#7]

