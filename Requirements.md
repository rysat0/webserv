# webserv 要件・決定事項一覧

> 課題PDF (Version 24.1) の全要求事項と、チームの設計決定をまとめた内部ドキュメント。
> ※提出用 README.md(英語・別途作成)とは別物。

---

## 1. プロジェクト概要

- **C++98でHTTPサーバーを自作する**
- 実行形式: `./webserv [configuration file]`
- 設定ファイルは引数で渡されるか、デフォルトパスから読み込めること
- 実際のWebブラウザでテスト可能であること
- HTTP/1.0 が参考基準(強制ではない)。RFC全文の実装は要求されていない。RFCを読むこと、telnet と NGINX で事前にテストすることが推奨されている
- チーム方針: **HTTP/1.0で応答し、HTTP/1.0・HTTP/1.1のリクエストを課題に必要な範囲で処理する**。詳細は「HTTP対応範囲」を参照

## 2. 全体ルール(違反 = 0点级)

- [ ] **いかなる状況でもクラッシュ禁止**(メモリ不足でも)。予期しない終了も禁止 → 違反で0点
- [ ] Makefile 必須: `$(NAME), all, clean, fclean, re` のルールを含む
- [ ] Makefile は不要な再リンクをしない
- [ ] コンパイラは `c++`、フラグ `-Wall -Wextra -Werror`
- [ ] **C++98 準拠**。`-std=c++98` を付けてもコンパイルが通ること
- [ ] C++機能を優先的に使う(例: `<string.h>` より `<cstring>`)。C関数は使用可だがC++版があればそちらを優先
- [ ] 外部ライブラリ・Boost は一切禁止

## 3. 提出物

- `Makefile`
- `*.h / *.hpp / *.cpp / *.tpp / *.ipp`
- **設定ファイル**(評価で全機能をデモできるもの)
- **デフォルトファイル群**(テスト用のHTML等)
- `README.md`(リポジトリ直下、英語、詳細は §9)
- プログラム名: `webserv`

## 4. 使用可能な外部関数(これ以外は使用不可)

```
execve, pipe, strerror, gai_strerror, errno, dup, dup2, fork,
socketpair, htons, htonl, ntohs, ntohl,
select, poll, epoll (epoll_create, epoll_ctl, epoll_wait),
kqueue (kqueue, kevent),
socket, accept, listen, send, recv, shutdown, chdir, bind, connect,
getaddrinfo, freeaddrinfo, setsockopt, getsockname, getprotobyname,
fcntl, close, read, write, waitpid, kill, signal,
access, stat, open, opendir, readdir, closedir
```

- libft: 使用不可 (n/a)
- poll() は select() / kqueue() / epoll() など同等関数で代替可

## 5. I/O 必須要件(最重要・違反で0点)

- [ ] **サーバーは常にノンブロッキング**で動作すること
- [ ] クライアント切断を適切に処理すること
- [ ] クライアント⇔サーバー間の**全I/O(listen含む)を1つの poll()(または同等)** で管理すること
- [ ] poll()(または同等)は**読み・書きを同時に監視**すること
- [ ] **poll() を通さずに read / write をしてはならない**
- [ ] ソケット・パイプ/FIFO 等「待ちが発生しうるfd」への readiness 確認なしの read/recv/write/send → **0点**
- [ ] read/write 実行後に **errno の値を見て挙動を変えるのは厳禁**
- [ ] 通常のディスクファイルは例外: poll 不要、read()/write() 直接可
- [ ] poll 等に付随するマクロ・ヘルパー関数(FD_SET 等)は使用可
- [ ] **fork は CGI 実行以外に使用禁止**
- [ ] 他のWebサーバーを execve するのは禁止

## 6. 機能要件

- [ ] **GET / POST / DELETE** メソッド(最低限この3つ)
- [ ] **完全な静的Webサイトを配信**できること
- [ ] クライアントが**ファイルをアップロード**できること
- [ ] HTTPレスポンスの**ステータスコードが正確**であること
- [ ] **デフォルトエラーページ**を持つこと(設定で未指定の場合に使う)
- [ ] リクエストが**永久にハングしない**こと(タイムアウト実装)
- [ ] 標準的な**Webブラウザで動作互換**であること
- [ ] **複数ポートで listen** し、ポートごとに異なるコンテンツを配信できること
- [ ] **ストレステストに耐え、常時稼働**し続けること(Resilience is key)
- [ ] NGINX と挙動・ヘッダーを比較して検証する(HTTPバージョン差異に注意)
- [ ] 仮想ホスト(virtual host)機能はスコープ外(実装は任意)
- [ ] テストは複数のプログラムで行う(Python / Golang / C / C++ 等でテストを書く)
- [ ] 公式の小型テスターが提供されている(使用は任意、デバッグ補助)

