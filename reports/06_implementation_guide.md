# 実装の始め方

HTTP担当の最初の目標は、受信データを何回かに分けて渡しても、`HttpRequest`が1件のGETを正しく組み立てられることです。この段階では文字列を入力に使えるため、ネットワーク側の完成を待たずに確認できます。

## 1. リクエストの解析を通す

最初は本文のない `GET / HTTP/1.1` を使います。次の順に `appendData()` へ渡し、各呼び出しの直後に状態を確認します。表の `\r\n` は、実際の入力ではCRとLFの2バイトです。

| 渡すデータ | 期待する状態 |
|---|---|
| `"GET / HTTP/1.1\r"` | `INCOMPLETE` |
| `"\nHost: localhost\r\n"` | `INCOMPLETE` |
| `"\r\n"` | `COMPLETE`。メソッドは `GET`、パスは `/` |

次に、HTTP/1.1での `Host` の欠落と重複を試し、400が返せる状態になることを確認します。その後、`Content-Length`付きの本文を断片で渡し、指定した長さに達したときだけ完了することと、上限超過を拒否することを確認します。最後にchunkedの復号と異常入力を検証します。[HttpRequestの内部設計](../Requirements.md#http-request-internals)、[本文とchunkedの規則](../Requirements.md#http-request-body)

## 2. 応答を組み立てる

`HttpResponse`で、本文付きの200と、400のエラー応答を作れるようにします。HTTP/1.0のステータス行、ヘッダーを閉じる空行、確定した本文の実バイト数に一致する `Content-Length` を確認します。カスタムのエラーページがある場合の選択と、読み取り結果を受け取って本文を確定する処理もここで実装します。ファイルの読み取りはClientが所有するタスクが担当します。[HTTP応答生成の規則](../Requirements.md#http-response)、[エラーページの準備](../Requirements.md#error-page-prepare)

送信側は `serializeHeaders()` で作ったヘッダー部を先に送り、続いて `getBody()` で参照した本文を送ります。本文を送らない応答の判定は `HttpResponse` が担当します。[送信用API](../Requirements.md#response-send-accessors)

## 3. ルーティングを実装し、ソケットにつなぐ

`Router`は、解析済みの要求と設定を受け取り、応答、ファイル読み取り、一覧生成、アップロード保存、CGIのどれに進むかを決めます。設定ファイルの解析は `ConfigParser` が担当します。Routerの単体試験では、準備済みのConfigと固定の `ConnectionInfo` を入力として使えます。[Configの型とAPI](05_config_agreement.md#config-api)、[RouterとClientの共有API](03_config_cgi_upload.md#cgi-api)

最初の結合では、小さな静的ファイルへのGETを1件通します。Clientが部分受信を `HttpRequest` に渡し、Routerの指示でファイルを読み取り、ヘッダーと本文を送れることを確認します。続いてエラーページ、一覧、アップロード、CGIをつなぎ、並行接続や切断、期限超過でも他の接続が進むことを確かめます。[結合試験と負荷試験](04_testing_and_workplan.md)

ネットワーク側では、`accept`直後に `getsockname` が失敗した接続のfdだけを一度閉じます。取得に成功した接続は、有効な `ConnectionInfo` を持ってHTTP側へ進みます。[接続先情報の取得失敗時の契約](../Requirements.md#getsockname-failure)
