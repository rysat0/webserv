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

※ HTTP対応範囲・接続方針、ConfigとCGIの合意状況は2026-10-04に更新。以下の表は更新後の決定を示す。課題の要求とチームが採用した範囲・初期値は区別する。

| 項目 | 決定 |
|---|---|
| ディレクティブ名 | NGINX風+独自名の混在(NGINXにある概念は同名、独自機能は独自名) |
| デフォルト値 | **省略された設定にはNGINXの既定値を適用する**。明示offを強制しない。独自項目の省略値は下記Config節で別途定義する |
| 文法エラー時 | 警告メッセージを表示して起動せず終了(行番号表示はしない) |
| HttpRequest | 逐次パース型: `appendData(chunk)` + 状態問い合わせ方式 |
| HttpResponse | `serialize()` で一括生成 → 送信バッファへ |
| CGI | 必須部分はPythonのみ。`cgi_extension .py /usr/bin/python3` 形式で拡張子→インタプリタ対応。RFC 3875を参考に、下記CGI節の範囲に限定する |
| ログ | レベル付き Logger(`[日時] [LEVEL] msg`)、ファイル出力なし、レベルで絞り込み可 |
| 継承 | **継承なし**。各locationの明示値と、その項目の既定値で設定を完成させる。省略時の既定値補完と上位からの継承は区別する |
| location マッチ | **最長前方一致**(より具体的に書いた区画が優先) |
| タイムアウト | 通常の接続処理は60秒(起点等は未確定)。CGIは起動から10秒で打ち切り504。途中の出力では延長しない |
| 接続方針 | **1接続1要求**。`Connection: close` を付け、最終応答の全バイトを送信してから切断する |
| keep-alive / pipelining | **今回の実装対象外**。同じ接続で次の要求を処理しない |
| HTTP応答バージョン | **HTTP/1.0** 固定。課題要件に必要な機能を実装する |
| HTTP受信バージョン | **HTTP/1.0・HTTP/1.1**。Host・ボディ終端・Expectの扱いは受信バージョンで判定する。RFC全文への準拠は範囲外 |
| chunked | **HTTP/1.1リクエストの復号は実装する**。chunkedレスポンスは生成しない |
| ステータスコード | 主要コードを網羅的に実装(基本全部) |
| デフォルト設定パス | `./conf/default.conf`(引数なし起動時に読む) |
| シグナル処理 | `SIGPIPE` は `SIG_IGN` で無視(send失敗時はerrnoを見ず接続close)。`SIGINT` はフラグ方式でグレースフル終了(全fd close・全メモリ解放してから終了) |
| CGI環境変数 | 対応する変数は `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR, SERVER_SOFTWARE` と `HTTP_*`。値の出所・空値/省略・HTTPヘッダー転送の例外/重複規則は03のC12で確定する |
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
- CGI出力はContent-Lengthの有無にかかわらず上限付きで蓄積し、EOFと子の終了を確認してからHTTP/1.0応答を生成する。長さ指定があれば本文の実長と照合し、不一致は502。上限・エラーの詳細は下記CGI節を参照。
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

## CGI対応範囲・確定事項(2026-10-04)

RFC 3875を参考に、対応範囲を以下に限定する。課題要件の一覧(第8節)に加え、二人が採用した動作・初期値を記録する。下表は [03のC12](reports/03_config_cgi_upload.md) と同じID・内容で管理し、変更時は関連するHTTP・I/O・Config・テスト計画も同時に更新する。

