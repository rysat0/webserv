# 03 設定・ルーティング・静的配信・アップロード・CGI — 理解すべきこと

一次情報: nginx ドキュメント(`ngx_http_core_module` の server/location/root/index/autoindex/error_page/client_max_body_size/limit_except/return)、RFC 3875(CGI)、RFC 7578(multipart/form-data)、RFC 3986、RFC 6265(ボーナス)。

---

## C1 設定ファイル文法 【B, R:M, D:L, ★★★】

**理解すべきこと**
- nginx の設定文法: `directive arg1 arg2;` と `block_name args { ... }`、`#` コメント、クォート(`"..."`)、空白/改行の扱い、`;` 必須。
- 字句解析(トークン種別: WORD, `{`, `}`, `;`)と構文解析の分離(決定: 1クラス、private関数で分離)。
- 文法エラー時の動作は「警告して終了(行番号なし)」(決定済み)。**どのエラーを区別するか**(未閉じブロック、未知ディレクティブ、引数個数不正、数値不正)をリスト化。
- 数値/サイズの単位表現(`10M`, `1k` など)を採用するか(`client_max_body_size`)。オーバーフロー検出。
- 設定ファイルのデフォルトパス `./conf/default.conf`(決定)、引数>1個や読めないファイルのエラー。

**資料化する内容**: 文法のBNF風定義、エラー種別一覧、**サンプル設定ファイル(評価デモ用に複数必要)**の設計方針。

## C2 ディレクティブ一覧と検証規則 【B(主)/共, R:M, D:L, ★★★】

`Requirements.md` の**未決定事項**そのもの。課題§7の10項目をディレクティブに落とす。決定事項:「継承なし・デフォルトなし・必須が欠けたら起動拒否」「NGINX風+独自名の混在」。

**表に起こす内容(各行: 名前 / 階層 / 型 / 必須か / 検証 / nginx対応名)**

| 課題要件 | 階層 | 叩き台名 | 備考 |
|---|---|---|---|
| listen interface:port | server | `listen` | 複数可。ポート範囲0–65535、重複の扱い(N2) |
| デフォルトエラーページ | server | `error_page` | `error_page 404 /path;`(複数コード可?)。ファイル存在検証をいつ行うか |
| ボディ最大サイズ | server | `client_max_body_size` | 継承なし方針なので locationでも指定必須か決める |
| 許可メソッド | location | `allowed_methods` / nginx の `limit_except` | 空リストの意味、未知メソッドの検証 |
| リダイレクト | location | `return` or `redirect` | コード指定の有無(H11) |
| ルート対応付け | location | `root`(nginx の `root` は location 部分を**パスに連結**、`alias` は置換) | **課題の例(`/kapouet`→`/tmp/www`、`/kapouet/pouic/toto` → `/tmp/www/pouic/toto`)は `alias` 型の挙動**。nginxの`root`と意味が違うため名前と仕様を慎重に決める(要注意) |
| autoindex | location | `autoindex on/off` | |
| デフォルトファイル | location | `index` | 複数候補? 順に探索? |
| アップロード許可/保存先 | location | `upload_path` 等 | 空=不可(Requirements.mdの案)。保存先の存在・書込権限検証 |
| CGI | location | `cgi_extension .py /usr/bin/python3`(決定済み) | 複数拡張子、インタプリタの存在/実行権限検証 |
| (任意)server_name | server | `server_name` | 仮想ホストはスコープ外。有無を決める |

**検証(validation)の論点**: 必須項目の定義(特に「継承なし」で全locationに書く項目を何にするか)、`location` パスが `/` 始まりか、重複 location、`root` ディレクトリ存在、`return` と他ディレクティブの併記禁止、ポート重複。

## C3 location マッチングとパス変換 【B, R:M, D:M, ★★★】

- **最長前方一致**(決定): 文字列としての前方一致か、**パス境界**(`/api` が `/apixyz` にマッチしない)を考慮するか → 仕様化が必要(nginxは文字列前方一致、境界は考慮しない。選択を決めて文書化)。
- `/` のみの location をフォールバックに。マッチなし=NULL→404(Requirements.md)。
- `root` による URL→ファイルパス変換(C2の `root`/`alias` 問題)。結合後の正規化と、**root外に出ていないことの再検証**(H2のトラバーサル対策との二重防御)。
- クエリ文字列の除去、パーセントデコード済みパスで照合(H2)。
- ディレクトリ要求のとき: 末尾`/`なし → 301補完か、`index` 探索、`autoindex`、いずれも不可で 403。この**判断フロー図**がRouterの中核仕様。

## C4 静的ファイル配信 【B, R:M, D:M, ★★★】

- `stat`/`access` による存在・種別(通常ファイル/ディレクトリ)・読み取り権限の判定と、**結果→ステータス対応**(なし=404、権限なし=403、ディレクトリ→C3)。
- `Content-Type`(H10)、`Content-Length`、`Last-Modified`(任意)。大きなファイルの扱い(N4/N10)。
- 条件付きリクエスト(`If-Modified-Since`、304)、Range(206)は**課題必須ではない**。スコープ外として明文化するかを決める(ブラウザ動画再生などでRangeが使われる点に注意)。
- シンボリックリンクの扱い(許可/拒否)、隠しファイル。