## 7. 設定ファイル要件(NGINX の server セクション参考)

### サーバー単位で設定できること
- [ ] 1. listen する interface:port の組(複数定義 = 複数サイト)
- [ ] 2. デフォルトエラーページの指定
- [ ] 3. クライアントリクエストボディの最大サイズ

### ルート(URL/route)単位で設定できること(regex 不要)
- [ ] 4. 許可する HTTP メソッドのリスト
- [ ] 5. HTTP リダイレクト
- [ ] 6. ルートディレクトリの対応付け(例: `/kapouet` → `/tmp/www` のとき `/kapouet/pouic/toto/pouet` は `/tmp/www/pouic/toto/pouet` を探す)
- [ ] 7. ディレクトリリスティング(autoindex)の有効/無効
- [ ] 8. ディレクトリが要求されたときのデフォルトファイル
- [ ] 9. アップロードの許可と保存先の指定
- [ ] 10. 拡張子ベースの CGI 実行(例: `.php`)

### その他
- 独自の設定項目を追加してよい(例: server_name)
- 評価時に全機能をデモできる設定ファイルを複数提出すること

## 8. CGI 要件

- [ ] 最低1種類の CGI をサポート(php-CGI, Python など)
- [ ] Webサーバー⇔CGI 間の**環境変数**を正しく設定(リクエスト全文と引数を CGI から参照可能にする)
- [ ] **chunked リクエストはサーバー側で un-chunk** して CGI に渡す(CGI は EOF をボディ終端として期待する)
- [ ] CGI 出力も同様: `content_length` が無ければ **EOF が出力終端**
- [ ] CGI は**正しいカレントディレクトリで実行**(相対パスでのファイルアクセスのため → chdir)
- [ ] fork / execve / pipe 等は CGI 実行のためにのみ使用

※ chunked リクエストの復号は課題PDFの明示要件。応答をHTTP/1.0にしても残す。チームではHTTP/1.1リクエストのchunkedを受理し、HTTP/1.0リクエストのTransfer-Encodingは不正として扱う。

## 9. 提出用 README.md 要件(英語)

- [ ] 1行目(斜体): *This project has been created as part of the 42 curriculum by \<login1\>[, \<login2\>...].*
- [ ] **Description**: プロジェクトの目的と概要
- [ ] **Instructions**: コンパイル・インストール・実行方法
- [ ] **Resources**: 参考資料(ドキュメント、記事、チュートリアル等)+ **AI の利用方法の説明(どのタスク・どの部分に使ったかを明記)**
- [ ] 必要に応じて追加セクション(使用例、機能一覧、技術選定など)
- [ ] **必ず英語で書く**

## 10. AI 利用ルール(Chapter III 要約)

- AI 生成物は**完全に理解し、説明責任を持てるものだけ**使用する
- 生成物は必ず自分で検証・テスト・レビューする
- ピアレビューを必ず受ける
- 評価で説明できないコードは失格につながる(「Copilot が書いたので pipe の扱いを説明できない → 不合格」が悪い例として明記)
- README に AI 利用箇所を記載する義務(§9)

## 11. ボーナス(必須部分が完璧な場合のみ評価)

- [ ] Cookie とセッション管理(簡単な例を用意)
- [ ] 複数の CGI タイプ対応

## 12. 評価(defense)について

- Git リポジトリの内容のみが評価対象。ファイル名を再確認すること
- 評価中に**その場でのコード修正**を求められる可能性あり(数分で可能な小さな変更: 関数の修正、表示の変更、データ構造の調整など)
- 理解確認が目的。自分の開発環境で作業してよい

## 13. macOS 限定ルール(参考・チームは Linux のため対象外)

- macOS では write() の挙動差のため fcntl() 使用可
- fcntl は `F_SETFL`, `O_NONBLOCK`, `FD_CLOEXEC` のみ許可、他フラグ禁止
- fd はノンブロッキングモードで使用すること

---

# チーム決定事項

## 確定済み

| 項目 | 決定 | 理由 |
|---|---|---|
| 開発OS | Linux | - |
| I/O多重化 | **poll()** | 接続数規模ではO(n)で十分。APIが単純で状態機械の設計に集中できる。評価で説明しやすい |
| 体制 | 2人分担 | A: ネットワーク層+イベントループ / B: HTTP層+設定ファイル。CGIは合流して共同 |
| norminette | 適用しない | C言語用ツールのため C++ 課題は対象外(キャンパスローカルルールは要確認) |

## モジュール分割(実装単位)