| ID | 項目 | 決定 |
|---|---|---|
| CGI-01 | 対応言語 | 必須部分ではPythonのみ。PHP固有の対応は作らない |
| CGI-02 | 起動方法 | 設定したPythonの絶対パスを `execve` で実行する。シェルやshebangの解釈は実装しない |
| CGI-03 | 入力 | リクエストを受信・復号してから起動する。クエリは環境変数、ボディは加工せずstdinへ渡す |
| CGI-04 | 出力 | stdoutを上限付きで蓄積し、EOFと子の終了を確認してからHTTP応答を作る。ブラウザへの逐次転送はしない |
| CGI-05 | CGIの種類 | 通常のCGIだけを対象とする。HTTPレスポンス全文を直接出すNPH、FastCGI、常駐プロセスは対象外 |
| CGI-06 | 周辺機能 | 認証、セッション、逆引きDNS、PHP専用環境変数は追加しない。C11は必須部分の実装対象に含めない |
| CGI-07 | Config | 既存の `cgi_extension` を使う。タイムアウトや出力上限の設定項目は増やさず、まずコード内定数にする |
| CGI-08 | 起動条件 | 選択されたlocationのCGI設定と、設定した拡張子で実行対象を判定する。実行するスクリプトは実在する通常ファイルとして確認する |
| CGI-09 | メソッド | CGIはGET/POSTに限定する。DELETEは通常locationで提供し、CGI用locationへのDELETEは405にする。CGIファイルの削除処理へ進めない |
| CGI-10 | index | `/cgi/` のようなディレクトリ要求で、設定されたindexとして `index.py` が選ばれた場合も、同じCGI実行判定を適用する。index未指定時に `index.py` を自動補完する意味ではない |
| CGI-11 | 後続パス | `/cgi/test.py/extra` の基本的な後続パスに対応する。スクリプトを示すURL上のパスをSCRIPT_NAME、後続部分をPATH_INFO、`?`以降をQUERY_STRINGとして渡す。後続部分の意味はPython側が解釈する |
| CGI-12 | アップロードとの分離 | アップロード保存とCGI実行のlocation・ディスク上の保存先を分ける。アップロード先ではCGIを実行せず、CGI実行場所にはアップロードさせない |
| CGI-13 | 通常の文書応答 | Content-Type・空行・本文を解釈する。通常の文書応答ではContent-Typeを要求し、最初の空行より後ろを本文として扱う |
| CGI-14 | Status | CGIのStatusをHTTPステータス行へ反映し、Statusヘッダー自体はクライアントへ転送しない。通常の文書応答でStatus省略時は200とする |
| CGI-15 | ヘッダーの改行 | CGI出力のヘッダーはLF/CRLFの両方を受理する。HTTP応答のヘッダーはCRLFで生成し、本文の改行はこの変換の対象にしない |
| CGI-16 | クライアント向けリダイレクト | 絶対URLのLocationだけを返すCGI応答は、Locationを伴う302のHTTP応答に変換する |
| CGI-17 | Content-Length | 長さの指定がなければEOFまで読み、本文の実際のバイト数からHTTP応答のContent-Lengthを生成する。指定がある場合もCGI-04の方針で蓄積し、宣言値と本文の実長を照合する。不一致は502とする |
| CGI-18 | 内部リダイレクト | ローカルパスのLocationだけを返してサーバー内部で再処理するCGI応答は対象外とする。Configで指定するHTTPリダイレクトは引き続き実装する |
| CGI-19 | エラー対応 | スクリプトなしは404、読み取り不可は403、pipe/fork失敗は500、exec失敗・子の異常終了・不正なCGI出力は502とする |
| CGI-20 | 出力サイズ上限 | CGI 1件あたりヘッダー8 KiB、ヘッダーを含むstdout全体8 MiBをコード内定数の初期値として採用する。読み取り中に超過を検出し、CGIを打ち切って502とする。入力ボディ上限は既存のConfigを使う |
| CGI-21 | 同時実行数 | アプリケーション独自のCGI同時実行数上限は設けない。同時実行数を理由に503を返す分岐や待ち行列は設けない。OSの資源不足等によるpipe/fork失敗はCGI-19に従い500とする |
| CGI-22 | タイムアウト | CGI起動からの経過時間10秒で打ち切り、504とする。途中で出力があっても期限を延長しない。EOF後に子が終了しない場合も期限の対象とする |
| CGI-23 | クエリの引数展開 | `?a+b` 等のクエリをコマンドライン引数へ展開する機能は実装しない。クエリは未デコードのままQUERY_STRINGへ渡し、argvにはPythonとスクリプトの実行に必要な引数だけを渡す |
| CGI-24 | 不正出力の扱い | 必要な形式・値・長さの検査を行い、不正なCGI出力は502にする。不正な値の推測・補正や、矛盾する値を選び直して受理する処理は実装しない。具体的な検証規則は下記の未合意事項として確定する |

