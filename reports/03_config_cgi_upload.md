# 03 設定・ルーティング・静的配信・アップロード・CGI — 理解すべきこと

一次情報: nginx ドキュメント(`ngx_http_core_module` の server/location/root/index/autoindex/error_page/client_max_body_size/limit_except/return)、RFC 3875(CGI)、RFC 7578(multipart/form-data)、RFC 3986、RFC 6265(ボーナス)。

CGIの最新の合意状況は、末尾の「C12 CGIの最小構成・決定事項と残る合意」を参照。C8〜C10には調査項目も含まれ、列挙された機能すべてを実装するという意味ではない。

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
- インタプリタ/スクリプトの検証: インタプリタは存在・実行権限、Pythonへ引数として渡すスクリプトは通常ファイルであること・読み取り権限を確認する。スクリプト自身の実行権限は要求しない。
- fd継承: 他クライアントのソケットfdが子に漏れる問題(`FD_CLOEXEC` を使うか、子で全fdをclose)。

## C9 CGI仕様(RFC 3875) 【B(主)/共, R:L, D:L, ★★★】

**理解すべきこと**
- **メタ変数**(RFC 3875 §4.1)。決定事項に挙がった `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR, SERVER_SOFTWARE` + `HTTP_*`。それぞれの**値の作り方**:
  - `SCRIPT_NAME`/`PATH_INFO` の分離(`/cgi/test.py/extra/path` → script=`/cgi/test.py`、PATH_INFO=`/extra/path`)。この分割ロジックの仕様が必要。
  - 必須部分はPythonのみとし、PHP固有の環境変数は追加しない(今回確定、C12)。`PATH_TRANSLATED` の採否は `PATH_INFO` の対応範囲と合わせて確定する。
  - `CONTENT_LENGTH` は復号後のボディ長、`CONTENT_TYPE` は要求ヘッダーから決める。ボディなし・ヘッダーなしの場合の空値/省略規則はC12で確定する。`QUERY_STRING` は未デコードのまま。
  - `HTTP_*` 変換: ヘッダー名を大文字化、`-`→`_`、接頭辞`HTTP_`。`Content-Type`/`Content-Length` は `HTTP_` を付けない。**`Proxy` ヘッダー(`HTTP_PROXY`)による httpoxy 脆弱性**対策として `Proxy` ヘッダーを渡さない選択。
- **リクエストボディの渡し方**: un-chunk 済みのボディを子のstdinへ全て書き、書き終えたらstdinを **close(EOF)**(課題要件)。ボディが大きいと pipe バッファ(64KB程度)を超える → **書き込みもノンブロッキング+POLLOUT**(死活: 子が出力を溜めて読まないとデッドロックし得る → 読み/書きを同時にpoll)。
- **CGI出力のパース**(§6): `CGI-response = document-response / local-redir-response / client-redir-response / client-redirdoc-response`。ヘッダー部(`Content-Type`, `Status`, `Location`, 任意ヘッダー)+空行+ボディ。
  - `Status: 404 Not Found` → HTTPのステータス行へ変換。通常の文書応答で省略された場合は200。リダイレクトは別の規則で扱う。
  - `Location:` のみ(ローカル絶対パス `/..`)→ サーバー内部リダイレクト(スコープ内か外か決める)。絶対URI → 302。
  - 通常の文書応答で `Content-Type` がない場合のエラー扱い(502案)。Locationのみのリダイレクトとは区別する。
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

## C12 CGIの最小構成・決定事項と残る合意 【共】

記録日: 2026-10-04。課題PDF Version 24.1と参考資料を踏まえ、必須部分の実装量を抑えるための合意状況を記録する。

**今回承認されたのは、次の7項目。後続の仕様案・数値案・分担の詳細は未確定であり、二人で合意してから決定事項へ移す。** 今回の記録先は本書のみ。`Requirements.md` の「Python第一」等は未更新のため、今回確定した7項目については本節を最新の記録として扱う。

### 決めたこと：今回確定した7項目

| ID | 項目 | 決定 |
|---|---|---|
| CGI-01 | 対応言語 | 必須部分ではPythonのみ。PHP固有の対応は作らない |
| CGI-02 | 起動方法 | 設定したPythonの絶対パスを `execve` で実行する。シェルやshebangの解釈は実装しない |
| CGI-03 | 入力 | リクエストを受信・復号してから起動する。クエリは環境変数、ボディは加工せずstdinへ渡す |
| CGI-04 | 出力 | stdoutを上限付きで蓄積し、EOFと子の終了を確認してからHTTP応答を作る。ブラウザへの逐次転送はしない |
| CGI-05 | CGIの種類 | 通常のCGIだけを対象とする。HTTPレスポンス全文を直接出すNPH、FastCGI、常駐プロセスは対象外 |
| CGI-06 | 周辺機能 | 認証、セッション、逆引きDNS、PHP専用環境変数は追加しない。C11は必須部分の実装対象に含めない |
| CGI-07 | Config | 既存の `cgi_extension` を使う。タイムアウトや出力上限の設定項目は増やさず、まずコード内定数にする |