| # | モジュール | 責務 | 担当 |
|---|---|---|---|
| 1 | ConfigParser | 設定ファイル → Config オブジェクト | B |
| 2 | ServerSocket | Config → listen 済み fd 群 | A |
| 3 | EventLoop | poll 本体。イベントを各所へ配送 | A |
| 4 | Client | 接続ごとの状態機械(受信/送信バッファ) | A |
| 5 | HttpRequest | HTTP/1.0・HTTP/1.1のバイト断片 → 完成判定+構造化(1.1のchunked受信に対応) | B |
| 6 | Router | Request+Config → 静的/CGI/アップロード/エラー振り分け | B |
| 7 | HttpResponse | ステータス+ヘッダー+ボディ → HTTP/1.0の送信バイト列 | B |
| 8 | CgiHandler | fork/execve/pipe 管理、環境変数構築 | 共同 |
| 9 | Utils | ログ、変換系の小物 | 共同 |

## 実装順序

1. ConfigParser(B)/ 最小 Hello World サーバー(A)← 並行スタート
2. EventLoop + Client(poll 化・複数接続)
3. HttpRequest パーサー(GET+ヘッダーから。受信バージョン別のHost・Expect判定を含む)
4. Router + HttpResponse(静的配信)← 合流ポイント
5. エラーページ・autoindex・DELETE
6. POST(アップロード)+ HTTP/1.1リクエストのchunked復号
7. CgiHandler(共同)
8. タイムアウト、body size 制限、ストレステスト、HTTP対応範囲の結合検証

## 決定事項(2026-08-14 確定)

※ HTTP対応範囲・接続方針は2026-10-04に更新。以下の表は更新後の決定を示す。

| 項目 | 決定 |
|---|---|
| ディレクティブ名 | NGINX風+独自名の混在(NGINXにある概念は同名、独自機能は独自名) |
| デフォルト値 | **設けない**。バリデーション時に必須項目が欠けていれば設定エラーとして弾く |
| 文法エラー時 | 警告メッセージを表示して起動せず終了(行番号表示はしない) |
| HttpRequest | 逐次パース型: `appendData(chunk)` + 状態問い合わせ方式 |
| HttpResponse | `serialize()` で一括生成 → 送信バッファへ |
| CGI | Python 第一。`cgi_extension .py /usr/bin/python3` 形式で拡張子→インタプリタ対応 |
| ログ | レベル付き Logger(`[日時] [LEVEL] msg`)、ファイル出力なし、レベルで絞り込み可 |
| 継承 | **継承なし**。全 location に全必須項目を明示(書き忘れ=起動拒否)。「書いた物がすべて」 |
| location マッチ | **最長前方一致**(より具体的に書いた区画が優先) |
| タイムアウト | 一律 60 秒。CGI のみ 10 秒で kill + 504 Gateway Timeout |
| 接続方針 | **1接続1要求**。`Connection: close` を付け、最終応答の全バイトを送信してから切断する |
| keep-alive / pipelining | **今回の実装対象外**。同じ接続で次の要求を処理しない |
| HTTP応答バージョン | **HTTP/1.0** 固定。課題要件に必要な機能を実装する |
| HTTP受信バージョン | **HTTP/1.0・HTTP/1.1**。Host・ボディ終端・Expectの扱いは受信バージョンで判定する。RFC全文への準拠は範囲外 |
| chunked | **HTTP/1.1リクエストの復号は実装する**。chunkedレスポンスは生成しない |
| ステータスコード | 主要コードを網羅的に実装(基本全部) |
| デフォルト設定パス | `./conf/default.conf`(引数なし起動時に読む) |
| シグナル処理 | `SIGPIPE` は `SIG_IGN` で無視(send失敗時はerrnoを見ず接続close)。`SIGINT` はフラグ方式でグレースフル終了(全fd close・全メモリ解放してから終了) |
| CGI環境変数 | RFC 3875 準拠: `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR, SERVER_SOFTWARE` + 全HTTPヘッダーを `HTTP_*` 形式(大文字化・`-`→`_`)で転送 |
| ConfigParser構成 | 1クラス(トークナイザとパーサーはprivate関数で分離) |
| CGI状態の持ち方 | 独立クラス `CgiProcess`(fd→オブジェクトの対応表で引ける形) |
| Router構成 | 1クラス+private関数分割(`handleGet/handlePost/handleDelete/handleCgi`...) |
| pollfd配列管理 | 毎周再構築(Client/CGI一覧から配列と監視フラグを毎回計算) |
| 例外方針 | 起動フェーズのみ例外可・ループ内は戻り値方式。保険としてループに `catch(std::exception&)` の防波堤(ログ+接続close)。bad_alloc もここで受ける |

## HTTP対応範囲(2026-10-04 更新)

課題PDF Version 24.1の印刷ページ7はHTTP/1.0を参考基準として推奨し、RFC全文の実装は要求していない。一方、印刷ページ8〜11のGET / POST / DELETE、アップロード、ブラウザ互換、設定機能、CGI、chunked復号などはHTTPバージョンにかかわらず満たす。