- CGI-03の入力はHTTPのchunked復号後のボディ。その後のフォーム分解・URLデコード・multipart分解はCGIへの受け渡しのためには行わない。
- `/cgi/test.py/extra?name=taro` はSCRIPT_NAME=`/cgi/test.py`、PATH_INFO=`/extra`、QUERY_STRING=`name=taro`。SCRIPT_NAMEはURL上のパスで、後続部分はファイルとして探索しない。追加パスなしのPATH_INFOの空値/省略規則は未確定。
- ヘッダー8 KiBは8,192バイト、stdout全体8 MiBは8,388,608バイト。1件あたりの上限であり、課題指定値ではなくチームの初期値。入力は既存の `client_max_body_size` に従う。
- 出力上限超過・タイムアウトではCGIを打ち切り、fdの後始末と子の回収を行う。最初の失敗理由を保持し、タイムアウト後の異常終了で504を502に上書きしない。
- CGIが正常に返したStatusと子の終了コードは区別する。204等の本文・Content-Lengthの可否は既存のHTTP規則を維持する。
- 親のパイプI/Oはノンブロッキングとし、入力書き込みと出力読み取りを同じpoll等で並行して進める。入力を書き終えたら親側stdinパイプを閉じる。
- 拡張子の文法・設定できる組数、環境変数やヘッダー検証の細則、担当境界・APIは未確定。詳細は03のC12と下記Config合意案を参照する。

## 未決定(次に決めること)

- [ ] 下記「Config合意案」の未チェック項目(upload_store / cgi_extensionの省略時off案・指定回数・残る文法細則・検証詳細等)を確定する。root/alias・error_page・相対パスとCGIの承認済み事項は維持する
- [ ] HttpRequest / HttpResponse / RouterとCGI受け渡しの**具体的なシグネチャ**(メソッド名・戻り値・所有権の確定)
- [ ] CGIの環境変数・パス正規化・出力ヘッダー検証・stderr等の残る詳細とA/B境界を [03のC12](reports/03_config_cgi_upload.md) に従って確定する

## Config合意案(2026-10-04・一部確定)

rootはURI全体追加、aliasはlocationのprefix置換、error_pageは直接ファイル指定(未指定・読み取り失敗時は内蔵ページ)、相対ファイルパスは設定ファイルのディレクトリ基準とすることが確定済み。CGIに関する設定方針も上記CGI節に従う。省略された設定にはNGINXの既定値を適用する(C-05)。明示offを強制しない。

限定文法の5点(C-04)も確定済み。server/location・`;`・`{}`・空白区切り・`#`コメントを使い、引用符・エスケープ・変数・include・正規表現・locationの入れ子・reload・http/eventsの外枠を省く。未対応の書き方・文法間違いは理由を表示して起動を中止する。

以下は [05 Configの設計案と合意事項](reports/05_config_agreement.md) と同じ詳細案。確定事項と既決定を除く文法・必須性・追加の検証条件は、二人で確認するための案として読む。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- **確定**: 省略された設定項目には下表のNGINX既定値を適用し、明示offを強制しない。明示された値を優先する。独自項目には対応するNGINX既定値がないため、別途省略値を決める。構文不正・不正な値をデフォルトで置き換えて受理する意味ではない。
- **確定**: `error_page` 未指定・読み取り失敗時は内蔵エラーページを使う。
- **案**: リダイレクト専用locationには配信設定を要求せず、配信項目のデフォルト補完・配信パスの検証も行わない。明示された配信設定との併記は拒否する。
- 1つのserverにつきlisten先は1組。複数のserverを並べて複数ポート・複数サイトに対応する。Hostによる仮想ホスト選択は行わない。
- **確定**: NGINXの `root` と `alias` は意味を分けて採用する。**案**: 同じlocationには片方だけを指定する。

### 省略時の既定値

C-05により、既定値のある設定を省略したことだけでは起動エラーにしない。以下は本課題で扱う設定項目に限った表であり、未対応のNGINXディレクティブや継承機構を追加する意味ではない。

