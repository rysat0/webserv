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
| 体制 | 2人分担 | tasugiya: ネットワーク層+イベントループ / rysato: HTTP層+設定ファイル。CGIは合流して共同 |
| norminette | 適用しない | C言語用ツールのため C++ 課題は対象外(キャンパスローカルルールは要確認) |

## モジュール分割(実装単位)

担当者の対応は **A = tasugiya、B = rysato** として確定。以降の担当表・設計案は名前で表記する。「共同」はtasugiyaとrysatoの両名を指す。CGI内の担当境界・APIなど未合意の詳細は、引き続き案として扱う。

| # | モジュール | 責務 | 担当 |
|---|---|---|---|
| 1 | ConfigParser | 設定ファイル → Config オブジェクト | rysato |
| 2 | ServerSocket | Config → listen 済み fd 群 | tasugiya |
| 3 | EventLoop | poll 本体。イベントを各所へ配送 | tasugiya |
| 4 | Client | 接続ごとの状態機械(受信/送信バッファ) | tasugiya |
| 5 | HttpRequest | HTTP/1.0・HTTP/1.1のバイト断片 → 完成判定+構造化(1.1のchunked受信に対応) | rysato |
| 6 | Router | Request+Config → 静的/CGI/アップロード/エラー振り分け | rysato |
| 7 | HttpResponse | ステータス+ヘッダー+ボディ → HTTP/1.0の送信バイト列 | rysato |
| 8 | CgiHandler | fork/execve/pipe 管理、環境変数構築 | 共同 |
| 9 | Utils | ログ、変換系の小物 | 共同 |

## 実装順序

1. ConfigParser(rysato)/ 最小 Hello World サーバー(tasugiya)← 並行スタート
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
| デフォルト値 | **省略された設定には原則NGINXの既定値を適用し、listenのみC-39の0.0.0.0:8000を優先する**。明示offを強制しない。独自項目の省略値は下記Config節で別途定義する |
| 文法エラー時 | 警告メッセージを表示して起動せず終了(行番号表示はしない) |
| HttpRequest | 逐次パース型: `appendData(chunk)` + 状態問い合わせ方式 |
| HttpResponse | `serialize()` で一括生成 → 送信バッファへ |
| CGI | 必須部分はPythonのみ。`cgi_extension .py /usr/bin/python3` 形式で拡張子→インタプリタ対応。RFC 3875を参考に、下記CGI節の範囲に限定する |
| ログ | レベル付き Logger(`[日時] [LEVEL] msg`)、ファイル出力なし、レベルで絞り込み可 |
| 継承 | **継承なし**。各locationの明示値と、その項目の既定値で設定を完成させる。省略時の既定値補完と上位からの継承は区別する |
| location マッチ | **文字列の最長前方一致**(C-23)。設定順に依存せず、`/a`は`/abc`にも一致する |
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
| Host | 省略可能。指定時の重複・空値・不正値は400(C-28・C-30) | 必須。欠落・重複・空値・不正値は400(C-28・C-30) |
| 要求行・ヘッダーの改行 | CRLF限定。単独LF・不正な単独CRは400(C-31) | 同左(C-31) |
| 要求行・ヘッダーのサイズ | 要求行8 KiB・ヘッダー合計32 KiB。超過時414/431(C-32) | 同左(C-32) |
| ヘッダーの折り返し・空白 | 折り返しとコロン直前の空白は400。値の前後SP/HTABだけを除去(C-33) | 同左(C-33) |
| ヘッダー名 | tokenを検証してASCII小文字で保持。空・不正文字・コロンなしは400(C-34) | 同左(C-34) |
| ボディの長さ | ボディ付き要求はContent-Lengthで区切る | Content-Length、またはTransfer-Encoding: chunkedで区切る |
| Content-Lengthの重複 | 最大1行。同値の重複・カンマ入りも400(C-35) | 同左(C-35) |
| 長さ指定が両方ない要求 | POSTも空ボディ。省略だけで411にしない(C-36) | 同左(C-36) |
| Transfer-Encoding | 400を返して切断する | chunkedを復号する。Content-Lengthとの併記は400を返して切断する |
| Expect: 100-continue | この期待を無視して通常の要求として処理する | Expect未対応として、ボディを待たずヘッダー完了時点で417を返して切断する |
| その他のExpect値 | 詳細な扱いは未決定 | Expect未対応として417を返して切断する |