CGI-03の「加工せず」は、HTTPのchunked復号後のボディをそのまま渡すという意味。CGIへの受け渡しのためにフォームの `name=value` 分解、URLデコード、multipartの分解をWebserv側で行わない。通常のアップロード機能(C6)の形式・保存処理は別途決める。

CGI-01は対応・検証する言語の範囲を決めたもの。Configで受理する拡張子を `.py` に限定するか、拡張子とインタプリタの組を何個許可するかまで承認したものではない。[Config合意案](05_config_agreement.md)の未確定事項と合わせて決める。

### 維持する課題要件・既存の決定

- 課題要件: 拡張子によるCGI実行、リクエスト情報・引数の受け渡し、chunkedの復号、入力/出力のEOFの扱い、相対ファイルにアクセスできる作業ディレクトリ、最低1種類のCGIを満たす。
- 課題要件: 親サーバーはノンブロッキングで動かし、待ちが発生するパイプI/Oも同じpoll等で管理する。入力書き込みと出力読み取りを並行して進め、部分読み書き、切断、子プロセスの後始末を省かない。
- 既決定: CGIは10秒でkill + 504。起点や延長の有無は下表で確定する。
- 既決定: `SERVER_PROTOCOL` は受信した要求のHTTPバージョン。chunkedの場合の `CONTENT_LENGTH` は復号後のバイト数。入力を全て書いたら親側のstdin用パイプを閉じる。
- 既決定: HTTP/1.0で応答し、1接続1要求、`Connection: close`。本文とContent-Lengthの可否は204等のステータスに応じて既存のHTTP規則を使う。
- Configの既決定: `root` / `alias` の意味と、設定中の相対パスを設定ファイルのディレクトリ基準で解決する規則は維持する。CGI実行時の作業ディレクトリとは区別する。

### 確定すべきこと：外部に見える仕様案

以下はすべて**未合意の推奨案**。採用する項目にチェックし、変更した場合は具体的な動作も書き換える。

| 合意 | 論点 | 推奨案・確定する内容 |
|---|---|---|
| [ ] | 起動条件・メソッド | 選択されたlocationで設定した拡張子のスクリプトを実行する。CGIはGET/POSTを許可し、DELETEは通常locationで提供する。indexとして選ばれた `index.py` にも同じ実行判定を適用する |
| [ ] | PATH_INFO | `/cgi/test.py/extra?x=1` を `SCRIPT_NAME=/cgi/test.py`、`PATH_INFO=/extra`、`QUERY_STRING=x=1` に分ける基本形に対応する。スクリプト部分の特定方法とURI正規化後の扱いを固定する |
| [ ] | アップロードとの分離 | アップロード保存とCGI実行のlocation・保存先を分ける。CGI自身によるボディ処理は `upload_store` と区別する |
| [ ] | 実行パス・cwd | Bがroot/aliasからスクリプトの絶対パスとそのディレクトリを解決する。Aは子プロセスだけで `chdir` し、Pythonとスクリプトの絶対パスを使って実行する |
| [ ] | 環境変数 | 既存の変数一覧について、値の出所と空値/省略の規則を表にする。特にボディなしの `CONTENT_LENGTH`、未指定の `CONTENT_TYPE`、追加パスなしの `PATH_INFO`、Hostなしの `SERVER_NAME` を確定する |
| [ ] | HTTPヘッダーの転送 | `HTTP_*` へ変換する例外規則を定める。Content-Type/Content-Lengthは専用変数を使い、接続制御用のヘッダーは除外する。同名ヘッダーの扱いもHTTPパーサーと揃える |
| [ ] | コマンドライン引数 | `?a+b` をargvへ展開する形式は対象外とする。クエリは常に `QUERY_STRING` に渡し、argvはPythonとスクリプトの実行に必要な引数だけにする |
| [ ] | 文書応答の解析 | ヘッダーと空行と本文を分離し、LF/CRLFの両方を受理する。Content-Typeを要求し、Status省略時は200。StatusはHTTPステータス行へ変換する |
| [ ] | リダイレクトの範囲 | 絶対URLのLocationのみなら302。LocationにStatus・Content-Type・本文を伴う応答の扱いも固定する。ローカルLocationのみの内部リダイレクトは対象外とし、未対応の出力として502にする |
| [ ] | 応答ヘッダー・長さ | EOFまで蓄積した本文の実長から最終Content-Lengthを生成する。CGIが長さを指定した場合は照合し、不一致は502。CGIのStatusは転送せず、Connection等の接続制御はサーバーが決める。他のヘッダーの転送規則も固定する |
| [ ] | 終了条件・stderr | 正常完了はstdoutのEOFと子の正常終了の両方。異常終了は502とする。stderrはstdoutへ混ぜずサーバーのstderrを継承し、追加の監視パイプは作らない |
| [ ] | タイムアウトの定義 | 起動から10秒の経過時間とし、出力が続いても延長しない。計測起点をfork成功時などに統一し、EOF後も子が終了しなければ対象とする |

RFC 3875に定義された機能の一部を対象外にするため、採用後はRequirementsの「RFC 3875準拠」を「RFC 3875を参考に、対応範囲を限定」と具体化する案。課題PDFに個別の対応範囲が明記されていない機能を、課題から免除されたと断定しない。