| 項目 | 省略時の値・動作 | 位置付け |
|---|---|---|
| `listen` | NGINXの規則に従い、特権起動時は全IPv4アドレスの80番、その他は8000番 | 省略可。起動権限を判定する実装方法は許可関数と照合して確定する。bind失敗時に別ポートへ切り替える規則ではない |
| `client_max_body_size` | 1 MiB = 1,048,576バイト | NGINXの`1m`を値として採用。設定ファイルで単位接尾辞を受理するかは別の文法の論点 |
| `root` | `html` | 配信locationでroot/aliasの両方がなければ適用。aliasが明示されている場合は補完しない |
| `alias` | 指定なし | 勝手にaliasの対象を作らない |
| `index` | `index.html` | `index.py`は自動選択しない。CGIのindexを使う場合は明示する |
| `autoindex` | `off` | 明示offと省略は同じ動作 |
| `error_page` | 個別ファイル指定なし。内蔵エラーページを使う | C-02を維持 |
| `return` | リダイレクトなし | 通常の配信locationとして扱う案 |
| `upload_store`(独自) | **案**: `off` | NGINX既定値ではない。省略時はアップロード保存を無効にする案 |
| `cgi_extension`(独自) | **案**: `off` | NGINX既定値ではない。Pythonのパスを推測しない |
| `allow_methods`(独自) | **確定**: GETのみ許可 | C-06のチーム独自の既定値。POST・DELETEは必要なlocationで明示する。CGI用locationのGET/POST限定・DELETEへの405は維持する |

既定の`root html;`にもC-03を適用する。例えば`conf/default.conf`では`conf/html`を指す。NGINXのインストール先等のパスをそのまま使うわけではない。既定値の補完と上位設定の継承は別であり、継承なしの方針は維持する。必須のブロック構造や引数不足・重複・不正値の検証も、設定項目の省略とは区別する。

根拠: [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)。

### 文法

基本文法はC-04の5点で確定済み。以下では、必須個数など今回の合意に含まれない細則を案として区別する。

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブとlocationブロック }
location    := location URL-prefix { location用ディレクティブ }
単純設定    := 名前 引数... ;
```

- **確定**: 最上位には `server` だけを書き、server内に設定とlocationを置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。**案**: ファイルにはserverを1個以上、各serverにはlocationを1個以上置く。
- **確定**: 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。単純設定は`;`で終える。改行そのものは設定の終わりではなく、`;`を省略できない。ブロックは`{}`で囲む。**案**: ブロック末尾の `}` に `;` は付けない。
- **確定**: `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を使った未対応の書き方は設定エラーとする。
- **案**: 名前・メソッド・`on` / `off` は大文字小文字を区別する。ディレクティブの記述順には依存しない。
- **確定**: 未対応の書き方・文法間違いは理由を表示して起動を中止する。未知の名前や未閉じブロックを無視・補正して起動しない。**案**: 各項目の階層・引数数・重複禁止規則は下表と検証条件を確認して確定する。
- **確定**: `include`、正規表現location、設定reloadは実装対象外。設定変更後は再起動する。**案**: `=` / `^~` / `~` 等の修飾子、名前付きlocation、`if` / `rewrite` / `try_files` も扱わない。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | 省略可・明示は最大1回の案 | IPv4アドレスと1〜65535のポートを明示。`0.0.0.0`も可。ポートだけの指定、ホスト名、IPv6、追加オプションは扱わない |
| `client_max_body_size` | `client_max_body_size 1048576;` | 省略可・明示は最大1回の案 | 正の十進整数、単位はバイト。`0`、符号、`1m`等の単位、size_tの範囲を超える値は拒否。復号後ボディに適用する |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可 | 1行に400〜599のコード1個とファイルパス1個。同一コードの重複は禁止。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | 必須・1個以上 | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させない |

- 同じIPv4:portの重複は起動エラー。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。異なる特定IPv4アドレス同士は同じポートを指定できる。
- `error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- `server_name` は実装しない。接続を受け付けたlisten先のServerConfigを使用する。

### locationの設定

locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 省略可・既定GETのみ。明示は最大1回の案 | 同左 |
| `root` | `root ../www/site1;` | 省略可。root/aliasとも未指定なら`html` | 指定不可(案) |
| `alias` | `alias ../uploads/;` | 省略可。明示rootとの併記は禁止の案 | 指定不可(案) |
| `index` | `index index.html;` | 省略可・既定`index.html`。明示は最大1回の案 | 指定不可(案) |
| `autoindex` | `autoindex off;` | 省略可・既定off。明示は最大1回の案 | 指定不可(案) |
| `upload_store`（独自） | `upload_store off;` または `upload_store ../uploads;` | **案**: 省略時off、明示は最大1回 | 指定不可(案) |
| `cgi_extension`（独自） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;` | **案**: 省略時off、明示は最大1回 | 指定不可(案) |
| `return` | `return 301 http://127.0.0.1:8080/;` | 指定不可 | 必須・1回 |