### リクエストの扱い

| 項目 | HTTP/1.0リクエスト | HTTP/1.1リクエスト |
|---|---|---|
| 受理 | 受理する | 課題に必要な範囲で受理する |
| Host | 省略可能 | 必須。欠落・重複・不正値は400 |
| ボディの長さ | ボディ付き要求はContent-Lengthで区切る | Content-Length、またはTransfer-Encoding: chunkedで区切る |
| Transfer-Encoding | 400を返して切断する | chunkedを復号する。Content-Lengthとの併記は400を返して切断する |
| Expect: 100-continue | この期待を無視して通常の要求として処理する | Expect未対応として、ボディを待たずヘッダー完了時点で417を返して切断する |
| その他のExpect値 | 詳細な扱いは未決定 | Expect未対応として417を返して切断する |

- 構文・ボディ長の不正など、先に確定したエラーはそのステータスで応答する。上のExpect方針のために400や413を417へ置き換える必要はない。
- Content-Lengthの不正値、桁あふれ、矛盾する重複は400。どちらの受信バージョンでも検証する。
- TCP接続の終了をリクエストボディの正常な終端として待たない。宣言された長さやchunkedの終端まで受信できず切断された要求は未完了として扱う。
- chunkedは終端chunk後のtrailer部分と最後の空行まで読み取り、復号後のボディにサイズ制限を適用する。trailerで既存ヘッダーを上書きしない。
- HTTP/1.1のExpect付き要求を無視してボディ待ちにしない。417への応答後、クライアントはExpectなしの別接続で再送できる。100 Continueを送る中間応答処理は実装しない。
- 1要求が完成した後の余剰データを次の要求として処理しない。keep-aliveやpipeliningの要求があっても、応答送信完了後に切断する。

### レスポンスとCGIの扱い

- ステータス行は `HTTP/1.0 <status-code> <reason-phrase>`。通常の本文付き応答は、本文のバイト数からContent-Lengthを確定し、Content-Typeとともに出力する。
- `Connection: close` を付ける。Transfer-Encodingは出力せず、chunked形式で送信しない。
- DELETE成功などで204を返す場合は、本文もContent-Lengthも出力しない。本文やContent-Lengthの可否はステータスに応じて判断する。
- 405・413・417・502・504など、実装する処理に応じたステータスはHTTP/1.0応答でも使用する。HTTP/1.0への変更を理由に削除しない。
- CGIの `SERVER_PROTOCOL` は受信した要求のバージョン(`HttpRequest::getVersion()`)を使う。応答側の固定値で上書きしない。
- chunkedで受信したボディをCGIへ渡す場合、`CONTENT_LENGTH` は復号後のバイト数から設定する。CGIへの全ボディの書き込み後は親側stdinパイプを閉じてEOFを渡す。
- CGI出力に長さがない場合はEOFまで受信する。既存の一括生成方針に従い、レスポンス本文の長さを確定してからHTTP/1.0応答を生成する。
- 逐次パース、部分送受信、タイムアウト、容量制限、CGIの後始末は引き続き必要。

### 結合時の確認項目

- [ ] HostなしのHTTP/1.0 GET → 正常配信、HTTP/1.0応答、送信完了後に切断
- [ ] 正常なHostを持つHTTP/1.1 GET → 正常配信、HTTP/1.0応答、送信完了後に切断
- [ ] Hostが欠落・重複・不正なHTTP/1.1要求 → 400
- [ ] HTTP/1.1のchunked POST → CGIが復号済みボディと正しいCONTENT_LENGTHを受け取る
- [ ] HTTP/1.0のTransfer-Encoding付き要求、およびHTTP/1.1のContent-Length / Transfer-Encoding併記 → 400で切断
- [ ] HTTP/1.1のExpect付きヘッダーだけを送る → ボディを待たず417。Expectなしで再送すると正常処理
- [ ] HTTP/1.0のExpect: 100-continue付き要求にContent-Length分のボディを送る → 100や417を返さず通常処理
- [ ] DELETE成功で204を返す場合 → 本文・Content-Lengthなし
- [ ] クライアントがkeep-aliveを要求しても、1回の応答送信完了後に切断

### 根拠