- **確定(C-31)**: HTTP/1.0・HTTP/1.1要求のリクエスト行とHTTPヘッダーの改行はCRLFだけを受理し、単独LF・不正な単独CRは400とする。受信断片の末尾がCRだけなら、その時点では不正とせず次のバイトを待つ。次がLFなら正常なCRLFとして扱い、LF以外なら不正と判定する。切断やタイムアウトで未完了になった場合は既存の接続処理に従う。この改行制限はボディのデータには適用せず、ボディ中のCR/LFを変更・拒否しない。CGI出力ヘッダーのLF/CRLF両方受理と、HTTP応答ヘッダーのCRLF生成はCGI-15の既決定を維持する。
- **確定(C-32)**: HTTP/1.0・HTTP/1.1要求のリクエスト行は末尾CRLF込みで8 KiB(8,192バイト)、その直後のヘッダー部分全体は各行のCRLFと終端の空行込みで32 KiB(32,768バイト)を上限とする。ヘッダー合計にリクエスト行・ボディを含めない。URLデコード等の前の受信バイト数で数え、上限と同じサイズは受理、超過時はそれぞれ414 / 431で拒否する。受信中に検査し、超過が確定したら行末・ヘッダー終端・ボディの到着を待たずエラー応答へ進み、既存方針どおり応答送信完了後に切断する。同じrecvで届いたボディをヘッダーサイズへ加算しない。ヘッダー1行ごとの別上限と個数上限は追加しない。2つの上限はコード内定数とし、Config項目を増やさない。ボディ上限は既存client_max_body_sizeと413、CGI出力上限はCGI-20の規則を維持する。
- **確定(C-33)**: HTTP/1.0・HTTP/1.1要求のヘッダーは折り返し(obs-fold)を受け付けず、SP/HTABで始まる空でないヘッダー行は400とする。最初のヘッダー行の字下げやSP/HTABだけの行も同様に拒否する。フィールド名とコロンの間の空白・タブは400とし、補正しない。コロン後の値は前後のSP/HTABだけを除去し、値の内部の空白・タブはこの処理では変更しない。`Host:localhost` と `Host: localhost` は同じ値として扱う。ヘッダー終端は完全に空のCRLF行とする。この処理はHTTP要求ヘッダーに限り、ボディには適用しない。フィールドごとの値の検証(HostのC-30等)は別途行い、CGI出力の検証規則は変更しない。
- **確定(C-34)**: HTTP/1.0・HTTP/1.1要求のヘッダー名はRFC 9110のtoken(ASCII英数字と ! # $ % & ' * + - . ^ _ ` | ~ のいずれか1文字以上)として検証し、ASCII大文字A〜Zを小文字a〜zに変換して保持する。名前の比較は大小文字を区別しない。空の名前・許可外の文字・名前と値を区切るコロンの欠落は補正せず400とする。ヘッダー行は最初のコロンで名前と値に分け、値内のコロンはそのまま扱う。値はC-33の前後SP/HTAB除去を行うが、一律の小文字化はしない。個別ヘッダーの値の検証は既存方針に従う。この決定はHTTP要求のヘッダー名についてのもので、重複はHostのC-28・C-30とContent-LengthのC-35に従い、その他の重複、値の残る検証規則、CGI出力・HTTP_*転送の詳細や具体的なAPIは別途確定する。
- 構文・ボディ長の不正など、先に確定したエラーはそのステータスで応答する。上のExpect方針のために400や413を417へ置き換える必要はない。
- **確定(C-35)**: HTTP/1.0・HTTP/1.1要求のContent-Lengthは最大1行とする。同じ値でも2行以上は400とし、名前の大小文字だけが異なる場合もC-34に従って重複と判定する。値にカンマを含む場合は、同値の一覧を含めて400とする。複数行の結合、先勝ち・後勝ち、同値の単一化による補正は行わない。不正な数値・桁あふれの400と、Transfer-Encoding併記時の400は既存方針を維持する。検出したエラーはボディの完了を待たず既存のエラー応答経路へ渡し、応答送信完了後に切断する。この決定は要求のContent-Lengthについてであり、他のヘッダーやCGI出力に一律適用しない。Content-LengthとTransfer-Encodingが両方ない場合はC-36に従う。
- **確定(C-36)**: HTTP/1.0・HTTP/1.1要求でContent-LengthもTransfer-Encodingもない場合は、POSTを含めてボディ長0として扱う。他のヘッダー検証でエラーがなければヘッダー終端で要求の受信を完了し、ボディや接続切断を待たない。長さ指定の省略だけを理由に411を返さない。Content-Length: 0も空ボディとし、正常な正のContent-Lengthは指定バイト数、HTTP/1.1の正常なchunkedは終端まで受信する。存在する不正な長さ指定やHTTP/1.0のTransfer-Encodingを省略扱いにはせず、既存のエラー規則を適用する。要求完成後の余剰データはボディや次の要求として処理しない既存方針を維持する。空ボディで受信完了しても処理成功を保証せず、メソッド許可や処理側の検証を行う。CGIへ進む場合はstdinにデータを書かず親側の書き込みパイプを閉じてEOFを渡す。
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
- [ ] HTTP/1.1のHost欠落と、HTTP/1.0・1.1のHost重複・空値・不正値(C-30の対象外を含む) → 400
- [ ] HTTP/1.0・1.1の要求行/ヘッダーの単独LF・不正な単独CR → 400。CRとLFを別受信に分割した正常要求は受理。ボディ中のCR/LFは保持(C-31)
- [ ] 要求行8,192バイト・ヘッダー合計32,768バイトは受理、各1バイト超過で414/431。終端なしでも超過検出でエラー応答へ進み、同じ受信に含まれるボディはヘッダー合計へ数えない(C-32)
- [ ] HTTP/1.0・1.1の折り返し・行頭SP/HTAB・空白だけの行・コロン直前SP/HTABは400。値の前後SP/HTABを除去し、内部とボディは保持。空のCRLF行だけがヘッダー終端(C-33)
- [ ] HTTP/1.0・1.1ともContent-Length/content-length/CONTENT-LENGTHを個別に送って同じ名前として解釈。RFC tokenの記号を受理。空の名前・名前内の空白/非ASCII/スラッシュ・コロン欠落は400。Cookie等の値の大小文字を保持(C-34)
- [ ] HTTP/1.0・1.1ともContent-Lengthが1行の正常値は受理。同値/異値の2行、名前の大小文字違いの2行、カンマ入りの10, 10や10, 20は400。重複行を別々の受信断片で送っても同じ結果(C-35)
- [ ] 長さ指定が両方ないPOSTとContent-Length: 0は空ボディで受信完了し、ボディ待ちや411にしない。CGIへ進む場合はstdinのEOFを受け取れる。不正な長さ指定を空ボディ扱いしない(C-36)
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

## CGI対応範囲・確定事項(2026-10-05 更新)

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
| CGI-25 | スクリプトの特定 | 選択済みのCGI有効locationのroot/aliasで対応するパスを先頭から辿り、設定拡張子で終わる実在する通常ファイルに到達したところを実行対象とする。そのURL上のパスをSCRIPT_NAME、後続部分をPATH_INFOとする。拡張子だけで分割せず、同じ拡張子のディレクトリは実行しない |

- CGI-03の入力はHTTPのchunked復号後のボディ。その後のフォーム分解・URLデコード・multipart分解はCGIへの受け渡しのためには行わない。
- `/cgi/test.py/extra?name=taro` はSCRIPT_NAME=`/cgi/test.py`、PATH_INFO=`/extra`、QUERY_STRING=`name=taro`。SCRIPT_NAMEはURL上のパスで、後続部分はファイルとして探索しない。追加パスなしのPATH_INFOの空値/省略規則は未確定。
- ヘッダー8 KiBは8,192バイト、stdout全体8 MiBは8,388,608バイト。1件あたりの上限であり、課題指定値ではなくチームの初期値。入力は既存の `client_max_body_size` に従う。
- 出力上限超過・タイムアウトではCGIを打ち切り、fdの後始末と子の回収を行う。最初の失敗理由を保持し、タイムアウト後の異常終了で504を502に上書きしない。
- CGIが正常に返したStatusと子の終了コードは区別する。204等の本文・Content-Lengthの可否は既存のHTTP規則を維持する。
- 親のパイプI/Oはノンブロッキングとし、入力書き込みと出力読み取りを同じpoll等で並行して進める。入力を書き終えたら親側stdinパイプを閉じる。
- 拡張子の文法・設定できる組数はC-19で確定済み。環境変数やヘッダー検証の細則、担当境界・APIは未確定。詳細は03のC12と下記Config合意案を参照する。

## 未決定(次に決めること)

- [x] Configの6項目はC-37〜C-42で合意済み。実装・検証はこれから行う。HTTP/Router/CGIの残るAPI等は下記の別議題とする
- [ ] HttpRequest / HttpResponse / RouterとCGI受け渡しの**具体的なシグネチャ**(メソッド名・戻り値・所有権の確定)
- [ ] CGIの環境変数・ファイルパスの結合/検証の残る詳細(C-26の運用前提に従う)・出力ヘッダー検証・stderr等の残る詳細とtasugiyaとrysatoの担当境界を [03のC12](reports/03_config_cgi_upload.md) に従って確定する

## Config合意事項(2026-10-05・6項目合意完了)

rootはURI全体追加、aliasはlocationのprefix置換、error_pageは直接ファイル指定(未指定・読み取り失敗時は内蔵ページ)、相対ファイルパスは設定ファイルのディレクトリ基準とすることが確定済み。CGIに関する設定方針も上記CGI節に従う。省略された設定には原則NGINXの既定値を適用し、listenのみ0.0.0.0:8000固定とする(C-05・C-39)。明示offを強制しない。

限定文法の5点(C-04)も確定済み。server/location・`;`・`{}`・空白区切り・`#`コメントを使い、引用符・エスケープ・変数・include・正規表現・locationの入れ子・reload・http/eventsの外枠を省く。未対応の書き方・文法間違いは理由を表示して起動を中止する。

以下は [05 Configの設計と合意事項](reports/05_config_agreement.md) と同じ合意内容。Configの6項目はC-37〜C-42で確定済み。HTTP/Router/CGI等の処理側の未決定事項は別議題として扱う。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- **確定**: 省略された設定項目には下表の既定値(NGINXを基本としlistenのみC-39の例外)を適用し、明示offを強制しない。明示された値を優先する。独自項目の既定値はチームで定め、allow_methodsはGETのみ(C-06)、upload_store / cgi_extensionは無効(C-07)とする。構文不正・不正な値をデフォルトで置き換えて受理する意味ではない。
- **確定**: `error_page` 未指定・読み取り失敗時は内蔵エラーページを使う。
- **確定(C-22)**: returnがあるlocationはリダイレクト専用とする。allow_methodsは併記でき、省略時はGETのみ。root / alias / index / autoindex / upload_store / cgi_extensionの明示は、offであっても設定エラーとして起動を中止する。配信項目のデフォルト補完・配信パスの検証は行わない。この併記禁止は本課題用の独自制限である。
- **確定(C-11)**: listenの明示は1つのserverにつき最大1回。同じ値でも異なる値でも2回以上の指定は設定エラーで起動中止。省略時は既定値で1組を補完し、複数ポートはserverブロックを分けて対応する。**確定(C-12)**: Hostによる仮想ホスト選択は行わない。
- **確定**: NGINXの `root` と `alias` は意味を分けて採用する。**確定(C-08)**: 同じlocationでrootとaliasを併記した場合は、記述順によらず設定エラーとして起動を中止する。aliasのみ指定した場合は既定のrootを補完しない。

### 省略時の既定値

C-05により、既定値のある設定を省略したことだけでは起動エラーにしない。以下は本課題で扱う設定項目に限った表であり、未対応のNGINXディレクティブや継承機構を追加する意味ではない。

- **確定(C-39)**: listenを省略したserverは、起動権限に関係なく`0.0.0.0:8000`で待ち受ける。権限判定は実装しない。C-05のNGINX既定値採用方針のうちlistenだけをこの固定値に変更し、他の項目の既定値は維持する。明示したlistenはC-10・C-11に従ってそのIP・ポートを使用し、80番も指定可能とする。既定値補完後の重複・競合はC-12に従って起動エラーとし、複数serverでlistenを省略して同一アドレスになった場合も拒否する。bind失敗時は起動を中止し、別IP・別ポートへ自動変更しない。

| 項目 | 省略時の値・動作 | 位置付け |
|---|---|---|
| `listen` | `0.0.0.0:8000`(C-39) | 省略可。起動権限の判定なし。NGINXとの意図的な差異。bind失敗時は起動エラーとし、別ポートへ切り替えない |
| `client_max_body_size` | 1 MiB = 1,048,576バイト | NGINXの`1m`を既定値として採用。明示する場合は正の十進整数・バイト単位のみとし、単位接尾辞は受理しない(C-13) |
| `root` | `html` | 配信locationでroot/aliasの両方がなければ適用。aliasが明示されている場合は補完しない |
| `alias` | 指定なし | 勝手にaliasの対象を作らない |
| `index` | `index.html` | `index.py`は自動選択しない。CGIのindexを使う場合は明示する |
| `autoindex` | `off` | 明示offと省略は同じ動作 |
| `error_page` | 個別ファイル指定なし。内蔵エラーページを使う | C-02を維持 |
| `return` | リダイレクトなし | 通常の配信locationとして扱う(C-22) |
| `upload_store`(独自) | **確定**: 無効 | C-07。Webservによるアップロード保存を行わず、保存先を自動補完しない |
| `cgi_extension`(独自) | **確定**: 無効 | C-07。CGIを実行せず、拡張子やPythonのパスを自動補完しない |
| `allow_methods`(独自) | **確定**: GETのみ許可 | C-06のチーム独自の既定値。POST・DELETEは必要なlocationで明示する。CGI用locationのGET/POST限定・DELETEへの405は維持する |

既定の`root html;`にもC-03を適用する。例えば`conf/default.conf`では`conf/html`を指す。NGINXのインストール先等のパスをそのまま使うわけではない。既定値の補完と上位設定の継承は別であり、継承なしの方針は維持する。必須のブロック構造や引数不足・重複・不正値の検証も、設定項目の省略とは区別する。

根拠: [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)。

### 文法

基本文法はC-04、以下の文法細則はC-37で確定済み。locationのprefixはC-38に従う。起動時のファイル検証はC-40に従い、設定の組み合わせはC-41に従い、型/APIはC-42に従う。

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブ(任意)とlocationブロック(1個以上、順不同) }
location    := location URL-prefix { location用ディレクティブ(0個以上) }
単純設定    := 名前 引数... ;
```

- **確定**: 最上位には `server` だけを書き、server内に設定とlocationを置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。**確定(C-37)**: ファイルにはserverを1個以上、各serverにはlocationを1個以上置く。空ファイル・コメントだけのファイル・locationがないserverは設定エラーで起動を中止する。
- **確定**: 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。単純設定は`;`で終える。改行そのものは設定の終わりではなく、`;`を省略できない。ブロックは`{}`で囲む。**確定(C-37)**: ブロック末尾の `}` に `;` は付けず、`};` は設定エラーとする。
- **確定**: `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を使った未対応の書き方は設定エラーとする。
- **確定**: メソッドはC-17、`on` / `off`はC-16・C-18・C-19の大小文字規則に従う。**確定(C-37)**: server/locationを含む設定名は大文字小文字を区別する。`root`を受理し、`ROOT` / `Root`は未知の設定として拒否する。同じ階層内ではディレクティブ・ブロックの記述順に依存しない。server内でlisten等をlocationの後に記述してもよい。重複はC-14に従って拒否し、後勝ちで上書きしない。
- **確定**: 未対応の書き方・文法間違いは理由を表示して起動を中止する。未知の名前や未閉じブロックを無視・補正して起動しない。各項目の階層・引数数・重複禁止規則は下表の確定事項に従い、実装時も確定済みの規則を適用する。
- **確定**: `include`、正規表現location、設定reloadは実装対象外。設定変更後は再起動する。**確定(C-37)**: 通常の `location URL-prefix { ... }` だけを受理し、`=` / `^~` / `~` / `~*` の修飾子、名前付きlocation、`if` / `rewrite` / `try_files` は設定エラーとして拒否する。
- **確定(C-37)**: `location / { }` のような空locationは文法上受理し、既決定の省略時規則を適用する(root html、index index.html、GETのみ、autoindex無効、upload/CGI無効)。空locationの受理は、root等の実在・権限検査を省く決定ではない。これらの起動時検証はC-40に従う。