- `allow_methods` はGET / POST / DELETEから1個以上、重複なし。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止されたメソッドは405とAllowヘッダーで応答する。
- **確定**: `allow_methods` を省略したlocationはGETのみ許可する。POSTを受けるCGI・アップロード用locationには、例えば `allow_methods GET POST;` を明示する。通常locationでDELETEも提供するなら `allow_methods GET POST DELETE;` 等を明示する。許可メソッドの設定だけでCGIやアップロード保存が有効になるわけではない。
- `index` は単一ファイル名のみ。`/`を含むパスや `.` / `..` は拒否する。指定ファイルがない場合はautoindexへ進み、`off`なら403。indexファイルの存在自体は起動条件にしない。
- **確定**: 設定されたindexとして `index.py` が選ばれた場合も、locationのCGI設定と拡張子で判定して実行する。index省略時は`index.html`を使い、`index.py`の自動補完は行わない。
- `autoindex` は `on` / `off` のみ。出力形式はHTMLに固定する。
- `upload_store off` はサーバーによるアップロード保存を無効にする。パス指定時はそのディレクトリへ保存する。GETで公開するには `root` / `alias` も保存先に対応させる。
- **確定**: `cgi_extension` で拡張子とPythonの絶対パスを指定する。必須部分はPythonのみ。locationのCGI設定と拡張子で実行対象を判定し、通常ファイルのスクリプトを `execve` で起動したPythonに渡す。FastCGI設定は扱わない。
- **案**: `cgi_extension` は `off`、または拡張子とインタプリタの組1個。拡張子は `.py` に固定せず、`.` と1文字以上の英数字からなる値を受理し、大文字小文字を区別する。受理する拡張子の文法と組数は未確定。
- **確定**: アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にはアップロードさせない。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- **案**: 配信locationでPOSTを許可するなら、アップロード保存またはCGIのどちらかを有効にする。
- **確定**: CGI用locationはGET / POSTに限定し、DELETEは405にする。通常locationでDELETEを提供し、CGIファイルの削除へ進めない。
- **確定**: `/cgi/test.py/extra` 等の基本的な後続パスはPATH_INFOとして渡す。スクリプトを特定する手順やパス正規化の詳細は別途合意する。
- **確定**: CGIは起動から10秒(出力による延長なし)、ヘッダー8 KiB、ヘッダーを含むstdout全体8 MiB。これらはコード内定数とし、追加のConfig項目は設けない。独自のCGI同時実行数上限も設けない。入力上限には既存の `client_max_body_size` を使う。
- `return` は301 / 302と `http://` または `https://` で始まる固定URLのみ。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。
- 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。これは本課題用の処理順序とする。

### locationの選択とパス

- **最長前方一致は確定**。詳細案は、queryを除いたURLパスに対する文字列前方一致とし、設定順には依存しない。例えば `/a` は `/abc` にも一致する。ディレクトリ単位の設定には `/a/` を使う。パス境界を考慮するかは、この案を二人で確認して確定する。
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

- **確定**: ファイルシステムの相対パスは**設定ファイルを置いたディレクトリ基準**。例えば `conf/default.conf` 内の `root ../www/site1;` は `www/site1` を指す。URL-prefix・returnのURL・indexのファイル名にはこの基準を適用しない。
- root / alias / upload_storeの対象ディレクトリは起動前に用意する。ConfigParserは存在・ディレクトリ種別を検証し、自動作成しない。CGIインタプリタも通常ファイル・実行可能であることを確認する。
- 起動後のファイル消失や権限変更は各要求のエラーとして処理する。起動時の検証だけで実行時の確認を省略しない。

### 設定例

以下は、Config・CGIの確定事項と残りの詳細案を組み合わせた例。独自項目の省略値・検証詳細の最終確定後に実装用設定へ反映する。以下の例は既定値も一部明示しているが、NGINX既定値のある項目の記述を必須とする意味ではない。

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