- 同梱 `webserv.pdf` Version 24.1: 印刷ページ7(HTTP/1.0基準)、8〜11(必須機能・CGI)
- [RFC 1945 §7.2.2](https://www.rfc-editor.org/rfc/rfc1945.html#section-7.2.2): HTTP/1.0のボディ長
- [RFC 9110 §2.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-2.5): 受信・応答のバージョン
- [RFC 9110 §10.1.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.1.1): Expectと417後の再送
- [RFC 9112 §3.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-3.2)、[§6.1](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.1): Host、Transfer-Encoding
- [RFC 3875 §4.1.16](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.16): CGIのSERVER_PROTOCOL

## 未決定(次に決めること)

- [ ] 下記「Config合意案」の文法・ディレクティブ一覧・必須/任意・NGINXとの差異を二人で確認して確定する
- [ ] HttpRequest / HttpResponse の**具体的なシグネチャ**(メソッド名・戻り値の確定)

## Config合意案(2026-10-04 作成・未承認)

以下は、NGINXの書き方を参考に、課題の設定要件を満たすための最小構成案。二人で確認するための草案であり、採用までは既存の決定事項・クラス案を置き換えない。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- 必須設定の値は補完しない。配信locationには無効な機能も `off` と明記する。
- `error_page` 未指定時の内蔵エラーページは課題要件として使用する。リダイレクト専用locationは配信設定を必要としない。これらを「全必須項目を明示」の適用範囲として二人で確認する。
- 1つのserverにつきlisten先は1組。複数のserverを並べて複数ポート・複数サイトに対応する。Hostによる仮想ホスト選択は行わない。
- NGINXの `root` と `alias` は意味を分けて採用する。両方を同じlocationに指定することは禁止する。

### 文法

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブとlocationブロック }
location    := location URL-prefix { location用ディレクティブ }
単純設定    := 名前 引数... ;
```

- 最上位には `server` だけを書く。各serverにはlocationを1個以上置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。
- 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。ブロック末尾の `}` に `;` は付けない。
- `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を含むトークンは設定エラーとする。
- 名前・メソッド・`on` / `off` は大文字小文字を区別する。ディレクティブの記述順には依存しない。
- 未知の名前、別階層の設定、引数数の違い、重複禁止項目の重複はすべて設定エラーとする。
- `include`、正規表現location、`=` / `^~` / `~` 等の修飾子、名前付きlocation、`if` / `rewrite` / `try_files`、設定reloadは実装対象外。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | 必須・1回 | IPv4アドレスと1〜65535のポートを明示。`0.0.0.0`も可。ポートだけの指定、ホスト名、IPv6、追加オプションは扱わない |
| `client_max_body_size` | `client_max_body_size 1048576;` | 必須・1回 | 正の十進整数、単位はバイト。`0`、符号、`1m`等の単位、size_tの範囲を超える値は拒否。復号後ボディに適用する |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可 | 1行に400〜599のコード1個とファイルパス1個。同一コードの重複は禁止。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | 必須・1個以上 | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させない |

- 同じIPv4:portの重複は起動エラー。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。異なる特定IPv4アドレス同士は同じポートを指定できる。
- `error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- `server_name` は実装しない。接続を受け付けたlisten先のServerConfigを使用する。

### locationの設定

locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 必須・1回 | 必須・1回 |
| `root` | `root ../www/site1;` | `root` / `alias` の片方だけ必須 | 指定不可 |
| `alias` | `alias ../uploads/;` | `root` / `alias` の片方だけ必須 | 指定不可 |
| `index` | `index index.html;` | 必須・1回 | 指定不可 |
| `autoindex` | `autoindex off;` | 必須・1回 | 指定不可 |
| `upload_store`（独自） | `upload_store off;` または `upload_store ../uploads;` | 必須・1回 | 指定不可 |
| `cgi_extension`（独自） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;` | 必須・1回 | 指定不可 |
| `return` | `return 301 http://127.0.0.1:8080/;` | 指定不可 | 必須・1回 |

- `allow_methods` はGET / POST / DELETEから1個以上、重複なし。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止されたメソッドは405とAllowヘッダーで応答する。
- `index` は単一ファイル名のみ。`/`を含むパスや `.` / `..` は拒否する。指定ファイルがない場合はautoindexへ進み、`off`なら403。indexファイルの存在自体は起動条件にしない。
- `autoindex` は `on` / `off` のみ。出力形式はHTMLに固定する。
- `upload_store off` はサーバーによるアップロード保存を無効にする。パス指定時はそのディレクトリへ保存する。GETで公開するには `root` / `alias` も保存先に対応させる。
- `cgi_extension` は `off`、または拡張子とインタプリタの組1個。拡張子は `.py` に固定せず、`.` と1文字以上の英数字からなる値を受理し、大文字小文字を区別する。動作確認はまずPythonで行う。インタプリタは絶対パスで明示する。拡張子一致で直接CGIを起動し、FastCGI設定は扱わない。
- 同じlocationでアップロード保存とCGIを両方有効にはしない。必要ならlocationを分ける。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- 配信locationでPOSTを許可するなら、アップロード保存またはCGIのどちらかを有効にする。CGIを有効にするlocationの許可メソッドはGET / POSTに限定する。
- `return` は301 / 302と `http://` または `https://` で始まる固定URLのみ。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。
- 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。これは本課題用の処理順序とする。