**確定(C-14)**: 同じserver内のlisten / client_max_body_size、同じlocation内のroot / alias / index / autoindex / allow_methods / upload_store / cgi_extension / returnはそれぞれ最大1回。値が同じでも重複は設定エラーで起動を中止し、上書き・結合しない。rootとaliasの併記禁止(C-08)も維持する。error_pageは同じserver内でコードが異なれば複数可、locationは同じserver内でprefixが異なれば複数可。別ブロックで同じ設定項目を使うことや、1行のallow_methodsに複数メソッドを書くこととは区別する。1行の引数数・値の検証は、個別に確定した範囲に従う。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | **確定(C-11)**: 省略可・明示は最大1回。2回以上は起動エラー | **確定(C-10)**: 明示時はIPv4:portのみ。IPv4はドットで区切る4個の十進数(各0〜255)、ポートは十進数の1〜65535。`0.0.0.0`も可。ポートのみ、ホスト名、IPv6、追加オプション、不正な書式・範囲外の値は設定エラーで起動中止 |
| `client_max_body_size` | `client_max_body_size 1048576;` | 省略可・明示は最大1回(確定、C-14) | **確定(C-13)**: 正の十進整数・バイト単位のみ。`0`、符号、小数、`1m`等の単位接尾辞、std::size_tの範囲を超える値は設定エラーで起動中止。復号後ボディに適用し、上限と同じサイズは受理、超過は413 |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可(確定、C-14) | **確定(C-20)**: server内に3桁の十進数100〜599のコード1個とファイルパス1個。引数不足・余分な引数・範囲外・複数コード・ステータス変更指定は起動エラー。**確定(C-14)**: 同じserver内の同一コードの重複は起動エラー。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | **確定(C-37)**: 各serverに1個以上必須。複数可(C-14) | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させたら起動エラー(確定、C-14) |

- **確定(C-12)**: server間で同じIPv4:portが重複したら設定エラーで起動を中止する。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。listenを省略したserverも既定値を補完してから同じ規則で検証する。異なる特定IPv4アドレス同士は同じポートを指定できるが、実際のbind成功は起動時に確認する。
- **確定(C-20)**: error_pageはserver内に「100〜599の3桁の十進コード1個＋ファイルパス1個」を指定する。絶対パスと設定ファイル基準の相対パスを受理する。複数コードを1行に書く形式や `=200` 等のステータス変更は扱わない。同一server内の同一コード重複はC-14に従い起動エラー。設定の受付範囲を広げる決定であり、全コードの発生処理を追加する意味ではない。既存のHTTP規則が優先され、1xx・204・304等の本文を禁止する応答には、指定ファイルや内蔵ページの本文を付けない。未対応の応答コードを生成する機能も、この設定によって有効にはならない。**確定(C-20追記)**: 設定としては100〜599を受理するが、error_pageによる本文差し替えは400〜599の応答に限定する。100〜399の指定は応答に適用せず、通常の成功応答・リダイレクト・CGIの成功応答の本文を変更しない。100〜399の指定ファイルを読み込んだり、その未指定・読み取り失敗を理由に内蔵ページへ差し替えたりしない。

- error_pageの適用対象である400〜599の応答では、`error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- **確定(C-12)**: `server_name` とHostによる仮想ホスト選択は実装しない。接続を受け付けたlisten先のServerConfigを使用する。未対応のserver_name指定はC-04に従い設定エラーとする。HTTP/1.1要求のHost必須・欠落/重複/不正時400という検証は維持する。

### locationの設定

**確定(C-22)**: locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 省略可・既定GETのみ。大文字GET/POST/DELETEを重複なく1個以上(C-17)、明示は最大1回(C-14) | 同左 |
| `root` | `root ../www/site1;` | 省略可・明示は最大1回(C-14)。root/aliasとも未指定なら`html` | 指定不可(C-22) |
| `alias` | `alias ../uploads/;` | 省略可・明示は最大1回(C-14)。明示rootとの併記は禁止(確定、C-08) | 指定不可(C-22) |
| `index` | `index index.html;` | 省略可・既定`index.html`。ファイル名1個のみ(C-15)、明示は最大1回(C-14) | 指定不可(C-22) |
| `autoindex` | `autoindex off;` | 省略可・既定off。小文字on/offの1引数のみ(C-16)、明示は最大1回(C-14) | 指定不可(C-22) |
| `upload_store`（独自、C-18） | `upload_store off;` または `upload_store ../uploads;`(引数1個) | **確定**: 省略時は無効。明示は最大1回(C-14) | 指定不可(C-22) |
| `cgi_extension`（独自、C-19） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;`(1組のみ) | **確定**: 省略時は無効。明示は最大1回(C-14) | 指定不可(C-22) |
| `return`(C-21) | `return 301 http://127.0.0.1:8080/;`(2引数) | 指定不可 | 必須・1回(C-14・C-22) |