- BのConfigParserが全設定を解析し、省略された設定の既定値を補完・パス解決・検証してから、不変のConfigをAへ渡す。1つでも設定エラーがあれば設定全体を不採用とし、listenを開始しない。
- AはServerConfigごとにlistenし、Clientにその設定を紐付ける。body上限は最初のappendDataより前にHttpRequestへ設定する。bind等の失敗で起動を中止する場合は、作成済みfdも閉じる。
- LocationConfigのデータ表現案は、共通の `path` / `allowedMethods` と、配信かリダイレクトかの種別を持つ。配信側は `pathMode(ROOT/ALIAS)` / `basePath` / `index` / `autoindex` / `uploadPath` / CGI設定、リダイレクト側はcode / URLを持つ。rootとaliasを区別する動作は確定済みだが、フィールド名や型は未確定。
- `upload_store off` は空のuploadPath、`cgi_extension off` は空のcgiExtensionsとして保持できる。解析時は明示の有無を保持し、明示値を優先して省略分だけを補完する。root/alias・returnの選択後に、そのlocationで使う設定だけを補完・検証する案。独自項目の省略時offを採用した場合、完成後のConfigでは省略と明示offを同じ値にできる。
- 文法エラーは既決定どおり、理由を表示して起動を中止する。行番号は実装しない。設定項目名と、分かる場合はserverのlisten先またはlocationをメッセージに含める。
- 二人で共有する確認例: 上記設定の受理、省略時の既定値、明示値による上書き、引数不足、未知設定、階層違い、root/alias併記、同一listen先、数値範囲外、redirect専用location、error_page未指定時の内蔵ページ、root/aliasの対応例。

### 合意状況と残る確認事項

チェック済みは今回の会話で確定した内容。未チェックは、先ほどの提案として残している詳細であり、すべてが承認済みという意味ではない。

| 状態 | 内容 |
|---|---|
| [x] | rootはURI全体追加、aliasはlocationのprefix置換として、NGINXと同じ意味で区別する |
| [x] | error_pageはエラー本文用のファイルパスを直接指定する。未指定・読み取り失敗時は内蔵ページを使う |
| [x] | 相対ファイルパスの基準を設定ファイルのディレクトリに統一する |
| [x] | cgi_extensionで指定したPythonの絶対パスを使い、locationのCGI設定と拡張子で起動を判定する。必須部分はPythonのみ |
| [x] | CGI用locationはGET/POST。DELETEは405。設定されたindex.pyにもCGI判定を適用し、基本的なPATH_INFOに対応する |
| [x] | アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける |
| [x] | CGIの時間・出力上限はコード内定数とし、Configを増やさない。CGI同時実行数の独自上限は設けない |
| [x] | C-04の限定文法5点を採用する。server/location・`;`・`{}`・空白区切り・`#`コメントを使い、引用符・エスケープ・変数・include・正規表現・locationの入れ子・reload・http/eventsの外枠を省く。文法間違いは起動エラー |
| [ ] | 残る文法の細則(ブロック個数、ブロック後の`;`、名前の大小区別・記述順、location修飾子等の扱い)を確定する |
| [x] | 省略された設定にはNGINXの既定値を適用し、明示offを強制しない(C-05)。継承なし・相対パスの基準は維持する |
| [x] | allow_methods省略時はGETのみ許可する(C-06)。POST・DELETEは必要なlocationで明示する。CGI用locationへのDELETEは引き続き405 |
| [ ] | upload_store / cgi_extensionの省略時off案、listen省略時の権限判定方法、指定回数とリダイレクト専用locationの条件を確定する |
| [ ] | root/aliasを排他にし、aliasはprefixと値が末尾 `/` のディレクトリ対応に限定する |
| [ ] | listenはIPv4:portを1serverに1組、body上限は正のバイト数に限定する。0による無制限指定は採用しない |
| [ ] | allow_methodsの検証詳細、upload_storeの独自設定名・書式と、cgi_extensionの拡張子文法・指定できる組数を確定する |
| [ ] | locationの前方一致でパス境界を考慮するか、上記の文字列前方一致案を確認する |
| [ ] | returnを301 / 302と固定の絶対URLに限定する |
| [ ] | 重複・数値範囲・パス存在等の起動時検証と、Configのデータ表現・A/B間のAPIを確定する |