### locationの選択とパス

- 既決定どおり、queryを除いたURLパスに対する**最長の文字列前方一致**。設定順には依存しない。例えば `/a` は `/abc` にも一致する。ディレクトリ単位の設定には `/a/` を使う。
- `/` は全パスに一致する通常のlocation。定義は推奨するが必須ではなく、一致するlocationがなければ404。
- locationのprefixにはquery、`%`によるエンコード、`.` / `..` のパス要素を記述しない。要求URIのデコード・正規化・公開範囲外へのアクセス防止はHTTP/Router側の責務とし、詳細は別途合意する。
- `root` は設定ディレクトリにURLパス全体を追加する。`alias` は一致したlocationのprefixを設定ディレクトリへ置き換える。
- aliasはディレクトリ対応だけを扱い、**locationのprefixとaliasの値の両方を `/` で終える**。正規表現や単一ファイルへのaliasは扱わない。

| 設定 | 要求パス | 探すファイル |
|---|---|---|
| `location /images/` + `root /data;` | `/images/a.png` | `/data/images/a.png` |
| `location /images/` + `alias /data/pictures/;` | `/images/a.png` | `/data/pictures/a.png` |
| `location /kapouet/` + `alias /tmp/www/;` | `/kapouet/pouic/toto/pouet` | `/tmp/www/pouic/toto/pouet` |

最後の例で課題PDFのprefix置換を満たす。`root` にaliasの意味を持たせない。`/images/` は `/images` に一致しないため、末尾 `/` なしも受けたい場合のリダイレクト方針は別途HTTP側で決める。

- ファイルシステムの相対パスは**設定ファイルを置いたディレクトリ基準**。例えば `conf/default.conf` 内の `root ../www/site1;` は `www/site1` を指す。URL-prefix・returnのURL・indexのファイル名にはこの基準を適用しない。
- root / alias / upload_storeの対象ディレクトリは起動前に用意する。ConfigParserは存在・ディレクトリ種別を検証し、自動作成しない。CGIインタプリタも通常ファイル・実行可能であることを確認する。
- 起動後のファイル消失や権限変更は各要求のエラーとして処理する。起動時の検証だけで実行時の確認を省略しない。

### 設定例

`conf/default.conf` に置く想定。ディレクトリ・HTML・CGI等のデモファイルは別途用意する。この例は本課題用であり、そのままNGINXに読み込ませる設定ではない。

```nginx
server {
    listen 127.0.0.1:8080;
    client_max_body_size 1048576;
    error_page 404 ../www/errors/404.html;

    location / {
        allow_methods GET;
        root ../www/site1;
        index index.html;
        autoindex off;
        upload_store off;
        cgi_extension off;
    }

    location /files/ {
        allow_methods GET POST DELETE;
        alias ../uploads/;
        index index.html;
        autoindex on;
        upload_store ../uploads;
        cgi_extension off;
    }

    location /cgi/ {
        allow_methods GET POST;
        alias ../cgi-bin/;
        index index.py;
        autoindex off;
        upload_store off;
        cgi_extension .py /usr/bin/python3;
    }

    location /old/ {
        allow_methods GET;
        return 301 http://127.0.0.1:8080/;
    }
}

server {
    listen 127.0.0.1:8081;
    client_max_body_size 1048576;

    location / {
        allow_methods GET;
        root ../www/site2;
        index index.html;
        autoindex off;
        upload_store off;
        cgi_extension off;
    }
}
```

### A/B間の契約と検証

- BのConfigParserが全設定を解析・検証してから、不変のConfigをAへ渡す。1つでも設定エラーがあれば設定全体を不採用とし、listenを開始しない。
- AはServerConfigごとにlistenし、Clientにその設定を紐付ける。body上限は最初のappendDataより前にHttpRequestへ設定する。bind等の失敗で起動を中止する場合は、作成済みfdも閉じる。
- 採用時のLocationConfigは、共通の `path` / `allowedMethods` と、配信かリダイレクトかの種別を持つ。配信側は `pathMode(ROOT/ALIAS)` / `basePath` / `index` / `autoindex` / `uploadPath` / CGI設定、リダイレクト側はcode / URLを持つ。既存クラス案の `root` だけではrootとaliasを区別できないため、採用時に変更する。
- `upload_store off` は空のuploadPath、`cgi_extension off` は空のcgiExtensionsとして保持できる。構文解析時は「未指定」と「明示off」を別に記録し、必須性を検証してからConfigを完成させる。
- 文法エラーは既決定どおり、理由を表示して起動を中止する。行番号は実装しない。設定項目名と、分かる場合はserverのlisten先またはlocationをメッセージに含める。
- 二人で共有する確認例: 上記設定の受理、必須項目不足、未知設定、階層違い、root/alias併記、同一listen先、数値範囲外、redirect専用location、error_page未指定時の内蔵ページ、root/aliasの対応例。