- **確定(C-17)**: `allow_methods` は大文字のGET / POST / DELETEから、空白区切りで重複なく1個以上指定する。空リスト・重複・小文字・カンマ区切り・その他の値(HEAD / PUT等)は設定エラーで起動中止。記述順に意味はなく、明示リストにGET等を自動追加しない。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止された対応メソッドは405とAllowヘッダーで応答する。HEAD等を後から採用する場合はHTTPの対応範囲とこのConfig規則を合わせて変更する。CGI用locationへのDELETEは引き続き405とする。
- **確定**: `allow_methods` を省略したlocationはGETのみ許可する。POSTを受けるCGI・アップロード用locationには、例えば `allow_methods GET POST;` を明示する。通常locationでDELETEも提供するなら `allow_methods GET POST DELETE;` 等を明示する。許可メソッドの設定だけでCGIやアップロード保存が有効になるわけではない。
- **確定(C-15)**: `index` は対象ディレクトリ内のファイル名1個のみ。複数候補、`/`を含むパス、`.` / `..` は設定エラーで起動を中止する。省略時はindex.html。indexファイルの存在自体は起動条件にしない。存在するディレクトリへのGET要求で指定ファイルがなければautoindexへ進み、onなら一覧、offなら403。ディレクトリ自体がなければ404。indexが存在するが読み取れない場合は、一覧表示へ切り替えない。末尾 `/` のない実ディレクトリへのGETは、C-27の301を先に返す。
- **確定**: 設定されたindexとして `index.py` が選ばれた場合も、locationのCGI設定と拡張子で判定して実行する。index省略時は`index.html`を使い、`index.py`の自動補完は行わない。
- **確定(C-16)**: `autoindex` の引数は小文字の `on` または `off` を1個だけ受理し、省略時はoff。引数なし・複数引数・`ON` / `true` / `1` 等は設定エラーで起動中止。出力形式はHTMLに固定し、autoindex_format / autoindex_exact_size / autoindex_localtime等の追加設定項目は実装しない。indexを優先するC-15の流れを維持する。offは一覧表示のみを無効にし、個別ファイルへのアクセスを禁止する設定ではない。一覧の並び順・表示項目は別途決める。
- **確定(C-18)**: 独自設定名は `upload_store` とし、引数は小文字 `off` または保存先ディレクトリのパス1個。引数不足・複数引数は設定エラーで起動中止。省略またはoffはWebservによるアップロード保存を無効にする。絶対パス・設定ファイル基準の相対パスを受理する。パス指定時は起動時に存在・ディレクトリ種別を検証し、不在・通常ファイルなら起動エラー。保存先や親ディレクトリを自動作成しない。GETで公開するにはroot/aliasを別途設定し、upload_storeから公開先を自動設定しない。実際の保存時の失敗処理、ファイル名・上書き規則は別途決める。
- **確定**: `cgi_extension` で拡張子とPythonの絶対パスを指定する。必須部分はPythonのみ。locationのCGI設定と拡張子で実行対象を判定し、通常ファイルのスクリプトを `execve` で起動したPythonに渡す。FastCGI設定は扱わない。
- **確定(C-19)**: `cgi_extension` は小文字 `off` の1引数、または拡張子とPythonの絶対パスの2引数(1組)のみ受理する。拡張子は `.py` に固定せず、`.` と1文字以上のASCII英数字(A-Z / a-z / 0-9)からなる値とし、大文字小文字を区別する。引数不足・余分な引数・複数組・不正な拡張子・Pythonの相対パスは設定エラーで起動中止。`*.py`等のパターンや正規表現は使わない。拡張子は実行対象の目印であり、`.cgi`等でも指定したPythonで実行する。省略時は無効(C-07)、同じlocationでの複数回指定は拒否(C-14)。Pythonの起動時検証はC-40に従う。
- **確定**: アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にはアップロードさせない。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- **確定(C-41)**: 同じlocation内の設定を、既定値補完後の有効/無効と許可メソッドで検証する。配信locationでPOSTを許可するならupload_storeまたはcgi_extensionのどちらか1つが有効であることを要求し、両方無効なら起動エラーとする。同じlocationでupload_storeとcgi_extensionが両方有効なら、POST許可の有無によらず起動エラー。CGI有効locationのallow_methodsにDELETEが含まれる場合も起動エラーとする。省略/offは無効として扱い、設定行の存在だけで有効とは判定しない。returnのリダイレクト専用locationはPOST処理先の検証対象外とし、C-22の併記禁止規則を維持する。CGIやuploadを有効にしてもPOSTは自動許可せず、GETのみでCGIを使う設定やupload有効かつPOST不許可の設定も受理する。起動後のCGI用locationへのDELETE要求は引き続き405とし、CGIファイルの削除へ進めない。CGIの拡張子判定、uploadとCGIのディスク上の保存先を分ける既決定も維持する。
- **確定**: CGI用locationはGET / POSTに限定し、DELETEは405にする。通常locationでDELETEを提供し、CGIファイルの削除へ進めない。
- **確定**: `/cgi/test.py/extra` 等の基本的な後続パスはPATH_INFOとして渡す。URLの分離・デコード・正規化はC-24・C-25に従う。symlinkの運用前提と制限はC-26に従う。スクリプト特定手順はCGI-25で確定済み。ファイルパスの結合・検証の残る詳細は別途合意する。
- **確定**: CGIは起動から10秒(出力による延長なし)、ヘッダー8 KiB、ヘッダーを含むstdout全体8 MiB。これらはコード内定数とし、追加のConfig項目は設けない。独自のCGI同時実行数上限も設けない。入力上限には既存の `client_max_body_size` を使う。
- **確定(C-21)**: `return` はlocation内に「301または302＋固定のHTTP(S)絶対URL」の2引数で指定する。URLは `http://` または `https://` で始まり、ホスト部分を持つものとする。相対URL、他のコード、引数不足・余分な引数、不正なURLは設定エラーで起動中止。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。設定したURLをLocationヘッダーでクライアントに返し、Webserv自身は移動先を取得しない。同じlocation内での複数回指定はC-14に従い拒否する。
- **確定(C-22)**: 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。禁止された対応メソッドは405とAllowヘッダーで応答し、リダイレクトしない。例えばreturnのみのlocationはGETなら指定した301/302、POST・DELETEなら405。これは本課題用の処理順序であり、未対応メソッドや不正なHTTP要求の扱いを新たに定めるものではない。

### locationの選択とパス