アップロード形式・ファイル名・上書き、URI正規化・symlinkはHTTP側の別議題として残す。CGI出力の基本動作は確定済みで、ヘッダー検証等の残る詳細は [03のC12](reports/03_config_cgi_upload.md) に記録する。ConfigParserにそれらの処理を持たせない。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト

---

# クラス設計(全13クラス+2表)

以下は実装に向けた設計案。具体的なシグネチャ・型・所有権とCGI内の担当境界は未確定であり、上記の確定済み動作を満たす形で二人が合意する。モジュール一覧の `CgiHandler` はCGI処理の総称で、この案では `CgiExecutor` / `CgiProcess` 等に分ける。

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
| `ServerSocket` | socket→setsockopt→bind→listen→ノンブロッキング化(listen先のhost:port 1組分) | 起動時・listen先数分 |
| `EventLoop` | poll 本体。fd 登録簿、イベント配送、タイムアウト見回り、終了時の後始末 | 起動時1個 |
| `Client` | 接続1本の状態機械(バッファ・状態・最終活動時刻) | accept 時に生成 / close 時に破棄 |
| `CgiProcess` | 実行中 CGI 1本の状態(pid・パイプfd・バッファ・開始時刻・EOF/終了の確認・失敗理由) | CGI開始時に生成 / 出力の処理・子の回収・結果受け渡し・後始末が済んだ後に破棄 |

### HTTPドメイン(B担当)

| クラス | 責務 |
|---|---|
| `HttpRequest` | HTTP/1.0・HTTP/1.1の逐次パース(断片投入→状態遷移→完成判定)。受信バージョン別の検証、ヘッダー段階の拒否、1.1のchunkedデコードを含む |
| `HttpResponse` | HTTP/1.0レスポンス組み立て+`serialize()`+エラーページ工場 |
| `Router` | Request+Config → 静的/autoindex/アップロード/DELETE/リダイレクト/CGI/エラーの振り分け |

### 共通(共同)

| クラス/表 | 責務 |
|---|---|
| `CgiExecutor` | pipe/fork/dup2/chdir/execveによる起動と環境変数の受け渡し。Aがプロセス制御、Bが環境変数の内容を担当する案。具体的な境界・APIは03のC12で確定する |
| `Logger` | `info/warn/error`。レベルフィルタ。ファイル出力なし |
| MIME タイプ表 | 拡張子→Content-Type。static 関数1個 |
| ステータスコード表 | code→文言。HttpResponse 内の static 関数 |

## 所有関係

```
main
 └─ Config(不変・全員が const 参照で覗く)
 └─ EventLoop
     ├─ ServerSocket × listen先数
     ├─ Client × 接続数          ← accept 時 new / close 時 delete
     │   └─ HttpRequest(値として内包)
     │   └─ send_buffer(serialize 済み Response)
     └─ CgiProcess × 実行中CGI数  ← Router 経由で生成 / 完了時 delete
         └─ 対応するClientとの関連付け(方式・寿命管理は未確定)
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
- Routerの即時応答とCGI実行要求を区別する戻り値・非同期結果の受け渡しは、HTTP層とイベントループの結合前に確定する(03のC12、04のD2)。

### Config 系(読み取り専用)

```cpp
class LocationConfig {   // 全て getter のみ
    // 表現案: path, pathMode(ROOT/ALIAS), basePath,
    // allowedMethods, index, autoindex,
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
    void checkTimeouts();           // 通常60秒 / CGI起動から10秒、出力で延長しない
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
    // 出力蓄積バッファ, 開始時刻, stdoutEOF, 子終了状態, 失敗理由,
    // 対応するClientとの関連付け(具体的な型・寿命管理は未確定)
    // 読み取り中にヘッダー8 KiB・stdout全体8 MiBの上限を検査
};
```

### CgiExecutor / Logger

```cpp
class CgiExecutor {
public:
    // 起動APIの引数・戻り値・生成物の所有権は未確定(03のC12)。
    // pipe/fork失敗は500、exec失敗は502として区別できる形にする。
    // 起動後はイベントループで入出力を進め、EOFと子の終了を確認する。
    // 環境変数生成の配置とHTTP_*の転送規則も合意してから定義する。
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