## C5 autoindex 【B, R:S, D:S, ★★】

- `opendir/readdir/closedir`、`stat` で種別・サイズ・更新時刻。`.` と `..` の扱い、ソート順(ディレクトリ先頭など)。
- **HTMLエスケープ**(ファイル名に `<>&"` が含まれる場合のXSS対策)と、URLエンコード(リンクのhref)。
- 結果の生成方法(HTML文字列組み立て)。巨大ディレクトリ時のサイズ上限。

## C6 アップロード 【B, R:L, D:L, ★★★】

**理解すべきこと**
- アップロード方式の選択肢: (a) `multipart/form-data`(ブラウザのフォーム標準)、(b) 生のボディ(`curl --data-binary`、`PUT`的な)。**課題は「クライアントがファイルをアップロードできる」とのみ規定**→評価でブラウザ/curlどちらを使うか不明のため、両方の対応範囲を決める(推奨: multipart対応+生ボディ対応の可否を判断)。
- **multipart/form-data の構造**(RFC 7578 / RFC 2046 §5.1): `Content-Type: multipart/form-data; boundary=XXXX`、各パートは `--boundary CRLF part-headers CRLF CRLF data CRLF`、終端 `--boundary--`。パートヘッダー `Content-Disposition: form-data; name="..."; filename="..."`、`Content-Type`。
  - boundary のクォート有無、境界文字列の検索がバイナリ安全(`\0`を含むデータ)であること(`std::string`で可だが `c_str()` 依存を避ける)。
  - 複数ファイル/ファイル以外のフィールドの扱い。
- 保存先と**ファイル名の扱い**: クライアント指定の `filename` を信用しない(`../` 等のトラバーサル、特殊文字、既存ファイル上書き、拡張子による実行(CGI拡張子のアップロード=リモートコード実行))。サニタイズ規則・衝突時の命名(連番/タイムスタンプ)を決める。
- ボディ全体をメモリに保持するか(`HttpRequest::getBody()` は std::string 返却=全メモリ保持の設計)。上限は `client_max_body_size` で制御。大きいファイルでのメモリ・性能の許容範囲を明文化。
- 応答: 201 + `Location`(保存先URL)、または 200 + 結果HTML。失敗時: 413(上限超過)、400(multipart不正)、403/500(書込失敗)、415(未対応のメディア型)。
- 保存先ディレクトリの存在/権限の起動時検証(C2)。

## C7 DELETE 【B, R:S, D:S, ★★】

- 対象解決は静的配信と同じ(C3)。ファイルのみ許可か、ディレクトリ削除を許すか(許す場合 `rmdir` 相当は許可関数に**無い**: `unlink`/`remove`/`rmdir` は許可リストに載っていない → **DELETEの実装手段を確認**。許可関数に削除系がない点は要確認事項)。
- 結果: 成功 204(または200)、404、403、405。
- 削除対象を限定するか(例: アップロード先ディレクトリ配下のみ)—セキュリティ上、全ファイル削除可能は危険なので `allowed_methods` と location 設計で制御する旨を明記。

> **確認事項**: 許可関数リストに `unlink`/`remove` が含まれていない。DELETEでファイルを実際に削除する方法を課題/運営に確認する(`Requirements.md` を再確認)。解決しないとC7の仕様が書けない。

## C8 CGIプロセス制御(システムコール面) 【A(主)/共, R:L, D:L, ★★★】

**理解すべきこと**
- `pipe` ×2(親→子のstdin用、子→親のstdout用)→ `fork` → 子: `dup2` で stdin/stdout へ付け替え、**不要な fd を close**、`chdir`(CGIスクリプトのディレクトリへ。課題要件)、`execve(interpreter, argv, envp)` → 失敗時 `_exit`(※`exit` でなく。`_exit` は許可リスト外だが、`exit` との差=親のバッファ/デストラクタ実行の問題。**使用可否を確認**)。
- 親: 使わない端の close、**パイプfdをノンブロッキング化し poll に登録**(CGI出力の読み取り、ボディの書き込み。課題の「pipeは必ずpoll経由」要件)。
- **fork は CGI 実行にのみ使用可**。ここ以外で `fork` しない。
- `fork` 後に子で C++ オブジェクトが多数存在する状態 → 子は `execve` 直前までに**複雑な処理をしない**(メモリ確保など)、`envp` は fork 前に構築しておく(`char**` の組み立て・解放責任の明確化)。
- 子プロセス回収: `waitpid`(`WNOHANG`)、ゾンビ防止。タイムアウト時の `kill(pid, SIGKILL)` → `waitpid`。
- `execve` 失敗の検知方法(子がexec失敗で `_exit(1)` した場合、親は「出力が空+終了コード非0」で判断 → 502/500)。
- インタプリタ/スクリプトの検証: ファイル存在、実行権限(`access(X_OK)`)。
- fd継承: 他クライアントのソケットfdが子に漏れる問題(`FD_CLOEXEC` を使うか、子で全fdをclose)。