### NGINXとの差異と二人の確認事項

| 確認 | 合意したい内容 |
|---|---|
| [ ] | NGINX風の限定文法を採用し、http/events・継承・変数・include・正規表現を実装しない |
| [ ] | 上記の必須項目と明示offを採用する。error_page未指定時は内蔵ページ、redirect専用locationは配信設定不要とする |
| [ ] | rootはURI全体追加、aliasはprefix置換とする。aliasは末尾 `/` のディレクトリ対応に限定する |
| [ ] | listenはIPv4:portを1serverに1組、body上限は正のバイト数に限定する。NGINXの `0 = 無制限` は採用しない |
| [ ] | error_pageはNGINXのURI指定・内部リダイレクトを省略し、エラー本文用のファイルパスを直接指定する |
| [ ] | allow_methods / upload_store / cgi_extensionは本課題用の独自設定とし、NGINXのアクセス制御・一時ボディ保存・FastCGI設定とは区別する |
| [ ] | 相対ファイルパスの基準を設定ファイルのディレクトリに統一する |

アップロード形式・ファイル名・上書き、URI正規化・symlink、CGI出力解析の詳細はHTTP側の別議題として残す。ConfigParserにそれらの処理を持たせない。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト

---

# クラス設計(全13クラス+2表)

## 一覧と責務

### 設定ドメイン(B担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ConfigParser` | ファイル→トークン化→構文解析→validation。エラーなら警告して終了 | 起動時1回・使い捨て |
| `Config` | パース結果の最上位入れ物 | 起動時1回・終了まで不変 |
| `ServerConfig` | server ブロック1個分のデータ | 同上 |
| `LocationConfig` | location ブロック1個分のデータ | 同上 |

※ Config 系3クラスは読み取り専用の構造体(ロジックは持たない)。

### ネットワーク/イベントドメイン(A担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ServerSocket` | socket→setsockopt→bind→listen→ノンブロッキング化(1ポート分) | 起動時・ポート数分 |
| `EventLoop` | poll 本体。fd 登録簿、イベント配送、タイムアウト見回り、終了時の後始末 | 起動時1個 |
| `Client` | 接続1本の状態機械(バッファ・状態・最終活動時刻) | accept 時に生成 / close 時に破棄 |
| `CgiProcess` | 実行中 CGI 1本の状態(pid・パイプfd・バッファ・開始時刻) | CGI 開始時に生成 / waitpid 後に破棄 |

### HTTPドメイン(B担当)

| クラス | 責務 |
|---|---|
| `HttpRequest` | HTTP/1.0・HTTP/1.1の逐次パース(断片投入→状態遷移→完成判定)。受信バージョン別の検証、ヘッダー段階の拒否、1.1のchunkedデコードを含む |
| `HttpResponse` | HTTP/1.0レスポンス組み立て+`serialize()`+エラーページ工場 |
| `Router` | Request+Config → 静的/autoindex/アップロード/DELETE/リダイレクト/CGI/エラーの振り分け |
| `CgiExecutor` | pipe×2→fork→dup2→chdir→execve と環境変数(char**)組み立て。起動のみ担当 |

### 共通(共同)

| クラス/表 | 責務 |
|---|---|
| `Logger` | `info/warn/error`。レベルフィルタ。ファイル出力なし |
| MIME タイプ表 | 拡張子→Content-Type。static 関数1個 |
| ステータスコード表 | code→文言。HttpResponse 内の static 関数 |

## 所有関係

```
main
 └─ Config(不変・全員が const 参照で覗く)
 └─ EventLoop
     ├─ ServerSocket × ポート数
     ├─ Client × 接続数          ← accept 時 new / close 時 delete
     │   └─ HttpRequest(値として内包)
     │   └─ send_buffer(serialize 済み Response)
     └─ CgiProcess × 実行中CGI数  ← Router 経由で生成 / 完了時 delete
         └─ 親 Client へのポインタ
```

Router / CgiExecutor / HttpResponse は状態を持たない道具(生成物を返すだけ)。

## 主要インターフェース

### HttpRequest