### 決めること：分担・受け渡し・数値

大枠の分担はA=ネットワーク/イベントループ、B=HTTP/Config、CGIの結合は共同。下表は、その境界を具体化する**未合意の案**。

| 担当案 | 責任 |
|---|---|
| A | pipe/fork/execve、パイプI/O、pid・fd・実行状態の所有、時間管理、子の終了・回収、切断時の後始末 |
| B | 実行対象・パスの解決、環境変数の内容、CGI出力の解析、HTTP応答の生成 |
| 共同 | 以下の受け渡し、エラー対応、上限値、結合テスト |

- [ ] **APIと戻り値**: B→Aは実行パス・argv・環境変数・cwd・入力ボディ、A→Bは標準出力・終了理由・終了コードとする案。Routerの「即時応答/CGI実行要求」を区別する戻り値を先に確定する。RequirementsのB側 `CgiExecutor` に起動処理も含まれている点を整理する。
- [ ] **データの寿命**: 実行中の `CgiProcess` が必要な入力を所有する案。コピー/参照の使い分け、envpの確保・解放担当、Clientとの関連付けを決める。
- [ ] **切断・終了時の処理**: Aが監視解除・fd close・必要なkill・非ブロッキングのwaitpid回収を行う案。削除済みClientへアクセスせず、同じfdを二重に閉じない。CGIの子へ不要なソケット・パイプを引き継がない方法も確定する。
- [ ] **子の起動失敗と時間計測**: chdir/dup2/execve失敗時の子の終了方法と親の検出方法を決める。C8にある `_exit` の許可リスト上の扱い、および時間取得方法は、課題の許可関数と照合する。失敗した子が親のイベントループへ戻らないことを確認する。
- [ ] **エラー対応表**: 次の案を採用するか確定する。既存の「起動失敗は一律502」と差があるため、採用時にRequirementsのAPI案も更新する。

| 状況 | 未確定の応答案 |
|---|---|
| スクリプトなし / 読み取り不可 | 404 / 403 |
| サーバー側のpipe・fork失敗 | 500 |
| exec失敗・子の異常終了・不正/未対応のCGI出力 | 502 |
| CGIの出力上限超過 | 子を終了・回収して502 |
| 10秒超過 | 子を終了・回収して504(504は既決定) |
| 同時実行数の上限到達 | 503 |

- [ ] **上限値**: 下表は初期案であり、課題指定値でも今回の承認事項でもない。入力ボディ上限は既存の `client_max_body_size` を使う。

| 定数 | 未確定の初期案 |
|---|---|
| CGIヘッダー上限 | 8 KiB |
| CGI出力全体の上限(ヘッダーを含むstdout) | 8 MiB |
| CGI同時実行数の上限(サーバープロセス全体) | 16本 |

- [ ] **共通の結合テスト**: GETのクエリ、POST/HTTP1.1 chunkedのボディ、cwdからの相対ファイル読み込み、パイプ容量を超える入出力、出力形式の異常、即時終了、タイムアウト、クライアント切断、各上限への到達を確認する。PATH_INFO等は合意した対応範囲に合わせて追加する。CGI実行中も静的配信でき、終了後にfdや未回収の子が残らないことを完了条件にする。

### 今回参照した資料

- [課題PDF Version 24.1](../webserv.pdf): 印刷ページ8のI/O・切断処理、10〜11のConfig/CGI要件。複数CGIタイプはボーナス。
- [RFC 3875 原文](https://www.rfc-editor.org/rfc/rfc3875.html) / [日本語訳](https://tex2e.github.io/rfc-translater/html/rfc3875.html): §4の入力・環境変数、§5のNPH、§6の出力。仕様上の判断は原文を基準にする。
- [筑波大学 2016-06-15](https://www.coins.tsukuba.ac.jp/~syspro/2016/2016-06-15/index.html): CGI側でクエリ・POST入力を処理する例。
- [筑波大学 2022-07-20](https://www.coins.tsukuba.ac.jp/~syspro/2022/2022-07-20/index.html): 最小のCGI出力、環境変数、子プロセス回収の説明。
- [筑波大学 2022-07-27](https://www.coins.tsukuba.ac.jp/~syspro/2022/2022-07-27/index.html): CGIプログラム側でのフォーム処理例。
- [とほほのCGI仕様](https://www.tohoho-web.com/wwwcgi3.htm): CGIの入出力の概観。掲載された全機能を今回の必須範囲とするわけではない。

---

## この章の作業量サマリ

| 区分 | 量 | コメント |
|---|---|---|
| C1–C3(設定・ルーティング) | L(D大) | 調査は軽いが**決定事項が多く、資料化に時間がかかる**(ディレクティブ表、判断フロー図) |
| C4–C5, C7 | S〜M | 短時間。ただしC7は許可関数の確認待ち |
| C6(アップロード) | L | RFC読み込み+設計判断(方式・サニタイズ・メモリ) |
| C8–C10(CGI) | XL | 共同。A=C8、B=C9を主に調査し、C10の結合設計を合同で行う |
| C11 | M | 後回し可 |