## C9 CGI仕様(RFC 3875) 【B(主)/共, R:L, D:L, ★★★】

**理解すべきこと**
- **メタ変数**(RFC 3875 §4.1)。決定事項に挙がった `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR, SERVER_SOFTWARE` + `HTTP_*`。それぞれの**値の作り方**:
  - `SCRIPT_NAME`/`PATH_INFO` の分離(`/cgi/test.py/extra/path` → script=`/cgi/test.py`、PATH_INFO=`/extra/path`)。この分割ロジックの仕様が必要。
  - `PATH_TRANSLATED`、`SCRIPT_FILENAME`(PHP-CGIでは `SCRIPT_FILENAME` と `REDIRECT_STATUS` が必須なことが多い。**Python第一**でも将来のPHP対応に備え記録)。
  - `CONTENT_LENGTH`/`CONTENT_TYPE` はボディがある場合のみ。`QUERY_STRING` は未デコードのまま。
  - `HTTP_*` 変換: ヘッダー名を大文字化、`-`→`_`、接頭辞`HTTP_`。`Content-Type`/`Content-Length` は `HTTP_` を付けない。**`Proxy` ヘッダー(`HTTP_PROXY`)による httpoxy 脆弱性**対策として `Proxy` ヘッダーを渡さない選択。
- **リクエストボディの渡し方**: un-chunk 済みのボディを子のstdinへ全て書き、書き終えたらstdinを **close(EOF)**(課題要件)。ボディが大きいと pipe バッファ(64KB程度)を超える → **書き込みもノンブロッキング+POLLOUT**(死活: 子が出力を溜めて読まないとデッドロックし得る → 読み/書きを同時にpoll)。
- **CGI出力のパース**(§6): `CGI-response = document-response / local-redir-response / client-redir-response / client-redirdoc-response`。ヘッダー部(`Content-Type`, `Status`, `Location`, 任意ヘッダー)+空行+ボディ。
  - `Status: 404 Not Found` → HTTPのステータス行へ変換。なければ200。
  - `Location:` のみ(ローカル絶対パス `/..`)→ サーバー内部リダイレクト(スコープ内か外か決める)。絶対URI → 302。
  - `Content-Type` がない場合のエラー扱い(502)。
  - **ヘッダー区切りが LF/CRLF 両方**あり得る(Pythonの`print`はLF)。
  - **`Content-Length` が無ければ EOF を終端とする**(課題要件)。サーバー側は終端後に自分で `Content-Length` を付けて返す(接続は `Connection: close`)。
- 終了ステータス(非0)、stderr の扱い(捨てる/ログに出す。stderrをpollするか、`/dev/null`へか)。

## C10 CGIとイベントループの統合 【共, R:M, D:M, ★★★】

- 状態: `Client::WAITING_CGI`、`CgiProcess` が親Clientを指し、EventLoop が **fd→オブジェクトの対応表**で引く(決定済み)。
- 複数のfd(CGIのstdin書込/stdout読取)を毎周のpollfd再構築で扱う方法。
- **失敗パターンの網羅**: execve失敗、CGIがすぐ終了、出力なし、出力途中でクライアント切断、タイムアウト(10秒→kill→504)、巨大出力(上限)、子が書込を詰まらせる、複数同時実行(プロセス数上限)。
- CGIの結果を `HttpResponse` に変換する責務の置き場所(Requirements.md の注記「Router::handle の戻り値を拡張予定」の具体化)→ **非同期結果をRouterに返す設計**(Routerは同期関数のままCGI起動だけ要求し、完了時にEventLoopが応答を組み立てる等)。要設計。

## C11 Cookie/セッション(ボーナス) 【B, R:M, D:M, ★】

- `Set-Cookie`/`Cookie` の形式(RFC 6265)、属性(`Path`, `Expires`/`Max-Age`, `HttpOnly`, `SameSite`)。セッションIDの生成(乱数源: 許可関数に `rand` 系があるか要確認)、サーバー側セッション保管(メモリ上のmap)。
- ボーナスは**必須部分が完璧な場合のみ評価**されるため、優先度は最低。複数CGIタイプ対応(PHP等)も同様。

---

## この章の作業量サマリ

| 区分 | 量 | コメント |
|---|---|---|
| C1–C3(設定・ルーティング) | L(D大) | 調査は軽いが**決定事項が多く、資料化に時間がかかる**(ディレクティブ表、判断フロー図) |
| C4–C5, C7 | S〜M | 短時間。ただしC7は許可関数の確認待ち |
| C6(アップロード) | L | RFC読み込み+設計判断(方式・サニタイズ・メモリ) |
| C8–C10(CGI) | XL | 共同。A=C8、B=C9を主に調査し、C10の結合設計を合同で行う |
| C11 | M | 後回し可 |