- **確定(C-36)**: HTTP/1.0・HTTP/1.1要求でContent-LengthもTransfer-Encodingもない場合は、POSTを含めてボディ長0として扱う。他のヘッダー検証でエラーがなければヘッダー終端で要求の受信を完了し、ボディや接続切断を待たない。長さ指定の省略だけを理由に411を返さない。Content-Length: 0も空ボディとし、正常な正のContent-Lengthは指定バイト数、HTTP/1.1の正常なchunkedは終端まで受信する。存在する不正な長さ指定やHTTP/1.0のTransfer-Encodingを省略扱いにはせず、既存のエラー規則を適用する。要求完成後の余剰データはボディや次の要求として処理しない既存方針を維持する。空ボディで受信完了しても処理成功を保証せず、メソッド許可や処理側の検証を行う。CGIへ進む場合はstdinにデータを書かず親側の書き込みパイプを閉じてEOFを渡す。
- **確定(C-35)**: HTTP/1.0・HTTP/1.1要求のContent-Lengthは最大1行とする。同じ値でも2行以上は400とし、名前の大小文字だけが異なる場合もC-34に従って重複と判定する。値にカンマを含む場合は、同値の一覧を含めて400とする。複数行の結合、先勝ち・後勝ち、同値の単一化による補正は行わない。不正な数値・桁あふれの400と、Transfer-Encoding併記時の400は既存方針を維持する。検出したエラーはボディの完了を待たず既存のエラー応答経路へ渡し、応答送信完了後に切断する。この決定は要求のContent-Lengthについてであり、他のヘッダーやCGI出力に一律適用しない。Content-LengthとTransfer-Encodingが両方ない場合はC-36に従う。
- **確定(C-34)**: HTTP/1.0・HTTP/1.1要求のヘッダー名はRFC 9110のtoken(ASCII英数字と ! # $ % & ' * + - . ^ _ ` | ~ のいずれか1文字以上)として検証し、ASCII大文字A〜Zを小文字a〜zに変換して保持する。名前の比較は大小文字を区別しない。空の名前・許可外の文字・名前と値を区切るコロンの欠落は補正せず400とする。ヘッダー行は最初のコロンで名前と値に分け、値内のコロンはそのまま扱う。値はC-33の前後SP/HTAB除去を行うが、一律の小文字化はしない。個別ヘッダーの値の検証は既存方針に従う。この決定はHTTP要求のヘッダー名についてのもので、重複はHostのC-28・C-30とContent-LengthのC-35に従い、その他の重複、値の残る検証規則、CGI出力・HTTP_*転送の詳細や具体的なAPIは別途確定する。
- **確定(C-33)**: HTTP/1.0・HTTP/1.1要求のヘッダーは折り返し(obs-fold)を受け付けず、SP/HTABで始まる空でないヘッダー行は400とする。最初のヘッダー行の字下げやSP/HTABだけの行も同様に拒否する。フィールド名とコロンの間の空白・タブは400とし、補正しない。コロン後の値は前後のSP/HTABだけを除去し、値の内部の空白・タブはこの処理では変更しない。`Host:localhost` と `Host: localhost` は同じ値として扱う。ヘッダー終端は完全に空のCRLF行とする。この処理はHTTP要求ヘッダーに限り、ボディには適用しない。フィールドごとの値の検証(HostのC-30等)は別途行い、CGI出力の検証規則は変更しない。
- **確定(C-32)**: HTTP/1.0・HTTP/1.1要求のリクエスト行は末尾CRLF込みで8 KiB(8,192バイト)、その直後のヘッダー部分全体は各行のCRLFと終端の空行込みで32 KiB(32,768バイト)を上限とする。ヘッダー合計にリクエスト行・ボディを含めない。URLデコード等の前の受信バイト数で数え、上限と同じサイズは受理、超過時はそれぞれ414 / 431で拒否する。受信中に検査し、超過が確定したら行末・ヘッダー終端・ボディの到着を待たずエラー応答へ進み、既存方針どおり応答送信完了後に切断する。同じrecvで届いたボディをヘッダーサイズへ加算しない。ヘッダー1行ごとの別上限と個数上限は追加しない。2つの上限はコード内定数とし、Config項目を増やさない。ボディ上限は既存client_max_body_sizeと413、CGI出力上限はCGI-20の規則を維持する。
- **確定(C-31)**: HTTP/1.0・HTTP/1.1要求のリクエスト行とHTTPヘッダーの改行はCRLFだけを受理し、単独LF・不正な単独CRは400とする。受信断片の末尾がCRだけなら、その時点では不正とせず次のバイトを待つ。次がLFなら正常なCRLFとして扱い、LF以外なら不正と判定する。切断やタイムアウトで未完了になった場合は既存の接続処理に従う。この改行制限はボディのデータには適用せず、ボディ中のCR/LFを変更・拒否しない。CGI出力ヘッダーのLF/CRLF両方受理と、HTTP応答ヘッダーのCRLF生成はCGI-15の既決定を維持する。
- **確定(C-30)**: Host値は前後のSP/HTABを取り除いてから、IPv4またはASCIIホスト名と任意の `:port` として検証する。ホスト名はASCII英数字と `-` からなる空でない要素を `.` で区切り、各要素の先頭・末尾は英数字とする。localhostのような単一要素、大文字・小文字、末尾の `.` 1個を受理する。IPv4はドット区切りの4個の十進数で各0〜255。ポートは省略可、明示するなら十進数の1〜65535で空値は拒否する。Host全体の空値、途中の空白・タブ、重複、不正な値、スキーム・パス・ユーザー情報付きの値は400。IPv6の角括弧表記とHost内のパーセント表記は受付対象外として400にする。DNSへの問い合わせや名前の実在確認は行わず、Hostによる仮想ホスト選択も行わない。HTTP/1.0のHost省略可、HTTP/1.1の欠落400、自動補完URLでの使用方法はC-28を維持する。この受付範囲は本課題用の限定仕様であり、RFCのHost構文全体を実装するものではない。
- **確定(C-29)**: C-27の自動補完のLocationを生成するときは、デコード・正規化済みのパスに末尾 `/` を付け、ASCII英数字(A-Z / a-z / 0-9)・`- . _ ~` と区切りの `/` だけをそのまま出力する。それ以外はバイトごとに大文字16進の `%XX` へ変換する。例: 内部パス `/hello world` → `/hello%20world/`、`/a+b` → `/a%2Bb/`、`/name?part` → `/name%3Fpart/`、`/literal%20` → `/literal%2520/`。URLとして既にエンコード済みの文字列へ重ねて適用せず、内部のデコード済みパスから一度生成する。queryは元の表記を保持して後ろに付け、デコードも再エンコードも行わない。例えばHost localhost:8080への `/hello%20world?name=a+b` は `http://localhost:8080/hello%20world/?name=a+b` へ案内する。Configのreturnの固定URLはこの変換の対象にしない。
- **確定(C-28)**: C-27の自動補完のLocationは `http://` で始まる絶対URLとする。有効なHostがあればホスト名と明示ポートを使い、HostにポートがなければURLにも追加しない(HTTPの既定80番)。HTTP/1.0でHostがない場合だけ、acceptした接続のサーバー側IP・ポートをgetsocknameで取得して使う。listenのワイルドカード値0.0.0.0やクライアント側IP・ポートを代用しない。HTTP/1.0・1.1ともHostの重複・不正は400、HTTP/1.1のHost欠落も400。tasugiyaが接続のサーバー側IP・ポートを取得して渡し、rysatoがHostの検証とURL組み立てを担当する。Forwarded / X-Forwarded-*を参照せず、ConfigのreturnはC-21の固定URLを使う。パスの再エンコードはC-29に従う。Hostの受付範囲はC-30に従う。接続情報の受け渡しAPI・取得失敗時の扱いは別途確定する。
- **確定(C-27)**: GETでlocationを選択し許可メソッドを確認した後、選択した配信locationのroot/aliasで解決した対象が実ディレクトリで、正規化後のURLパスが `/` で終わらなければ、末尾 `/` 付きURLへ301を返す。queryは保持し、例えば `/docs?lang=ja` の移動先のパスとqueryは `/docs/?lang=ja` とする。末尾 `/` 付きの要求でC-15のindex→未発見ならautoindexの処理へ進む。POST・DELETEにはこの自動補完を適用せず、各メソッドの処理規則に従う。returnがあるlocationはC-21・C-22の指定を使う。location選択は変更せず、`location /docs/` を `/docs` に一致させる特別扱いは追加しない。Locationのホスト名・ポートの出所はC-28に従う。パスの再エンコードとqueryの保持はC-29に従う。
- **確定(C-26)**: 公開・CGI・アップロード領域は二人が管理し、配下にsymlinkを置かない運用前提とする。Webservにはsymlinkの検出・拒否機能を実装せず、関連するConfig項目も増やさない。symlinkが置かれた場合はOSがリンク先を辿り得るため、公開領域外へのアクセスを完全に遮断する仕様とはしない。URL由来のパスにはC-24・C-25のデコード・正規化を適用し、その処理だけでsymlink経由の領域外アクセスを防げるとは扱わない。ファイルの存在・種別・アクセス失敗の確認、root/aliasのパス結合、アップロード名の扱い等の残る実装規則は別途確定する。
- **確定(C-25)**: C-24で1回デコードしたURLパスをlocation選択前に正規化する。連続する `/` は1個にまとめ、`/` で区切った要素全体が `.` なら除去、`..` なら直前の要素を1個取り除く。URLの先頭の `/` より上へ戻ろうとした時点で400とする。`file..txt`・`.hidden` 等は変更しない。末尾 `/` と、末尾 `/.`・`/..` が表すディレクトリの末尾 `/` を保つ。例: `/a//b.txt`・`/a/./b.txt`・`/a/x/../b.txt` は `/a/b.txt`、`/a/../b.txt` は `/b.txt`、`/a/.`・`/a/b/..` は `/a/`、`/a/..` は `/`。`/../secret.txt`・`/a/../../secret.txt` は400。`/a/%2e%2e/b.txt` はデコード後に `/b.txt` としてlocationを選ぶ。この処理はURL文字列の整理であり、symlinkの扱いと公開範囲の保証の制限はC-26に従う。ファイルパスの結合・検証の残る詳細は別途確定する。末尾 `/` のない実ディレクトリへのGETはC-27に従う。
- **確定(C-24)**: 最初のリテラルの `?` でパスとqueryを分離し、パスの `%XX` を1回だけデコードする。その後にパスを正規化してからlocationを選ぶ。queryは未デコードのままCGIのQUERY_STRINGへ渡す。パスでは `%20` は空白、`%2F` / `%2f` は区切りの `/` となり、`+` は空白へ変換しない。`%252e` は `%2e` までとし、HTTP/Router/CGIへの受け渡しで二重デコードしない。復号して生じた `?` をqueryの区切りとして再解釈しない。パスにある `%` / `%2` / `%GG` 等の不正なパーセント表記と `%00` (NUL) は400とする。この400規則はパスについての合意であり、queryの受付検証を新たに定めるものではない。`.` / `..` / 連続スラッシュの正規化はC-25に従い、symlinkの運用前提と制限はC-26に従う。その他の不正パスとファイルパスの結合・検証の詳細は別途確定する。
- **確定(C-23)**: queryを除いたURLパスに対する文字列前方一致とし、一致する最も長いprefixを選ぶ。設定順には依存せず、パス要素の境界は一致条件にしない。`/a` は `/a`・`/a/file.txt`・`/abc` に一致する。ディレクトリ配下を指定するには `/a/` と書き、これは `/a/file.txt` には一致するが `/a`・`/abc` には一致しない。`/`・`/a`・`/a/b/` があれば、`/a/b/file.txt` には `/a/b/`、`/abc` には `/a` を選ぶ。URIの分離・デコード・処理順序はC-24で確定。正規化はC-25で確定。末尾スラッシュ補完リダイレクトはC-27に従う。
- **確定(C-38)**: Configのlocation prefixは先頭`/`を必須とし、ASCII英数字(A-Z / a-z / 0-9)と `/ - . _ ~` だけを受理する。連続する`//`、パス要素全体が`.`または`..`の指定は設定エラーで起動を中止する。queryや%エンコード、非ASCII文字など許可外の文字を受け付けず、設定値をデコード・正規化して救済しない。`/`・`/images/`・`/cgi-bin/`・`/v1.0/`・`/.hidden/`・`/file..txt`は受理する。prefixの比較はOSによらず大小文字を区別し、`/Images/`と`/images/`は別のprefixとする。`location /`は推奨するが必須ではなく、一致するlocationがなければ404。各serverにlocationを1個以上要求するC-37は維持する。alias使用時のprefix末尾`/`必須(C-09)も維持し、root使用時は末尾`/`を必須にしない。この文字制限はConfigのprefixだけに適用し、HTTP要求URLやファイル名の受付範囲には一律適用しない。要求側はC-24・C-25のデコード・正規化後にC-23の最長前方一致で照合する。
- **確定(C-38)**: locationの設定値の受付は上記に従う。要求URIの分離・デコード・正規化はHTTP/Router側の責務とし、処理順序はC-24に従う。正規化はC-25に従い、symlinkの運用前提と公開範囲の保証の制限はC-26に従う。ファイルパスの結合・検証の残る詳細は別途合意する。
- `root` は設定ディレクトリにURLパス全体を追加する。`alias` は一致したlocationのprefixを設定ディレクトリへ置き換える。
- **確定(C-09)**: aliasはディレクトリ対応だけを扱い、**locationのprefixとaliasの値の両方を `/` で終える**。どちらかの末尾 `/` が欠けていたら、自動補完せず設定エラーとして起動を中止する。正規表現や単一ファイルへのaliasは扱わない。この末尾制限はaliasを使うlocationに適用し、rootを使うlocationには追加しない。要求URLの末尾 `/` 補完リダイレクトはC-27に従う。

| 設定 | 要求パス | 探すファイル |
|---|---|---|
| `location /images/` + `root /data;` | `/images/a.png` | `/data/images/a.png` |
| `location /images/` + `alias /data/pictures/;` | `/images/a.png` | `/data/pictures/a.png` |
| `location /kapouet/` + `alias /tmp/www/;` | `/kapouet/pouic/toto/pouet` | `/tmp/www/pouic/toto/pouet` |

最後の例で課題PDFのprefix置換を満たす。`root` にaliasの意味を持たせない。`/images/` は `/images` に一致しない。C-27は選択済みの配信locationで実ディレクトリと判定できたGETだけに適用し、別のlocationを末尾 `/` の補完によって探索する処理は追加しない。

- **確定**: ファイルシステムの相対パスは**設定ファイルを置いたディレクトリ基準**。例えば `conf/default.conf` 内の `root ../www/site1;` は `www/site1` を指す。URL-prefix・returnのURL・indexのファイル名にはこの基準を適用しない。
- **確定(C-40)**: 起動時は、配信locationのroot/aliasと有効なupload_storeについて、パス解決・既定値補完後にstatで存在とディレクトリ種別を確認し、確認失敗・非ディレクトリなら設定エラーで起動を中止する。root省略時のhtmlにも適用し、conf/default.confならconf/htmlが必要になる。ディレクトリや親ディレクトリは自動作成しない。ディレクトリの読み取り・書き込み等の権限を追加のaccess検査で事前確認せず、実際の配信・一覧取得・保存時に失敗を処理する。有効なCGI設定のPythonはstatで存在する通常ファイルであること、access(path, X_OK)の事前確認が成功することを要求し、失敗したら起動エラーとする。起動時にPythonを試験実行しない。indexの存在は起動条件にせず、error_pageの不在・読み取り不可も起動エラーにしない。これらは要求時に既存のindex/autoindex・内蔵エラーページの規則で処理する。個々のHTML・画像・CGIスクリプトを起動時に全件検査しない。リダイレクト専用locationはC-22に従い配信設定を補完・検証しない。起動時検査は実行時の成功を保証するものではなく、ファイル消失・権限変更やexecve等の失敗は引き続き要求時に処理する。symlinkの運用前提と検出機能を省くC-26は維持する。
- 起動後のファイル消失や権限変更は各要求のエラーとして処理する。起動時の検証だけで実行時の確認を省略しない。

### 設定例

以下は、Config・CGIの確定事項に沿った例。実装用設定と対応するデモファイルは別途用意する。以下の例は既定値も一部明示しているが、NGINX既定値のある項目の記述を必須とする意味ではない。

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

### tasugiyaとrysatoの間の契約と検証

**確定(C-42)**: ConfigはConfig / ServerConfig / LocationConfigの3つの読み取り専用データクラスで保持する。rysatoのConfigParserが設定ファイルを読み、構文解析・既定値補完・設定ファイル基準のパス解決・C-40/C-41等の検証まで完了してから、Configを値で返す。tasugiyaは完成した値を使用し、設定文字列の再解析や既定値の再計算を行わない。

| 保持先 | データ | 型・表現 |
|---|---|---|
| Config | servers | std::vector<ServerConfig> |
| ServerConfig | host / port / clientMaxBodySize | std::string / int / std::size_t |
| ServerConfig | errorPages / locations | std::map<int, std::string> / std::vector<LocationConfig> |
| LocationConfig | path / allowedMethods | std::string / std::set<std::string> |
| LocationConfig | kind / pathMode | enum Kind { SERVE, REDIRECT } / enum PathMode { ROOT, ALIAS } |
| LocationConfig | basePath / index / autoindex | std::string / std::string / bool |
| LocationConfig | uploadPath | std::string。空文字列は無効 |
| LocationConfig | cgiExtension / cgiInterpreter | std::stringを2つ。1組だけ保持し、無効なら両方空。mapは使用しない |
| LocationConfig | redirectCode / redirectUrl | int / std::string。REDIRECTのとき使用 |

- ポートとステータスコードはintに変換する際に各既決定の範囲を検証する。ボディ上限はstd::size_tで保持し、範囲を超える値は設定エラーとする(C-13)。文字列から数値への変換で桁あふれを起こしてから判定しない。
- 設定値はconstのgetterで公開し、公開setterは設けない。文字列・コンテナはconst参照、int / std::size_t / bool / enumは値で取得する。getterの名前はgetServers / getHost / getPort / getClientMaxBodySize / getErrorPages / getLocations / getPath / getAllowedMethods / getKind / getPathMode / getBasePath / getIndex / getAutoindex / getUploadPath / getCgiExtension / getCgiInterpreter / getRedirectCode / getRedirectUrlとする。
- kindを確認して該当する設定だけを使う。REDIRECTでは配信項目を使用せず、配信既定値の補完・配信パス検証もしない(C-22)。解析中は明示の有無を保持して重複や禁止された併記を検出し、検証後の設定だけを公開する。
- 設定ファイルの読み込み・構文解析・検証の失敗はstd::runtime_errorで理由を伝える。mainが捕捉して理由を表示し、listenを始めず異常終了する。ConfigParser内部でプロセスを終了させず、不完全なConfigを返さない。これは起動時のみ例外を許可する既決定に従う。
- mainがconst Configを保持し、EventLoopはconst Config&、ServerSocket/Clientは対応するconst ServerConfig&またはconst ServerConfig*を参照する。設定の所有権は移さず、参照先をdeleteしない。Configとそのvectorの要素は起動後に変更せず、EventLoop・Client等を先に終了/破棄し、Configを最後まで生存させる。
- tasugiyaはServerConfigごとにlistenし、accept元に対応するServerConfigをClientに紐付ける。body上限は最初のappendDataより前にHttpRequestへ渡す。bind等の起動失敗では作成済みfdも閉じる。
- エラーメッセージには設定項目名と、分かる場合はserverのlisten先またはlocationを含める。行番号は実装しない既決定を維持する。

公開APIの中心は次のとおり。メンバー構築用のprivate関数等は実装詳細とする。ConfigParserは1クラスで、字句解析と構文解析をprivate関数に分ける既決定を維持する。

```cpp
class ConfigParser {
public:
    Config parse(const std::string& configPath);
};

// Configのgetter
// const std::vector<ServerConfig>& getServers() const;
// ServerConfigのgetterの例
// const std::string& getHost() const;
// int getPort() const;
// std::size_t getClientMaxBodySize() const;
// const std::vector<LocationConfig>& getLocations() const;
```

```cpp
// main内の起動処理。呼び出し元でstd::runtime_errorを捕捉する。
ConfigParser parser;
const Config config = parser.parse(configPath);
EventLoop loop(config);
loop.run();
// スコープ終了ではloopが先、configが後に破棄される。
```

二人で共有する確認例は、既定値補完後の型・値、C-40/C-41の検証、不正設定でruntime_errorとなりlistenへ進まないこと、upload/CGI無効時の空文字列、getterが書き換え可能な参照を公開しないこと、Configの寿命が参照元より長いこと。HTTP/Router/CGI間の処理結果のAPIはC-42とは別の未決定事項として維持する。

### 合意状況と残る確認事項

Configについて整理した6項目は、文法(C-37)・location設定値(C-38)・listen省略値(C-39)・起動時ファイル検証(C-40)・設定の組み合わせ(C-41)・型とAPI(C-42)のすべてを合意済み。以下は決定事項のチェック表であり、実装やテストが完了したことを意味しない。HTTP要求の受付・CGI出力・アップロード等の処理側の未決定事項は別途残る。

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
| [x] | 文法の細則(C-37): serverを1個以上、各serverにlocationを1個以上。ブロック後の`;`を拒否、設定名は大小区別、同じ階層の記述順に依存しない。特殊locationとif/rewrite/try_filesを拒否し、空locationは既定値で受理する |
| [x] | location設定値(C-38): /始まり、ASCII英数字と/-._~のみ。//と単独./..要素を拒否し、補正しない。大小文字を区別。location /は任意でマッチなし404。aliasの末尾/必須を維持し、要求URIの受付文字は制限しない |
| [x] | 省略された設定にはNGINXの既定値を適用し、明示offを強制しない(C-05)。継承なし・相対パスの基準は維持する。listenのみC-39の固定値を優先する |
| [x] | allow_methods省略時はGETのみ許可する(C-06)。POST・DELETEは必要なlocationで明示する。CGI用locationへのDELETEは引き続き405 |
| [x] | upload_store / cgi_extension省略時は無効(C-07)。保存先・拡張子・Pythonのパスを自動補完しない |
| [x] | listen省略時は起動権限によらず0.0.0.0:8000(C-39)。権限判定なし。明示値を優先し、補完後の重複・競合やbind失敗は起動エラー。自動ポート変更なし |
| [x] | returnがあればリダイレクト専用(C-22)。allow_methodsのみ併記可、配信設定はoffでも併記を拒否。配信項目の既定値補完・パス検証は省き、許可メソッドを確認してからリダイレクトする |
| [x] | 同じlocationでroot/aliasを併記した場合は設定エラーで起動を中止する(C-08)。記述順で上書きしない |
| [x] | aliasはディレクトリ対応に限定し、prefixと値の末尾 `/` を必須とする(C-09)。欠けていれば補完せず起動エラー |
| [x] | 明示するlistenはIPv4:portのみ(C-10)。IPv4の各数値は0〜255、ポートは1〜65535。その他の書式・追加オプション・不正値は起動エラー |
| [x] | listenの明示は1serverに最大1回(C-11)。同じ値でも2回以上は起動エラー。省略時は既定値を使い、複数ポートはserverを分ける |
| [x] | server間の同一IPv4:portと同じポートでの0.0.0.0/特定IPの競合は、既定値補完後に検証し起動エラーとする。server_name・Hostによる仮想ホスト選択は省く(C-12) |
| [x] | client_max_body_sizeは正の十進整数・バイト単位のみ(C-13)。0による無制限・符号・小数・単位接尾辞・桁あふれは起動エラー。省略時は1 MiB |
| [x] | 通常の設定項目は同じブロック内に最大1回とし、同じ値でも重複は起動エラー(C-14)。error_pageは同じserver内の同一コード、locationは同じserver内の同一prefixの重複を拒否する |
| [x] | indexはファイル名1個のみ(C-15)。複数候補・`/`を含むパス・`.` / `..` は起動エラー。存在は起動条件にせず、ディレクトリ要求でindexがなければautoindexがonなら一覧、offなら403 |
| [x] | autoindexは小文字on/offの1引数のみ(C-16)。省略時off、HTML固定。その他の値・引数数は起動エラー。形式・サイズ表記・時刻表示等の追加Configは作らない |
| [x] | allow_methodsは大文字GET/POST/DELETEを空白区切りで重複なく1個以上(C-17)。空・重複・小文字・カンマ区切り・その他の値は起動エラー。記述順に意味はなく、GET等を自動追加しない |
| [x] | upload_storeはoffまたは保存先パス1個(C-18)。保存先は起動時に存在するディレクトリを要求し、自動作成しない。GET公開先はroot/aliasで別途指定する |
| [x] | cgi_extensionは小文字offまたは「拡張子＋Python絶対パス」1組(C-19)。拡張子は`.`＋ASCII英数字1文字以上・大小区別。不正な拡張子・引数数・相対パスは起動エラー |
| [x] | locationは文字列の最長前方一致とし、設定順・パス要素の境界を一致条件にしない(C-23)。/aは/abcにも一致し、ディレクトリ配下には/a/を使う |
| [x] | 最初の?でqueryを分離→パスを1回デコード→正規化→location選択(C-24)。queryは未デコードでCGIへ渡す。パスの%2F/%2fは/、+はそのまま。不正な%表記・%00は400 |
| [x] | 連続/・.・..をlocation選択前に正規化する(C-25)。先頭/より上への移動は400、末尾のディレクトリを示す/は維持。symlinkの運用前提と制限はC-26に従う |
| [x] | 公開・CGI・アップロード領域にsymlinkを置かない運用前提とし、検出・拒否機能とConfig項目を省く(C-26)。symlinkがあればOSが辿り得るため、公開領域外へのアクセスを完全に遮断する仕様とはしない |
| [x] | 実ディレクトリへのGETは末尾/なしならqueryを保持して301、/付きでindex/autoindexへ進む(C-27)。メソッド確認後に適用し、POST・DELETEとConfigのreturnは各規則に従う。Locationのホスト・ポートはC-28、パスの再エンコードとquery保持はC-29 |
| [x] | 自動補完のLocationはhttp絶対URL、有効Hostを明示ポートごと使い、ポート省略時は追加しない(C-28)。1.0でHostなしなら接続のサーバー側IP/port。Hostの重複・不正は両版400、1.1は欠落も400。転送ヘッダーは使わない |
| [x] | 自動補完URLは正規化済みパスのASCII英数字・-._~・区切り/以外をバイト単位の大文字%XXへ戻し、queryは元のまま保持する(C-29)。Configのreturnは変換しない |
| [x] | Hostの受付はIPv4またはASCIIホスト名と任意portに限定する(C-30)。前後SP/HTABを除去し、名前の各要素と末尾.、IPv4各0〜255・port1〜65535を検証。重複・空値・不正値・IPv6・%表記は400。DNS照会なし |
| [x] | HTTP要求行・ヘッダーはCRLF限定(C-31)。単独LF・不正な単独CRは400。断片末尾のCRは続きを待ち、ボディの改行とCGI出力の規則は維持する |
| [x] | 要求行8,192バイト・ヘッダー合計32,768バイト(C-32)。CRLFとヘッダー終端空行を数え、要求行とボディはヘッダー合計から除外。超過時414/431、コード内定数。行別・個数の追加上限なし |
| [x] | HTTP要求ヘッダーの折り返し・行頭SP/HTAB・コロン直前の空白/タブは400、値の前後SP/HTABだけを除去する(C-33)。空白だけの行は終端にせず、空のCRLF行だけを終端にする。ボディは変更しない |
| [x] | HTTP要求ヘッダー名はRFCのtokenを受理してASCII小文字で保持する(C-34)。空の名前・不正文字・コロンなしは400。値は一律に小文字化しない |
| [x] | 要求のContent-Lengthは最大1行とし、同値・名前の大小文字違いを含む重複、カンマ入りの値は400とする(C-35)。結合・補正せず、他のヘッダーやCGI出力へ一律適用しない |
| [x] | Content-LengthとTransfer-Encodingが両方ない要求はPOSTも空ボディとしてヘッダー終端で受信完了し、省略だけで411にしない(C-36)。CGIに渡す入力は空でEOFを通知する |
| [x] | returnはlocation内の301/302＋ホスト部分を持つ固定HTTP(S)絶対URLの2引数のみ(C-21)。相対URL・他コード・不正な値/引数数は起動エラー。変数・元URI/queryの自動追加・本文指定は省く |
| [x] | error_pageはserver内に100〜599の3桁コード1個＋ファイルパス1個(C-20)。複数コード・ステータス変更・不正値/引数数は起動エラー。本文禁止のHTTP規則は維持する |
| [x] | error_pageの受付範囲は100〜599、本文差し替えの適用は400〜599のみ(C-20追記)。100〜399の指定は受理するが応答に適用しない |
| [x] | 起動時のファイル検証(C-40): root/alias/upload先はstatで存在するディレクトリ、Pythonは通常ファイルとaccess(X_OK)を確認。既定rootにも適用し、自動作成・配下全件検査・ディレクトリの追加権限検査は省く。index/error_pageは要求時、redirect専用locationの配信パスは検証対象外 |
| [x] | 設定の組み合わせ(C-41): 配信POSTにはupload/CGIのどちらか1つを要求し、両方有効・CGI有効でDELETE許可は起動エラー。省略/offは無効。returnのPOSTは処理先不要。機能有効でもPOSTを自動許可せず、POST不許可のupload/CGI設定は受理する |
| [x] | Configの型とAPI(C-42): parse(path)が検証済みConfigを値で返し、設定エラーはruntime_errorをmainで処理する。ボディ上限size_t、CGIは文字列2つの1組、upload/CGI無効は空文字列。getterのみ公開し、mainのconst Configをconst参照/ポインタで利用。参照元を先に破棄する |

アップロード形式・ファイル名・上書き、ファイルパスの結合・検証の残る詳細はHTTP側の別議題として残す。symlinkの運用前提と制限はC-26で確定済み。URIの分離・デコード・正規化はC-24・C-25で確定済み。CGI出力の基本動作は確定済みで、ヘッダー検証等の残る詳細は [03のC12](reports/03_config_cgi_upload.md) に記録する。ConfigParserにそれらの処理を持たせない。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト

- [RFC 9112 §5.1–5.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-5.1): コロン直前の空白拒否、値の前後OWSの除去、obs-foldの扱い。C-33では折り返しを補正せず400にする。
- [RFC 9110 §5.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1)、[§5.6.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.2): ヘッダー名の大小無視とtokenの文字集合。C-34では名前をASCII小文字に揃えて保持する。
- [RFC 9110 §8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6): 同値を繰り返したContent-Lengthは拒否または単一値への補正が可能。C-35では拒否を選択する。
- [RFC 9112 §6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3)、[RFC 1945 §7.2](https://www.rfc-editor.org/rfc/rfc1945.html#section-7.2): 要求ボディの長さ判定。C-36では長さ指定が両方ない要求を空ボディとして扱う。

---

# クラス設計(全13クラス+2表)

Configの型・parse API・getter・所有関係はC-42で確定済み。以下のHTTP/Router/CGI等の具体的なシグネチャ・型・所有権とCGI内の担当境界は設計案であり、上記の確定済み動作を満たす形で二人が合意する。モジュール一覧の `CgiHandler` はCGI処理の総称で、この案では `CgiExecutor` / `CgiProcess` 等に分ける。

## 一覧と責務

### 設定ドメイン(rysato担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ConfigParser` | ファイル→トークン化→構文解析→既定値補完/パス解決/検証。Configを値で返し、設定エラーはruntime_errorをmainへ伝える(C-42) | 起動時1回・使い捨て |
| `Config` | パース結果の最上位入れ物 | 起動時1回・終了まで不変 |
| `ServerConfig` | server ブロック1個分のデータ | 同上 |
| `LocationConfig` | location ブロック1個分のデータ | 同上 |

※ Config 系3クラスは読み取り専用の構造体(ロジックは持たない)。

### ネットワーク/イベントドメイン(tasugiya担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ServerSocket` | socket→setsockopt→bind→listen→ノンブロッキング化(listen先のhost:port 1組分) | 起動時・listen先数分 |
| `EventLoop` | poll 本体。fd 登録簿、イベント配送、タイムアウト見回り、終了時の後始末 | 起動時1個 |
| `Client` | 接続1本の状態機械(バッファ・状態・最終活動時刻) | accept 時に生成 / close 時に破棄 |
| `CgiProcess` | 実行中 CGI 1本の状態(pid・パイプfd・バッファ・開始時刻・EOF/終了の確認・失敗理由) | CGI開始時に生成 / 出力の処理・子の回収・結果受け渡し・後始末が済んだ後に破棄 |

### HTTPドメイン(rysato担当)

| クラス | 責務 |
|---|---|
| `HttpRequest` | HTTP/1.0・HTTP/1.1の逐次パース(断片投入→状態遷移→完成判定)。受信バージョン別の検証、ヘッダー段階の拒否、1.1のchunkedデコードを含む |
| `HttpResponse` | HTTP/1.0レスポンス組み立て+`serialize()`+エラーページ工場 |
| `Router` | Request+Config → 静的/autoindex/アップロード/DELETE/リダイレクト/CGI/エラーの振り分け |

### 共通(共同)

| クラス/表 | 責務 |
|---|---|
| `CgiExecutor` | pipe/fork/dup2/chdir/execveによる起動と環境変数の受け渡し。tasugiyaがプロセス制御、rysatoが環境変数の内容を担当する案。具体的な境界・APIは03のC12で確定する |
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

    // Client(tasugiya側)が呼ぶ
    void   appendData(const char* data, size_t len);  // recv 断片を投入
    State  getState() const;
    int    getErrorCode() const;      // ERROR 時: 400, 413, 417, 501 など
    void   setMaxBodySize(size_t n);  // Config の値を注入(早期413判定)

    // Router(rysato側)が COMPLETE 後に呼ぶ
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

### Config 系(読み取り専用・C-42確定)

型・getter一覧・エラー処理・寿命は上記「tasugiyaとrysatoの間の契約と検証」に従う。CGIの複数組mapを使う旧案は廃止し、cgiExtension / cgiInterpreterの文字列2つを使う。

```cpp
class ConfigParser {
public:
    Config parse(const std::string& configPath);
};
// Config: const std::vector<ServerConfig>& getServers() const;
// ServerConfig: host(string), port(int), clientMaxBodySize(std::size_t),
//               errorPages(map<int,string>), locations(vector<LocationConfig>)
// LocationConfig: path, allowedMethods(set<string>), kind(SERVE/REDIRECT),
//                 pathMode(ROOT/ALIAS), basePath, index, autoindex(bool),
//                 uploadPath, cgiExtension, cgiInterpreter,
//                 redirectCode(int), redirectUrl
// 公開setterなし。文字列/コンテナのgetterはconst参照を返す。
```

最長前方一致・マッチなし404は確定済み。以前のfindLocationヘルパーの配置・具体的シグネチャはRouter側APIの設計に属し、Configの公開データの型やparse APIを未確定に戻すものではない。

### EventLoop / Client / CgiProcess(tasugiya側内部)

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
    // 所属 const ServerConfig* を参照(所有しない、C-42)
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