```cpp
class HttpRequest {
public:
    enum State { PARSING_HEADERS, PARSING_BODY, PARSING_CHUNKED,
                 COMPLETE, ERROR };

    HttpRequest();

    // Client(A側)が呼ぶ
    void   appendData(const char* data, size_t len);  // recv 断片を投入
    State  getState() const;
    int    getErrorCode() const;      // ERROR 時: 400, 413, 417, 501 など
    void   setMaxBodySize(size_t n);  // Config の値を注入(早期413判定)

    // Router(B側)が COMPLETE 後に呼ぶ
    const std::string& getMethod() const;
    const std::string& getPath() const;         // ? 以降除去済み
    const std::string& getQueryString() const;  // ? の後ろ、生のまま
    const std::string& getVersion() const;     // 受信したHTTP/1.0またはHTTP/1.1
    std::string        getHeader(const std::string& key) const;  // 大小無視
    const std::string& getBody() const;         // un-chunk 済み
};
```

- エラー判定(400/413/417/501)はパース時点で Request 自身が行う
- ヘッダー完了時に受信バージョン別のHost・Transfer-Encoding・Expectを検証する。拒否する場合はボディを待たずERRORへ遷移し、Clientがエラー応答を送る
- ClientはappendDataのたびに状態を確認する。ERRORになった要求のボディ受信を続けてCOMPLETEを待たない
- ヘッダーキーは内部で小文字化して格納
- chunked デコードはこのクラスに閉じる。HTTP/1.1要求でのみ受理し、CGIには復号済みボディを渡す

### HttpResponse

```cpp
class HttpResponse {
public:
    HttpResponse();
    void setStatus(int code);   // reason phrase は内部表で自動解決
    void setHeader(const std::string& key, const std::string& value);
    void setBody(const std::string& body, const std::string& contentType);
                                // 通常の本文付き応答のContent-Length / Content-Typeを設定
    std::string serialize() const;
                                // HTTP/1.0固定。Server, Date, Connection: close自動付与
                                // 204では本文・Content-Lengthを出力しない
    static HttpResponse makeError(int code, const ServerConfig& conf);
                                // error_page 指定 or 内蔵デフォルトHTML
};
```

- `\r\n` の組み立ては serialize() に封じ込める
- Transfer-Encodingは出力しない。本文を送れないステータスはserialize時にも検証し、setStatus/setBodyの呼び出し順にかかわらず正しい形式にする
- 送信完了後はClientをCLOSINGへ遷移させる。次の要求を受けるための再初期化は行わない
- エラー生成は makeError に一元化(発生箇所: Router / Request / EventLoop)
- 注記: CGI 対応時(フェーズ7)に Router::handle の戻り値を拡張予定

### Config 系(読み取り専用)

```cpp
class LocationConfig {   // 全て getter のみ
    // path, root, allowedMethods, index, autoindex,
    // uploadPath(空=不可), cgiExtensions(map<ext,interpreter>), redirect
};
class ServerConfig {
    // host, port, clientMaxBodySize, errorPages(map<int,path>)
    const LocationConfig* findLocation(const std::string& path) const;
                         // 最長前方一致。マッチなしは NULL → 404
};
class Config {
    // servers への const アクセスのみ
};
```

### EventLoop / Client / CgiProcess(A側内部)

```cpp
class EventLoop {
public:
    EventLoop(const Config& conf);
    void run();   // シグナルフラグが立つまで無限ループ
private:
    void rebuildPollfds();          // 毎周再構築(決定事項)
    void handleListenEvent(int fd);
    void handleClientEvent(Client& c, short revents);
    void handleCgiEvent(CgiProcess& p, short revents);
    void checkTimeouts();           // 60秒 / CGI 10秒
    void cleanup();                 // 終了時: 全fd close・全delete
};

class Client {
public:
    enum State { READING_REQUEST, PROCESSING, WAITING_CGI,
                 WRITING_RESPONSE, CLOSING };
    // fd, state, recvBuffer, sendBuffer, HttpRequest, lastActivity,
    // 所属 ServerConfig* を保持
};

class CgiProcess {
    // pid, stdinFd(書込), stdoutFd(読取), 残り書込ボディ,
    // 出力蓄積バッファ, 開始時刻, 親 Client*
};
```

### CgiExecutor / Logger

```cpp
class CgiExecutor {
public:
    // 成功: 新しい CgiProcess を返す / 失敗: NULL(→ 502)
    static CgiProcess* launch(const HttpRequest& req,
                              const LocationConfig& loc,
                              const ServerConfig& srv,
                              Client& owner);
private:
    static char** buildEnvp(...);   // RFC 3875 変数 + HTTP_* 変換
};

class Logger {
public:
    enum Level { DEBUG, INFO, WARN, ERROR };
    static void setLevel(Level l);
    static void debug(const std::string& msg);
    static void info (const std::string& msg);
    static void warn (const std::string& msg);
    static void error(const std::string& msg);
    // 形式: [YYYY-MM-DD HH:MM:SS] [LEVEL] msg
};
```
