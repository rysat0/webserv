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

## 2. 全体ルール(違反は0点につながる)

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

## 4. 使用可能な外部関数とC++98標準ライブラリ

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

C++98標準ライブラリと、上に列挙されたOS/POSIX等の外部関数を区別する。標準ライブラリは課題の他の制約を守る範囲で使用可能とし、一覧にないという理由だけで一律に禁止しない。`std::`を付けた自作関数・非標準関数が許可される意味ではない。

2026-10-07のユーザー決定により、`<cstdio>`のファイル削除用`int std::remove(const char*)`を使用可能とし、DELETEとHTTP-43の途中ファイル削除に採用する。以前の使用不可という判断をこの決定で更新する。`unlink`/`rmdir`を直接使う許可へは広げない。`_exit`・`clock_gettime`の使用不可という既決定は維持する。Date生成の暦日時取得手段はHTTP-56の別の設計判断として扱う。

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

担当者の対応は **A = tasugiya、B = rysato** として確定。以降の担当表・設計案は名前で表記する。「共同」はtasugiyaとrysatoの両名を指す。CGI内の担当境界・受け渡しAPIはCGI-91〜98で確定済み。CGIの所有権・後始末・エラー適用境界も合意済み。9/10の起動・計時は動作確認で見直し得る代替案とする。

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

※ 初回決定日は2026-08-14。Configの合意完了日は2026-10-05。2026-10-07に最新のCGI合意を反映し、資料間の記述を整理した。以下の表は更新後の決定を示す。課題の要求とチームが採用した範囲・初期値は区別する。

| 項目 | 決定 |
|---|---|
| ディレクティブ名 | NGINX風+独自名の混在(NGINXにある概念は同名、独自機能は独自名) |
| デフォルト値 | **省略された設定には原則NGINXの既定値を適用し、listenのみC-39の0.0.0.0:8000を優先する**。明示offを強制しない。独自項目を含む既定値は05のConfig合意事項を参照 |
| 文法エラー時 | 警告メッセージを表示して起動せず終了(行番号表示はしない) |
| HttpRequest | 逐次パース型: `appendData(chunk)` + 状態問い合わせ方式 |
| HttpResponse | `serialize()` で一括生成 → 送信バッファへ |
| CGI | 必須部分はPythonのみ。`cgi_extension .py /usr/bin/python3` 形式で拡張子→インタプリタ対応。RFC 3875を参考に、下記CGI節の範囲に限定する |
| ログ | レベル付き Logger(`[LEVEL] message`、HTTP-73)、ファイル出力なし、レベルで絞り込み可 |
| 継承 | **継承なし**。各locationの明示値と、その項目の既定値で設定を完成させる。省略時の既定値補完と上位からの継承は区別する |
| location マッチ | **文字列の最長前方一致**(C-23)。設定順に依存せず、`/a`は`/abc`にも一致する |
| タイムアウト | 要求受信はacceptから60秒、応答送信は送信バッファ準備から60秒、送信後の切断待ちは1秒。CGIは起動から10秒で504。途中のI/Oで延長しない(HTTP-67〜70) |
| 接続方針 | **1接続1要求**。`Connection: close` を付け、最終応答の全バイトを送信してから切断する |
| keep-alive / pipelining | **今回の実装対象外**。同じ接続で次の要求を処理しない |
| HTTP応答バージョン | **HTTP/1.0** 固定。課題要件に必要な機能を実装する |
| HTTP受信バージョン | **HTTP/1.0・HTTP/1.1**。HTTP/1.2〜1.9は1.1の規則で処理し、受信表記を保持する(HTTP-04)。RFC全文への準拠は範囲外 |
| chunked | **HTTP/1.1リクエストの復号は実装する**。chunkedレスポンスは生成しない |
| ステータスコード | 主要コードを網羅的に実装(基本全部) |
| デフォルト設定パス | `./conf/default.conf`(引数なし起動時に読む) |
| シグナル処理 | `SIGPIPE` は `SIG_IGN` で無視(send失敗時はerrnoを見ず接続close)。`SIGINT` はフラグ方式でグレースフル終了(全fd close・全メモリ解放してから終了) |
| CGI環境変数 | CGI-27〜50・75〜83に従う。変数の一覧・設定値・転送規則は本書のCGI決定表、受け渡しAPIは03を参照 |
| ConfigParser構成 | 1クラス(トークナイザとパーサーはprivate関数で分離) |
| CGI状態の持ち方 | 独立クラス `CgiProcess`(fd→オブジェクトの対応表で引ける形) |
| Router構成 | 1クラス+private関数分割(`handleGet/handlePost/handleDelete/handleCgi`...) |
| pollfd配列管理 | 毎周再構築(Client/CGI一覧から配列と監視フラグを毎回計算) |
| 例外方針 | 起動フェーズのみ例外可・ループ内は戻り値方式。保険としてループに `catch(std::exception&)` の防波堤(ログ+接続close)。bad_alloc もここで受ける |

<a id="http-agreements"></a>

## HTTP対応範囲

課題PDF Version 24.1の印刷ページ7はHTTP/1.0を参考基準として推奨し、RFC全文の実装は要求していない。一方、印刷ページ8〜11のGET / POST / DELETE、アップロード、ブラウザ互換、設定機能、CGI、chunked復号などはHTTPバージョンにかかわらず満たす。

<a id="http-request-line"></a>

### 要求行の合意事項(HTTP 1/10・2026-10-07)

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-01 | 要求行の書式 | method SP request-target SP HTTP-version CRLFを厳密に読む。区切りは半角SP各1個とし、タブ・余分な空白・行頭/行末の空白・要素欠落・先行する空行は400。補正・読み飛ばしは行わない。CRLF限定とCRLF込み8 KiB上限・超過414はC-31・32を維持する |
| HTTP-02 | メソッドの受付 | 空でないRFC tokenとして検証し、不正文字は400。実装は大文字GET/POST/DELETEのみ。基本書式が正常な未実装メソッド(HEAD/PUT/UNKNOWN/get等)は501とし、大小文字を補正しない。対応メソッドがlocationで禁止されていれば既決定の405/Allow |
| HTTP-03 | request-targetの形式 | 対応メソッドの要求先は/始まりのorigin-form(パスと任意query)に限定する。絶対URL・authority-form・asterisk-form等は400。絶対URL受付はRFC 9112では必須だが、本課題向けの対応範囲から外すチーム方針。未実装メソッドの基本書式が正常な場合の501と区別する |
| HTTP-04 | バージョンの検査 | 大文字HTTP/と1桁の十進数字.1桁の十進数字の書式を検証し、不正は400。HTTP/1.0・1.1を受理する。対応しないメジャー(HTTP/2.0等の表記)は505。HTTP/1.2〜1.9は1.1と同じ要求検証・ボディ終端・Host/Expect等の規則で扱い、受信した版の表記は保持する。応答はHTTP/1.0固定、CGIのSERVER_PROTOCOLは受信表記を使う |
| HTTP-05 | 要求先の文字検査 | origin-formのパス・queryはRFC 9112 §3.2.1・RFC 3986 §3.3〜3.4の文字集合で検査する。生の空白・制御文字・DEL・非ASCII・#等の許可外文字、不正な%XXは400。queryは構文を検査するだけでデコードせず保持する。パスの1回デコード・正規化・%00拒否はC-24・25を維持する |

パス内の通常文字はASCII英数字、-._~、!$&'()*+,;=:@と区切り/。queryにはさらに?を認める。%は必ず16進数字2桁とし、大文字・小文字とも受理する。この検査はデコード前の要求先に対して行い、デコード後のファイル名の文字制限へ転用しない。連続/の正規化はC-25、/を持たない絶対URLの補完などは対象外とする。

RFC上の要求と本チームの限定仕様は区別する。要求行前の空行を読み飛ばさない点はRFC 9112 §2.2の推奨と異なり、absolute-formの不受理は同§3.2.2の必須規則を対応範囲から外す判断。課題PDFはRFC全文実装を要求しておらず、これらを課題から個別に免除されたとは断定しない。

根拠: [課題PDF](webserv.pdf)、[RFC 9112 §2.2〜3.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-2.2)、[RFC 9110 §6.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-6.2)・[§9.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.1)、[RFC 3986 §3.3〜3.4](https://www.rfc-editor.org/rfc/rfc3986.html#section-3.3)。実装・受入試験の完了は示さない。

<a id="http-input-headers"></a>

### 入力ヘッダーの合意事項(HTTP 2/10・2026-10-07)

要求行・名前・改行・空白・制御文字・32 KiB上限、Host、Expect、CGI転送の既決定を維持する。以下は残る入力ヘッダー規則の正本。HTTP/1.2〜1.9には1.1と同じ規則を適用する(HTTP-04)。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-06 | Content-Lengthの値 | 前後SP/HTAB除去後、ASCII数字1文字以上を受理する。0・0012等の先頭0を受理し、空値・符号・小数・途中の空白・非数字・桁あふれは400。最大1行、同値重複・カンマ入り400、Transfer-Encoding併記400、正常な値のボディ上限超過413は維持する |
| HTTP-07 | Content-Typeの共通検証 | CGI以外も最大1行に統一し、同値・空値を含む重複は400。単一値は共通の文字検査・前後SP/HTAB除去後に保持し、HTTP共通処理でメディアタイプ全体の検証・登録一覧照合・本文の推測は追加しない。CGIでは空値をCONTENT_TYPEの空文字列、欠落を省略とするCGI-80を維持する。アップロード固有の形式・boundary検証はHTTP-33〜37に従う |
| HTTP-08 | Transfer-Encodingの指定回数 | 最大1行とし、同値・空値を含む重複は400。RFCでは複数行の一覧が許容されるが、本課題向けの対応範囲として限定する。HTTP/1.0での指定とContent-Length併記の400は維持する |
| HTTP-09 | Transfer-Encodingの値 | 符号化の一覧としてRFC構文を検査し、名前はASCIIの大小文字を区別しない。空要素は無視し、有効な符号化が残らない値・不正構文・chunkedの複数回指定・chunkedのヘッダーパラメーターは400。最後がchunkedでない要求は400。構文が正しくchunkedで終わるが未対応の符号化を含む場合は501(gzip, chunked等)。受理・復号するのは有効な符号化が単独chunkedの要求だけ |
| HTTP-10 | Connectionの共通検証 | CGI-78の検証を通常要求にも適用する。各行をカンマ区切りのtokenとして検査し、不正な非空要素は400、空要素は無視、複数行は全行を扱う。keep-alive等の指定で1接続1要求・応答後closeを変更せず、本文受信やHost等の専用処理も変更しない。CGI転送時の除外はCGI-44・78を維持する |
| HTTP-11 | その他の入力ヘッダー | 共通構文を通った全行を保持し、専用機能を実装しないヘッダーはHTTP処理で解釈しない。通常処理では重複だけを理由に拒否せず、結合・上書きで全行を失わない。CGIでは既決定の転送対象の重複400・除外・空値省略を適用する。静的配信のRange/条件付き要求はHTTP-25・26、アップロード固有の検証はHTTP-33〜39に従う |

Content-Typeの単一値という定義と、Transfer-Encodingの1行限定というチーム方針を区別する。ヘッダー内容をHTTP処理で解釈しないことと、CGIへ転送することは別に扱う。例えば通常の静的配信で受理するX-Testの重複でも、CGI転送対象ならCGI-48により400となる。

根拠: [課題PDF](webserv.pdf)、[RFC 9110 §5.1〜5.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1)・[§7.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-7.6.1)・[§8.3〜8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.3)、[RFC 9112 §6.1〜6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.1)・[§7.1](https://www.rfc-editor.org/rfc/rfc9112.html#section-7.1)。実装・受入試験の完了は示さない。

<a id="http-request-body"></a>

### 本文とchunkedの合意事項(HTTP 3/10・2026-10-07)

以下は本文受信の正本。HTTP/1.0のTransfer-Encoding拒否、CL/TE併記拒否、HTTP/1.2〜1.9への1.1規則適用、1接続1要求、CGIへの本文の渡し方を維持する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-12 | 本文の完成条件 | Content-Length: Nは正確にNバイト、0またはCL/TE両方なしはヘッダー終端で完成。chunkedはサイズ0の終端チャンク・trailer・最後の完全に空のCRLF行まで必要。本文の区切りはGET/POST/DELETE共通で、用途は処理先が判断する。完成後の余剰データを本文や次の要求として処理しない |
| HTTP-13 | 逐次受信と本文の保持 | chunkedはサイズ行→指定バイト数の本文→CRLFを繰り返し、サイズ0でtrailerへ進む。サイズ行・本文・CRLFの受信断片境界では状態を保持して続きの受信を待つ。本文のNUL・CR・LFを保持し、転送の区切りだけ除去する。文字コード変換やフォーム解析等を共通受信処理へ追加しない |
| HTTP-14 | チャンクサイズと区切り | サイズは1文字以上の16進数字。大小文字・先頭0を受理し、0/00等は終端。空・符号・0x接頭辞・非16進文字・数値の桁あふれ・本文直後のCRLF不正は400。断片末尾のCRは続き待ちとする。本文中の改行は区切りとして検索せず、宣言されたバイト数を読む |
| HTTP-15 | chunk extension | RFC 9112 §7.1.1のtoken、任意のtoken/quoted-stringの値、引用符・エスケープと認められるSP/HTABを検査する。正常なextensionは名前によらず解釈せず捨て、不正構文は400。本文・CGI環境変数へ追加しない。サイズ行の4;foo=barは受理するが、TEヘッダーのchunked;foo=barはHTTP-09の400を維持する |
| HTTP-16 | trailer | 最後の空CRLF行まで読み、名前・コロン・改行・空白・制御文字を既存の共通ヘッダー構文で検査する。不正は400、正常な全行は捨てる。Host等の個別値・重複の意味検査は行わず、既存ヘッダー・本文長・CGI環境変数を変更しない。初期Trailerヘッダーの指定や予告名との一致を必須にしない |
| HTTP-17 | chunked付加情報の上限 | サイズ＋extensionの1行はCRLF込み8 KiBで超過400。extension部分の要求全体の合計は32 KiBで超過400。trailer全体は各CRLFと最後の空行込み32 KiBで超過431。上限と同じ長さは受理する。いずれもコード内定数でConfig項目を増やさない |
| HTTP-18 | 本文上限の早期判定 | 復号後の本文にclient_max_body_sizeを適用し、等しい長さは受理、超過413。正常に検証・数値変換した次のサイズ行から累積超過が分かれば、その本文を待たず413。サイズ・区切り・extension・trailerは本文バイト数へ含めない。CLで超過が分かった時の既存の早期413も維持する |
| HTTP-19 | 未完了要求 | 必要な本文・終端が届く前の受信EOFでは未完了要求を破棄してcloseし、追加のエラー応答は生成しない。正常な要求が完成するまでアップロード保存・DELETE・CGI起動を行わない。400/413等の早期拒否は維持する。タイムアウトでも未完了要求を処理しない。期限・408・具体的な後始末はHTTP-65〜70に従う。完成後のCGI受信EOFはCGI-113を維持する |

HTTP-17のextension合計は、各サイズ行の最後の16進数字の直後からCRLF直前までを数え、SP/HTAB・セミコロン・引用符・値を含む。サイズ0の終端行も対象とする。trailerの32 KiBは初期ヘッダーの32 KiBとは別枠。付加情報は構文検査後に捨て、要求全体として蓄積しない。追加のチャンク個数制限や符号化済み本文全体の上限は設けない。

8 KiB(8,192バイト)・32 KiB(32,768バイト)はチームの上限で、課題/RFC指定の数値ではない。受信中に超過が確定した場合は終端を待たず拒否する。本文を正常受信した場合はバイト列をそのまま保持し、CGIには復号済み本文と既決定のCONTENT_LENGTH・stdin EOFを渡す。

根拠: [課題PDF](webserv.pdf)のchunked復号・CGI入力EOF・要求を永久に待たせない要件、[RFC 9112 §6〜6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6)・[§7.1〜7.1.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-7.1)・[§8](https://www.rfc-editor.org/rfc/rfc9112.html#section-8)。未知のextensionは無視が必須、trailerの読み捨てと未完了要求へのエラー応答は任意。実装・受入試験の完了は示さない。

<a id="http-static-files"></a>

### 静的ファイルの合意事項(HTTP 4/10・2026-10-07)

以下は通常の静的GET配信の正本。location・root/alias・末尾/補完・index優先・autoindex on/offと、CGIへ進む条件の既決定を維持する。CGIの出力・ヘッダー転送規則は変更しない。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-20 | 配信する種類 | statで通常ファイル・ディレクトリを判別する。通常ファイルを読み取って200とし、空ファイルも200/本文0バイト。ディレクトリはC-27の末尾/補完→C-15のindex→未発見ならautoindexへ進む。FIFO・デバイス等の他の種類は開かず403。存在するindexが読み取り不可なら403、通常ファイル以外でも403とし、一覧へ切り替えない。選ばれたindex.py等への既存CGI判定を維持する |
| HTTP-21 | ファイル確認の失敗 | stat/access(R_OK)の失敗理由はCGI-84〜90と同じ分類に揃える。ENOENT/ENOTDIRは404、EACCES/EPERMは403、ENAMETOOLONGは414、その他は500。stat/access時のerrnoによる分類と、read/write/recv/send後のerrno分岐禁止を区別する |
| HTTP-22 | 読み取りと成功応答 | 事前確認後のファイルopen・読み取り失敗は500。読み取り成功を確認してから200の送信を始め、失敗時の部分本文を成功応答へ使わない。ファイルのバイト列をそのまま返し、Content-Lengthは実際の応答本文バイト数から生成する。読み取り方法・保持・全体メモリはHTTP-71・72の採用案に従う |
| HTTP-23 | Content-Type | 実際に配信するファイル名の最後の拡張子をASCIIの大小文字を区別せず下表で判定する。indexも実際のファイル名で判定する。未知・拡張子なしはapplication/octet-streamで配信する。内容推測・MIME設定項目・文字コード変換は追加せず、charset=utf-8を一律に付けない |
| HTTP-24 | 隠しファイル | 名前が.で始まるファイル・ディレクトリも通常と同じ扱いとし、名前だけによる拒否規則を追加しない。autoindexへの表示はHTTP-27に従う。symlinkを置かない運用前提と検出機能を省くC-26を維持する |
| HTTP-25 | Range | 通常の静的配信でRangeを解釈せず、通常どおりファイル全体を返す。範囲配信の206/416処理を追加しない。Rangeの無視はRFC 9110 §14.2で許容される。その他の存在・権限等の失敗応答は維持する |
| HTTP-26 | 条件付き要求 | 通常の静的配信でIf-Modified-Since/If-None-Match/If-Match/If-Unmodified-Since/If-Rangeを解釈せず、通常のGETとして処理する。静的配信から条件判定による304/412を生成せず、ETag/Last-Modifiedも生成しない。RFC上の必須評価規則を含めた対応を本課題向けの範囲から外すチーム方針であり、RFC全文準拠とは異なる。CGIへの既存のヘッダー転送規則は維持する |

HTTP-23の固定表。タイプ名はIANA/RFCに基づき、採用する拡張子の範囲はチーム方針とする。

| 拡張子 | Content-Type |
|---|---|
| .html / .htm | text/html |
| .css | text/css |
| .js / .mjs | text/javascript |
| .json | application/json |
| .txt | text/plain |
| .xml | application/xml |
| .png | image/png |
| .jpg / .jpeg | image/jpeg |
| .gif | image/gif |
| .svg | image/svg+xml |
| .ico | image/vnd.microsoft.icon |
| .pdf | application/pdf |
| その他・拡張子なし | application/octet-stream |

本文の文字コードを確認しないため、静的ファイル全体をUTF-8と宣言しない。MIMEの拡張子比較だけを大小文字非区別とし、ファイルパスやCGI拡張子の既存比較へ転用しない。Last-Modified生成はRFCの推奨、If-Match/If-None-Match等の評価は必須規則を含むため、すべてをRFC上任意とは説明しない。課題PDFにはこれらの各機能を必須とする記載はないが、個別の評価免除を保証しない。

根拠: [課題PDF](webserv.pdf)の完全な静的サイト配信・ブラウザ互換・正確なステータス、[RFC 9110 §8.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.3)・[§8.8.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8.2)・[§13.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1)・[§14.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-14.2)、[IANA Media Types](https://www.iana.org/assignments/media-types)、[RFC 9239 §6.1.1](https://www.rfc-editor.org/rfc/rfc9239.html#section-6.1.1)、[Linux stat](https://man7.org/linux/man-pages/man2/stat.2.html)・[access](https://man7.org/linux/man-pages/man2/access.2.html)。OSエラーからHTTPコードへの対応はチーム方針。実装・受入試験の完了は示さない。

<a id="http-autoindex"></a>

### autoindexの合意事項(HTTP 5/10・2026-10-07)

以下は一覧生成の正本。C-15・16・27のindex優先・autoindex on/off・HTML形式・GETの末尾/補完を維持する。indexが存在するが読み取り不可/通常ファイル以外ならHTTP-20の403とし、一覧へ切り替えない。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-27 | 表示項目と対象 | ディレクトリ内の各項目の名前とリンクだけをHTMLで表示する。サイズ・更新日時・種類を表示せず、項目ごとのstatを追加しない。ファイルとディレクトリは同じ形式。.example等の隠し項目も載せるが、.と..は載せず、親へのリンクも作らない。空ディレクトリは空の一覧を200で返す。掲載はアクセス成功を保証せず、クリック後に通常のルーティング・権限確認を行う |
| HTTP-28 | 並び順 | 元の名前のバイト列を大小文字を区別して昇順に並べる。各バイトは符号なしとして比較し、同じ接頭辞なら短い名前を先にする。ディレクトリ優先・日本語の読み順・利用者による並び替え機能は追加しない |
| HTTP-29 | HTMLエスケープ | 表示名の&・<・>・二重引用符・単一引用符をHTMLエスケープし、名前をHTMLとして解釈させない。動的な文字列をHTMLへ埋め込む場合も同じ規則を適用する。URLのpercent-encodingとは別の処理として扱う |
| HTTP-30 | リンク先 | 正規化済みURLパスをC-29の規則で再エンコードし、その末尾/にエンコードした項目名を追加する。名前はASCII英数字・-._~以外の各バイトを大文字%XXにする。root/aliasの実ファイルパスや元queryを使わない。種類による末尾/は追加せず、ディレクトリのクリック後はC-27の301へ任せる。hrefは二重引用符で囲み、HTMLへの埋め込み規則を守る |
| HTTP-31 | 一覧取得の失敗 | opendir失敗は消失/パス不正(ENOENT/ENOTDIR)404、権限(EACCES/EPERM)403、パス長(ENAMETOOLONG)414、その他500。readdirの列挙途中の失敗・closedirの失敗は500。readdir直前にerrnoを0にしてNULL時に正常終端と失敗を区別する。read/write/recv/send後のerrno参照禁止とは区別する。最後まで列挙・生成・closeに成功してから200とし、途中の一覧を成功応答にしない |
| HTTP-32 | 大きな一覧 | 完成するHTML本文全体を8 MiB(8,388,608バイト)以内とし、等しい長さは受理、超過500。列挙中からエスケープ後の名前・リンク・固定HTMLを含む完成時の長さを確認し、超過が分かったら打ち切り、取得済みの一覧を破棄してディレクトリを閉じる。ページ分割・追加の件数上限・Config項目は増やさない |

例: /docs/内のa b&c.txtは、`<a href="/docs/a%20b%26c.txt">a b&amp;c.txt</a>`として生成する。名前のpercent記号も%25へ変換し、要求のパスは既決定どおり1回だけデコードする。相対URLの解決に依存せず、要求パス内のエンコードされた/等があっても正規化済みパスを使う。

HTTP-32の8 MiBは一覧専用のチーム定数で、CGI出力上限と数値を揃えたもの。課題/RFC指定の値ではなく、HTTP応答ヘッダーはこのHTML本文上限へ数えない。要求本文の超過ではないため413とせず、通常の500/error_page経路へ進む。全体メモリ・I/OはHTTP-71・72の採用案、検証条件はHTTP-74に従う。

根拠: [課題PDF](webserv.pdf)の一覧表示on/off、[NGINX autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html)、[RFC 3986 §2.1〜2.3](https://www.rfc-editor.org/rfc/rfc3986.html#section-2.1)、[HTML Standard](https://html.spec.whatwg.org/multipage/syntax.html#syntax)、[Linux opendir](https://man7.org/linux/man-pages/man3/opendir.3.html)・[readdir](https://man7.org/linux/man-pages/man3/readdir.3.html)・[closedir](https://man7.org/linux/man-pages/man3/closedir.3.html)。表示項目・順序・上限・失敗時のHTTPコードはチーム方針。実装・受入試験の完了は示さない。

<a id="http-upload"></a>

### アップロードの合意事項(HTTP 6/10・2026-10-07)

以下はWebserv自身によるPOST保存の正本。POST許可・upload_store有効・CGIとの分離、root/aliasによるGET公開、HTTP本文完成後の処理開始を維持する。CGIへのPOSTには本節のmultipart・単一ファイル制限を適用せず、既決定どおり本文を渡す。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-33 | 受付形式 | multipart/form-dataに限定し、ブラウザのフォームとcurl -Fに対応する。生の本文・URLエンコードされたフォーム等の他形式、Content-Type欠落/空値は415。1ファイル限定と他形式の不受理はチームの受付範囲で、課題/RFCが単一ファイルへ限定したとは説明しない |
| HTTP-34 | boundaryと本文構文 | boundaryは引用符付き/なしを受理し、値は大小文字を区別する。RFC 2046の文字規則に従う1〜70文字とし、欠落・空・重複・不正は400。CRLFによる区切り・終端を検査し、終端欠落は400。区切り行末のSP/HTABを受理し、最初の区切り前/終端後の付加部分は捨てる。中身の同じ文字列だけで分割せず、区切りの位置と形式を確認する |
| HTTP-35 | パートヘッダー | 各パートにContent-Disposition: form-dataとnameを要求し、欠落・重複・不正は400。ヘッダーはCRLF限定・折り返しなし、終端空行込み8 KiBで超過400。パートのContent-Typeは省略可能で保存に使わず、その他のヘッダーも構文確認後に捨てる。ただしContent-Transfer-EncodingはHTTP-37を適用する。名前とfilenameを解析して本文の範囲を確定する |
| HTTP-36 | 保存するファイル数 | 非空のfilenameを持つファイルが正確に1つなら保存し、0個/2個以上は400。フォームのnameをfile等へ固定しない。filename欠落の通常フォーム項目は構文確認後に捨て、filenameが空なら未選択扱い。非空の名前と0バイトの中身は空ファイルとして保存する |
| HTTP-37 | 中身と符号化 | NUL/改行を含むバイト列のまま保存し、文字コード変換・圧縮展開・旧multipart/mixed等のネスト解析は追加しない。要求のContent-Encodingが非空なら415。パートのContent-Transfer-Encodingは省略/空/binaryだけ受理し、その他415。binaryは符号化名として大小文字を区別しない。CGIのContent-Encoding転送・本文加工なしの規則は変更しない |
| HTTP-38 | 保存名と保存先 | filenameの/・逆斜線で区切った最後の名前を使い、ディレクトリ情報は保存先へ使わない。結果が空・.・..・制御文字を含む場合は400。名前の空白/非ASCIIを保持し、percent-encodingをデコードしない。常にupload_store＋保存名へ保存し、POST先のURLはlocation選択に使う |
| HTTP-39 | 検証と上限 | HTTP要求とmultipart全体の構文・単一ファイル・保存名を確認してから作成する。client_max_body_sizeはパートヘッダー/区切り/付加部分を含む復号後の要求本文全体へ適用し、超過413。追加のファイルサイズ上限を設けない。全体メモリ・I/OはHTTP-71・72の採用案に従う |
| HTTP-40 | 新規作成と競合 | openのO_CREAT/O_EXCL/O_WRONLYで新規作成し、同名のファイル・ディレクトリ等の存在は409。事前の存在確認だけに依存せず、上書き・自動採番は追加しない。新規ファイルのmode指定は0600とする |
| HTTP-41 | 保存の完了と失敗 | 短いwriteでは残りを書き、0以下なら500。write後のerrno分岐をせず、close失敗も500。open時の同名対象は409、権限不足/読み取り専用は403、保存名によるパス長制限超過は400、保存先消失/その他のopen失敗は500。保存とcloseが成功するまで成功応答にしない |
| HTTP-42 | 成功応答 | 保存/close成功後に200と保存名を表示する小さなHTMLを返し、動的な名前はHTTP-29でエスケープする。RFCの新規作成時201/Locationの推奨と区別し、保存結果を200で返すチーム方針とする。公開URLを自動推定してLocationを生成しない。本文の実際のバイト数からContent-Lengthを作る |
| HTTP-43 | 途中ファイルの後始末 | 保存途中に失敗して今回新規作成したファイルだけを、保存用fdの後始末後にHTTP-50のstd::removeで削除する。削除失敗時はHTTP-49に従う。409等で作成していない既存対象を削除してはならない。元の保存失敗の500を成功へ変更しない |
| HTTP-44 | GETでの取得 | root/aliasを保存先へ対応させて通常のGETで公開する。upload_storeからGET公開先を自動設定せず、locationとディスク上の保存先をCGI実行場所から分け、アップロード先でCGIを実行しない既決定を維持する |

例: ../photo.jpgやCのドライブ/フォルダ表記を含むfilenameも、区切り後のphoto.jpgだけを保存名とする。a%20b.txtはその名前をそのまま保存し、GETのURLではpercent記号を%25へ再エンコードする。filenameの文字列処理と、要求URLパスの1回デコード・正規化を混同しない。パートの区切りに付属するCRLFはファイル本文へ含めず、ファイル自身の改行やNULを保持する。

パートヘッダーの8 KiB(8,192バイト)はチーム定数で、課題/RFC指定の上限ではない。初期HTTPヘッダーの32 KiBとは別枠だが、パートヘッダーは要求本文の一部なのでclient_max_body_sizeには含む。CGI転送用のHTTP環境変数を、multipartのパートヘッダーから作る処理は追加しない。

根拠: [課題PDF](webserv.pdf)のアップロード/POST/保存先指定、[RFC 7578 §4.1〜4.8](https://www.rfc-editor.org/rfc/rfc7578.html#section-4.1)、[RFC 2046 §5.1.1](https://www.rfc-editor.org/rfc/rfc2046.html#section-5.1.1)、[RFC 9110 §9.3.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.3)・[§15.3.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.3.1)、[Linux open](https://man7.org/linux/man-pages/man2/open.2.html)。実装・受入試験の完了は示さない。保存失敗の後始末を、保存の成功扱いやファイルの切り詰めで代替しない。

<a id="http-delete"></a>

### DELETEと途中ファイルの後始末の合意事項(HTTP 7/10・2026-10-07)

HTTP-45〜50は合意済み。2026-10-07のユーザー決定により、削除手段をC++98標準ライブラリのstd::removeに確定した。本節がDELETEと保存失敗時の削除の正本であり、実装・受入試験は未完了。

| ID | 項目 | 決定・確認状況 |
|---|---|---|
| HTTP-45 | DELETEの対象解決 | 通常locationのroot/aliasで、GETと同じパス解決規則を使う。allow_methodsによる405＋Allow、Configのreturnの処理順、CGI用locationのDELETE禁止は維持する。upload_storeからDELETE先を自動設定しない |
| HTTP-46 | 削除する種別 | 通常ファイルだけを削除する。ディレクトリとその他の種別は403。ディレクトリの再帰削除・indexファイルへの置き換えを追加せず、GET用の末尾スラッシュ補完・index/autoindex処理へ進めない |
| HTTP-47 | DELETE成功 | std::removeの戻り値0を確認した後に204を返し、本文もContent-Lengthも出力しない。削除前に成功応答を送らない |
| HTTP-48 | DELETE失敗 | 対象なし404、通常ファイル以外/権限不足/読み取り専用403、URLから解決したパスが長すぎる場合414、その他の処理失敗500。statで通常ファイルと確認してからstd::removeを呼ぶ。削除失敗直後のerrnoを保存し、ENOENT/ENOTDIRは404、EACCES/EPERM/EROFSは403、ENAMETOOLONGは414、その他は500へ分類する。削除権限を対象ファイルのR_OK/W_OKで代用せず、親ディレクトリの権限等を含む実際の削除結果で判定する。消失等を理由に削除を再試行しない |
| HTTP-49 | 保存失敗時の後始末 | 今回新規作成したファイルだけを、保存用fdの後始末後にstd::removeで削除する。後始末の削除にも失敗したら、元の保存失敗の500を維持し、削除できなかったパスをログに記録する。削除を無期限に繰り返さず、409で作成していない既存ファイルへ触れない |
| HTTP-50 | 削除手段 | C++98標準ライブラリの`<cstdio>`にある`int std::remove(const char*)`を使用可能とし、解決済みのファイルパスの`c_str()`を渡す。戻り値0は成功、非0は失敗。ディレクトリも削除できる関数だが、DELETEではHTTP-46の通常ファイル確認を維持し、空ディレクトリも呼出し前に403とする。read/write後のerrnoによる分岐禁止を維持し、std::remove自身の失敗理由はHTTP-48で分類する。unlink/rmdirの直接呼出し、削除用の外部コマンド/CGI、追加Config/APIは導入しない |

課題はDELETEを必須とするが、通常ファイルだけに限定することや失敗分類はチーム方針。[RFC 9110 §9.3.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.5)では、DELETEの完了後に追加情報を返さない場合は204が推奨される。RFC上はURLと資源の対応を取り除く意味で、ディスク上の実体削除そのものが必須ではない。ただし、実ファイルを残して配信だけ隠す方式は本合意と異なり、同名再アップロード・再起動後の扱いも別途設計が必要。切り詰めや成功応答だけで削除を実装済みとはしない。実装・受入試験は未完了。

std::removeの使用可能という判断はC++98標準ライブラリとしてのチーム決定であり、PDFの列挙にremoveが追加されたと説明するものではない。ファイル/ディレクトリの削除と戻り値は[remove(3)](https://man7.org/linux/man-pages/man3/remove.3.html)、親ディレクトリの権限と削除失敗のerrnoは[unlink(2)](https://man7.org/linux/man-pages/man2/unlink.2.html)を参照する。通常ファイル以外を拒否する既存方針と、保存失敗の500を後始末で上書きしない既存方針に沿って具体化し、追加の設計選択は設けない。

<a id="http-response"></a>

### HTTP応答生成の合意事項(HTTP 8/10・2026-10-07)

HTTP-51〜58は応答生成の正本。HTTP-56のDate省略は見直し得る暫定案。Configのerror_pageとCGIの出力検証・転送・説明文・本文禁止規則を維持する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-51 | 基本形式 | HTTP/1.0のステータス行、CRLFによるヘッダーと終端空行を生成し、Connection: closeを付ける。Transfer-Encodingを出力せず、1接続1要求・送信完了後の切断を維持する |
| HTTP-52 | 本文と長さ・種類 | Content-Lengthはエラーページ置換等を終えた最終本文のバイト数から生成する。通常の空本文は0。静的ファイルのContent-TypeはHTTP-23、一覧・保存結果・エラーページはtext/html。DELETE成功204は本文/Content-Lengthなし。CGIの204/304の本文・CL省略、205の空本文/CL0等の既決定を維持する |
| HTTP-53 | 405のAllow | Webservが生成する405には、選ばれたlocationで許可したメソッドだけをGET, POST, DELETEの固定順でカンマ＋SP区切りで列挙する。Configの記述順に依存せず、禁止メソッドを自動追加しない。CGI由来のAllowの転送規則は変更しない |
| HTTP-54 | Webservのリダイレクト | ConfigのreturnとディレクトリGETの末尾スラッシュ補完による301/302は、既決定のLocation、空本文、Content-Length: 0で応答する。移動先を説明する追加HTMLは生成しない。CGIのLocation・本文の規則は変更しない |
| HTTP-55 | Server | Webserv自身ではServerヘッダーを生成しない。RFCで任意のためBare minimumとして省く。CGI由来ヘッダーの既存の転送規則は維持する |
| HTTP-56 | Date(暫定案) | 使用可能な暦日時の取得手段が確定するまで、Webserv自身ではDateを生成しない。固定日時・ファイル更新日時で代用しない。HTTP/1.0でも推奨、RFC 9110では時計を持つサーバーの2xx/3xx/4xxに必須であり、関数制限だけを理由に免除されたとは説明しない。課題向けの暫定的な制限として記録し、使用可能な取得手段が確認できたら見直す。CGI由来ヘッダーの転送規則は維持する |
| HTTP-57 | エラー判定の順序 | 全コードの一律優先順位表を作らず、要求行・ヘッダー・本文の検証で拒否が確定したら通常処理へ進めない。正常な要求はlocation選択→メソッド許可→Configのreturn→ファイル/アップロード/CGI処理。確定したエラーを後始末の失敗で上書きしない。既決定の早期拒否とExpectより先に確定した400/413等の優先を維持する |
| HTTP-58 | error_page適用後の応答 | 400〜599の本文を指定ファイルで差し替え、未指定/読み取り失敗なら内蔵HTMLへ一度だけ戻す。元のステータス、CGIの検証済み説明文、必要なAllow等の保持を維持し、置換後の本文からCLを生成する。200〜399へは適用せず、再ルーティング・CGI実行・再帰的なエラー生成を追加しない(C-20・CGI-130〜133) |

例: DELETE禁止のlocationに正常なDELETE要求が届いた場合は、ファイルの有無を調べる前に405＋Allowを返す。対象がないことを理由に404へ変更しない。エラーページ読み取り失敗でも元の404等を500へ変更しない。本文の長さは文字数ではなくバイト数で数える。

根拠: [課題PDF](webserv.pdf)の正確なHTTPステータスとデフォルトエラーページ、[RFC 1945 §10.4](https://www.rfc-editor.org/rfc/rfc1945.html#section-10.4)・[§10.6](https://www.rfc-editor.org/rfc/rfc1945.html#section-10.6)、[RFC 9110 §6.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-6.6.1)・[§8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6)・[§10.2.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.4)・[§15.5.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.6)。必要なヘッダーに限定する構成・空本文のWebservリダイレクト・エラー処理順はチーム方針。Dateの暫定的省略を含め、RFC全体への完全準拠とは説明しない。実装・受入試験は未完了。

<a id="http-api"></a>

### HTTP APIとデータの保持の合意事項(HTTP 9/10・2026-10-07)

本節がHTTPクラス間の責務・API・保持方針の正本。具体的なHttpRequest/HttpResponseの宣言は本書の主要インターフェースを参照し、CGIの既存シグネチャは03を正本として維持する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-59 | 責務 | Clientがソケットの受信/送信、HttpRequestがバイト列の解析と完成/エラー判定、Routerが完成した要求とConfigによる処理選択、HttpResponseがステータス/ヘッダー/本文と送信用バイト列の生成を担当する。HttpRequestはrecv/sendを呼ばない |
| HTTP-60 | HttpRequestの生成と状態 | explicit HttpRequest(std::size_t maxBodySize)でConfigの本文上限を生成時に渡し、途中では変更しない。外部へ公開するStateはINCOMPLETE/COMPLETE/ERRORの3つ。要求行/本文/chunkedの内部解析段階は非公開。appendData(const char*, std::size_t)のたびClientが状態を確認し、途中なら受信継続、完成ならRouterを1回だけ呼び、ERRORならエラー応答へ進む |
| HTTP-61 | エラーコードと終端状態 | getErrorCode()はINCOMPLETE/COMPLETEなら0、ERRORなら最初に確定したHTTPコード。COMPLETE/ERROR後にappendDataしても結果を変更せず、余剰バイトを次の要求へ使わない。要求の再利用・途中の上限変更を行わない |
| HTTP-62 | 要求の保持とgetter | ClientがHttpRequestを値として保持し、要求全体を別の受信バッファへ重複保存しない。Routerへは完成済みHttpRequestをconst参照で渡す。getPathはquery分離・1回デコード・正規化済みのURLパスで、Routerが再デコードしない。getQueryStringは未デコード、getBodyはchunked復号済み。メソッド/受信版/単一ヘッダーgetterを維持し、getHeadersは重複を含む全行を保持する。参照はHttpRequestの生存中だけ使う |
| HTTP-63 | Routerの寿命と結果 | EventLoop内のRouterを1個使い回し、要求ごとのデータを保持し続けない。既存routeのRESPONSE_READYは有効なHttpResponse、CGI_REQUIREDは有効なCgiRequestを出力する。戻り値に対応する出力引数だけを使用し、Config/CGIの公開APIとCGI用データのコピー・寿命は変更しない |
| HTTP-64 | 応答と送信バッファの保持 | HttpResponseがヘッダー/本文/ステータスを値で保持する。serialize()で送信用std::stringを1回だけ生成し、Clientの送信バッファへswapで渡す。Clientは送信済みバイト数を保持し、短いsendなら残りを続ける。バイト列を渡した後は元のHttpResponseを保持し続ける必要がない。I/O量・全体メモリはHTTP-65・71・72に従う |

課題/RFCはこのクラス構成やAPIを指定していない。上の責務分担はチーム方針であり、バイト列としての解析は[RFC 9112 §2.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-2.2)、完成/未完了の区別は[§6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3)・[§8](https://www.rfc-editor.org/rfc/rfc9112.html#section-8)を踏まえる。実装・受入試験は未完了。

<a id="http-network-boundary"></a>

### ネットワーク境界・計時・通常ファイルI/Oの合意事項(HTTP 10/10・2026-10-07)

課題PDFを再確認して採用する最小構成。計時・切断待ち時間・通常ファイルの一括処理・独自の全体資源上限を追加しない構成は、実装と負荷試験で見直し得る現時点の採用案とする。HttpRequestの公開3状態、Routerの2種類の結果、Config/CGI APIは維持する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-65 | 通常ソケットのI/O | 単一pollの準備通知に従い、接続ごと・各周にrecv/sendをそれぞれ最大1回、各最大8 KiBとする。ソケットはノンブロッキング。sendは残量との小さい方を渡し、正の戻り値だけ送信オフセットを進める。recvの負値、送るデータがあるsendの0以下は接続を閉じ、errnoで再試行・待機へ分岐しない。送信失敗後に追加のHTTP応答を作らない。CGIパイプの既決定は維持する |
| HTTP-66 | 受信EOF | 受信中の未完了要求のrecv=0はHTTP-19の破棄/close。正常な要求の完成後、または早期エラー応答へ進んだ後は受信EOFを記録しPOLLIN監視を外すが、生成・送信中の応答を続ける。EOFを根拠に次の要求を作らない。通常ソケットのPOLLHUPだけで未読データを破棄せず、残りをrecvして0を確認する。POLLERR/POLLNVALやI/O失敗はclose。CGI待機中の切断判定・停止/回収はCGI-113等を変更しない |
| HTTP-67 | 応答後の切断(採用案) | 正常応答も413/417等の早期応答も、送信中の追加受信は8 KiBの固定バッファで読み捨て、解析・保存しない。全応答のsend完了後にshutdown(SHUT_WR)し、受信EOF確認済みならclose、未確認なら読み捨てを続けEOF/エラーまたは送信完了から1秒でcloseする。期限は受信で延長せず、shutdown失敗もclose。内部ClientにDRAININGを設け、待機中は送信データがないのでPOLLOUTを外す。1秒は応答到達・ACKを保証する値ではなく、実際の早期拒否/負荷試験で見直す |
| HTTP-68 | 要求受信の期限 | accept成功から60秒以内に要求を完成させる。途中のバイト到着では延長しない。未完了のまま期限になった場合、1バイト以上受信済みなら408の応答経路へ進み、未受信の空接続なら応答なしでcloseする。未完了要求の保存/削除/CGI起動をしない。受信失敗/EOFが先に確定した場合はその切断方針を維持する |
| HTTP-69 | 応答送信の期限 | 最終応答の送信バッファが準備できた時から60秒。部分sendで延長せず、未送信のまま期限になったらcloseし、別のエラー応答を送らない。CGI待機中にこの期限を適用せず、CGIは起動から10秒・延長なし・504を維持する。全送信後はHTTP-67の1秒を使う。これらは処理がEventLoopへ戻ってから検出する期限であり、同期ファイル処理の停止を防ぐ保証ではない |
| HTTP-70 | 共通の経過時間(採用案) | CGI-124〜126のLinux /proc/uptimeの取得方法を通常接続にも使い、各周1回の有効な現在値を共有する。CGI起動直前の追加取得は維持。単一pollの待ち時間は通常接続だけの時も最大100 ms。通常の受信/送信/切断待ちで取得・解析失敗や値の逆行を検出したら、その接続を閉じ、計時なしで無期限に継続しない。CGI起動前/実行中の計時失敗は既存の500と停止/回収規則を維持する。使用可能な時計APIと断定せず、利用可否・精度・追加I/O負荷を実環境で検証する |
| HTTP-71 | 通常ファイル処理(採用案) | 静的ファイル/error_pageは8 KiBずつ読み、全読み取り成功後に応答を生成する。アップロードは本文全体の検証後、対象部分の位置/長さを使って8 KiBずつ書き、ファイル内容の別コピーを作らない。短い正のread/writeを処理し、readの0をEOF、負値を失敗、書く残量があるwriteの0以下を失敗とする。read/write後にerrnoで分岐しない。通常ディスクファイルはpoll対象にせず、現在のRouter呼出し内で完了まで処理する。これは同期の一括処理であり、8 KiBに分けても周ごとに他接続へ処理を譲る意味ではない |
| HTTP-72 | メモリと資源(採用案) | 要求/CGI/一覧の既存上限を維持し、静的ファイル専用サイズ上限・独自の接続数上限・全体メモリ上限・件数による503を追加しない。HTTP-62〜64の保持・参照・swapを使い、不要な応答/要求/送信バッファは接続破棄時に解放する。serialize時は元本文と生成文字列が一時的に共存するため、全体メモリの一定上限は保証しない。接続処理のbad_alloc等は例外を捕捉して資源を解放し、500を生成できなければ接続を閉じる。関連CGIは既存の停止/回収へ進める。これだけでOSによる強制終了を防げるとはせず、負荷試験で資源量と回収を確認する |
| HTTP-73 | ログ形式 | Loggerは[LEVEL] messageとし、暦日時を出力しない。ファイル出力なし・レベルによる絞り込みを維持する。/proc/uptimeをDateや暦日時へ流用せず、DateのHTTP-56の暫定案とは区別する |
| HTTP-74 | 受入条件と見直し | 04の通常接続/ファイル/負荷試験は未実施。部分送受信、EOF/RST、早期拒否後の応答受信、期限、計時失敗、ファイル処理中の他接続/CGIの進行、終了後のfd/所有メモリ回収を確認する。評価項目の空ページGETのSiege -bでAvailabilityが99.5%を超え、長時間の継続でメモリが増え続けず、接続がハングしないことを確認する。一括処理でイベントループが滞る、資源量/精度/可用性が基準を満たさない場合は採用案を見直し、影響するAPI/規則/試験を関連資料に揃えて記録する |

根拠: [課題PDF](webserv.pdf) p.8〜9は常にノンブロッキング・単一poll・read/write後のerrnoによる分岐禁止・無期限ハング禁止・負荷耐性を要求し、通常のディスクファイルだけは準備通知を不要としている。[RFC 9112 §9.6](https://www.rfc-editor.org/rfc/rfc9112.html#section-9.6)の段階的な切断を参考にする。[Linux poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)では通常ファイルは常に準備完了となるため、pollへ登録してもディスクI/Oが非同期になるわけではない。経過時間の取得元は[proc_uptime(5)](https://man7.org/linux/man-pages/man5/proc_uptime.5.html)。8 KiB・60秒・1秒・100 ms・クラス構成・独自の全体上限を省く選択は課題/RFCの指定値ではなくチーム方針である。

[42EvalHubの評価項目](https://www.42evalhub.com/common/webserv)(2026-10-07確認)には通常ファイルの同期I/Oがイベントループを停滞させないことと負荷試験が含まれる。ディスクファイルのpoll免除を、イベントループの停止や無制限の資源消費を認める根拠にはしない。現時点で合格・完全ノンブロッキングを実証したとは扱わない。DELETEの削除手段はHTTP-50で確定済み。Date(HTTP-56)は暫定案として残る。

### リクエストの扱い

| 項目 | HTTP/1.0リクエスト | HTTP/1.1リクエスト |
|---|---|---|
| 受理 | 受理する | 課題に必要な範囲で受理する。HTTP/1.2〜1.9にも同じ列の規則を適用(HTTP-04) |
| Host | 省略可能。指定時の重複・空値・不正値は400(C-28・C-30) | 必須。欠落・重複・空値・不正値は400(C-28・C-30) |
| 要求行・ヘッダーの改行 | CRLF限定。単独LF・不正な単独CRは400(C-31) | 同左(C-31) |
| 要求行・ヘッダーのサイズ | 要求行8 KiB・ヘッダー合計32 KiB。超過時414/431(C-32) | 同左(C-32) |
| ヘッダーの折り返し・空白 | 折り返しとコロン直前の空白は400。値の前後SP/HTABだけを除去(C-33) | 同左(C-33) |
| ヘッダー名 | tokenを検証してASCII小文字で保持。空・不正文字・コロンなしは400(C-34) | 同左(C-34) |
| ボディの長さ | ボディ付き要求はContent-Lengthで区切る | Content-Length、またはTransfer-Encoding: chunkedで区切る |
| Content-Lengthの重複 | 最大1行。同値の重複・カンマ入りも400(C-35) | 同左(C-35) |
| 長さ指定が両方ない要求 | POSTも空ボディ。省略だけで411にしない(C-36) | 同左(C-36) |
| Transfer-Encoding | 400を返して切断する | 単独chunkedを復号する。重複・不正構文・終端不正・CL併記は400、正しい一覧内の未対応符号化は501(HTTP-08・09) |
| Expect: 100-continue | この期待を無視して通常の要求として処理する | Expect未対応として、ボディを待たずヘッダー完了時点で417を返して切断する |
| その他のExpect値 | 値によらず無視する(CGI-76) | Expect未対応として417を返して切断する |

- **確定(C-31)**: HTTP/1.0・HTTP/1.1要求のリクエスト行とHTTPヘッダーの改行はCRLFだけを受理し、単独LF・不正な単独CRは400とする。受信断片の末尾がCRだけなら、その時点では不正とせず次のバイトを待つ。次がLFなら正常なCRLFとして扱い、LF以外なら不正と判定する。切断やタイムアウトで未完了になった場合は既存の接続処理に従う。この改行制限はボディのデータには適用せず、ボディ中のCR/LFを変更・拒否しない。CGI出力ヘッダーのLF/CRLF両方受理と、HTTP応答ヘッダーのCRLF生成はCGI-15の既決定を維持する。
- **確定(C-32)**: HTTP/1.0・HTTP/1.1要求のリクエスト行は末尾CRLF込みで8 KiB(8,192バイト)、その直後のヘッダー部分全体は各行のCRLFと終端の空行込みで32 KiB(32,768バイト)を上限とする。ヘッダー合計にリクエスト行・ボディを含めない。URLデコード等の前の受信バイト数で数え、上限と同じサイズは受理、超過時はそれぞれ414 / 431で拒否する。受信中に検査し、超過が確定したら行末・ヘッダー終端・ボディの到着を待たずエラー応答へ進み、既存方針どおり応答送信完了後に切断する。同じrecvで届いたボディをヘッダーサイズへ加算しない。ヘッダー1行ごとの別上限と個数上限は追加しない。2つの上限はコード内定数とし、Config項目を増やさない。ボディ上限は既存client_max_body_sizeと413、CGI出力上限はCGI-20の規則を維持する。
- **確定(C-33)**: HTTP/1.0・HTTP/1.1要求のヘッダーは折り返し(obs-fold)を受け付けず、SP/HTABで始まる空でないヘッダー行は400とする。最初のヘッダー行の字下げやSP/HTABだけの行も同様に拒否する。フィールド名とコロンの間の空白・タブは400とし、補正しない。コロン後の値は前後のSP/HTABだけを除去し、値の内部の空白・タブはこの処理では変更しない。`Host:localhost` と `Host: localhost` は同じ値として扱う。ヘッダー終端は完全に空のCRLF行とする。この処理はHTTP要求ヘッダーに限り、ボディには適用しない。フィールドごとの値の検証(HostのC-30等)は別途行い、CGI出力の検証規則は変更しない。
- **確定(C-34)**: HTTP/1.0・HTTP/1.1要求のヘッダー名はRFC 9110のtoken(ASCII英数字と ! # $ % & ' * + - . ^ _ ` | ~ のいずれか1文字以上)として検証し、ASCII大文字A〜Zを小文字a〜zに変換して保持する。名前の比較は大小文字を区別しない。空の名前・許可外の文字・名前と値を区切るコロンの欠落は補正せず400とする。ヘッダー行は最初のコロンで名前と値に分け、値内のコロンはそのまま扱う。値はC-33の前後SP/HTAB除去を行うが、一律の小文字化はしない。個別ヘッダーの値の検証は既存方針に従う。この決定はHTTP要求のヘッダー名についてのもので、重複はHostのC-28・C-30とContent-LengthのC-35に従い、その他の重複と個別値の検証規則はHTTP-06〜11に従う。CGI出力・HTTP_*転送とCGIの公開APIはCGI-39〜98で合意済み。HTTP全体の公開API/保持はHTTP-59〜64で合意済み。
- 構文・ボディ長の不正など、先に確定したエラーはそのステータスで応答する。上のExpect方針のために400や413を417へ置き換える必要はない。
- **確定(C-35)**: HTTP/1.0・HTTP/1.1要求のContent-Lengthは最大1行とする。同じ値でも2行以上は400とし、名前の大小文字だけが異なる場合もC-34に従って重複と判定する。値にカンマを含む場合は、同値の一覧を含めて400とする。複数行の結合、先勝ち・後勝ち、同値の単一化による補正は行わない。不正な数値・桁あふれの400と、Transfer-Encoding併記時の400は既存方針を維持する。検出したエラーはボディの完了を待たず既存のエラー応答経路へ渡し、応答送信完了後に切断する。この決定は要求のContent-Lengthについてであり、他のヘッダーやCGI出力に一律適用しない。Content-LengthとTransfer-Encodingが両方ない場合はC-36に従う。
- **確定(C-36)**: HTTP/1.0・HTTP/1.1要求でContent-LengthもTransfer-Encodingもない場合は、POSTを含めてボディ長0として扱う。他のヘッダー検証でエラーがなければヘッダー終端で要求の受信を完了し、ボディや接続切断を待たない。長さ指定の省略だけを理由に411を返さない。Content-Length: 0も空ボディとし、正常な正のContent-Lengthは指定バイト数、HTTP/1.1の正常なchunkedは終端まで受信する。存在する不正な長さ指定やHTTP/1.0のTransfer-Encodingを省略扱いにはせず、既存のエラー規則を適用する。要求完成後の余剰データはボディや次の要求として処理しない既存方針を維持する。空ボディで受信完了しても処理成功を保証せず、メソッド許可や処理側の検証を行う。CGIへ進む場合はstdinにデータを書かず親側の書き込みパイプを閉じてEOFを渡す。
- 本文の完成・chunked構文・付加情報上限・未完了EOFはHTTP-12〜19に従う。TCP接続の終了を正常な本文終端とせず、trailerを初期ヘッダーへ混ぜない。
- HTTP/1.1のExpect付き要求を無視してボディ待ちにしない。417への応答後、クライアントはExpectなしの別接続で再送できる。100 Continueを送る中間応答処理は実装しない。
- 1要求が完成した後の余剰データを次の要求として処理しない。keep-aliveやpipeliningの要求があっても、応答送信完了後に切断する。

### レスポンスとCGIの扱い

- ステータス行は `HTTP/1.0 <status-code> <reason-phrase>`。通常の本文付き応答は、本文のバイト数からContent-Lengthを確定し、Content-Typeとともに出力する。
- `Connection: close` を付ける。Transfer-Encodingは出力せず、chunked形式で送信しない。
- DELETE成功などで204を返す場合は、本文もContent-Lengthも出力しない。本文やContent-Lengthの可否はステータスに応じて判断する。
- 405・413・417・502・504など、実装する処理に応じたステータスはHTTP/1.0応答でも使用する。HTTP/1.0への変更を理由に削除しない。
- CGIの `SERVER_PROTOCOL` は受信した要求のバージョン(`HttpRequest::getVersion()`)を使う。応答側の固定値で上書きしない。
- chunkedで受信したボディをCGIへ渡す場合、`CONTENT_LENGTH` はCGI-31に従い、復号後の本文が1バイト以上ならそのバイト数、0バイトなら空文字列を設定する。CGIへの全ボディの書き込み後は親側stdinパイプを閉じてEOFを渡す。
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

<a id="cgi-decisions"></a>

## CGI対応範囲・確定事項

RFC 3875を参考に、対応範囲を以下に限定する。課題要件の一覧(第8節)に加え、二人が採用した動作・初期値を記録する。CGI-01〜135の決定表は本節を正本とし、説明・受け渡しAPIは [03のC12](reports/03_config_cgi_upload.md#cgi-details) にまとめる。変更時は正本と関連する説明・テスト計画を更新する。

CGI-120〜127は**現時点で採用する代替案**。実装・実行環境・動作確認の結果で見直し得るもので、動作確認済み・最終固定の仕様とは扱わない。詳細は03の項目9/10に従う。子の起動失敗時にWebservが所有するメモリを明示的に解放し、親用の停止・回収処理を実行しない方針は合意済み。具体的な終了・解放経路は代替案として実装・動作確認で検証する。CGIの10項目は合意完了であり、実装・結合試験の完了を意味しない。

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
| CGI-24 | 不正出力の扱い | 必要な形式・値・長さの検査を行い、不正なCGI出力は502にする。不正な値の推測・補正や、矛盾する値を選び直して受理する処理は実装しない。具体的な検証規則はCGI-52〜83に従う |
| CGI-25 | スクリプトの特定 | 選択済みのCGI有効locationのroot/aliasで対応するパスを先頭から辿り、設定拡張子で終わる実在する通常ファイルに到達したところを実行対象とする。そのURL上のパスをSCRIPT_NAME、後続部分をPATH_INFOとする。拡張子だけで分割せず、同じ拡張子のディレクトリは実行しない |
| CGI-26 | 実行パス・cwd | CGIの子プロセスで、実行するスクリプトが置かれたディレクトリへchdirしてから起動する。Pythonとスクリプトはともに絶対パスで指定する。cwdはPATH_INFOから決めず、追加のConfig項目も設けない |
| CGI-27 | REQUEST_METHOD | 解析済みの要求メソッドをCGI起動時に必ずREQUEST_METHODへ設定する。CGI-09のGET/POST限定に従い、値はGETまたはPOSTとする。空文字列や省略にはしない |
| CGI-28 | QUERY_STRING | CGI起動時に必ずQUERY_STRINGを設定する。値は最初の?より後ろの未デコードのクエリとし、?自体は含めない。クエリなし・末尾が?だけの場合は空文字列とし、省略しない |
| CGI-29 | PATH_INFO | CGI起動時に必ずPATH_INFOを設定する。値は既存のデコード・正規化後のURLパスからCGI-25で切り分けた後続部分とする。追加パスがなければ空文字列とし、省略しない。スクリプト直後の/だけが残る場合は/とする |
| CGI-30 | SCRIPT_NAME | CGI起動時に必ずSCRIPT_NAMEを設定する。値はデコード・正規化後のスクリプトを示すURLパスとし、PATH_INFOとクエリを含めない。root/aliasで解決したディスク上のパスは使わない。index経由では選ばれたindexのファイル名をURL上のディレクトリパスに付け、/cgi/でindex.pyを実行する場合は/cgi/index.pyとする |
| CGI-31 | CONTENT_LENGTH | CGI起動時に必ずCONTENT_LENGTHを設定する。CGIへ渡す復号後の本文が1バイト以上なら、その実際のバイト数を10進数の文字列で設定する。0バイトなら空文字列とし、省略しない。長さ指定なし・Content-Length: 0・空のchunked本文も同じ0バイトの規則に従う |
| CGI-32 | CONTENT_TYPE | 要求にContent-Typeヘッダーがあれば、HTTPパーサーで前後のSP/HTABを除去した値を、パラメーターも含めてCONTENT_TYPEへ設定する。本文が0バイトでもヘッダーがあれば設定する。ヘッダーがなければ変数を省略し、形式の推測や既定値の補完は行わない |
| CGI-33 | SERVER_PROTOCOL | CGI起動時に必ずSERVER_PROTOCOLを設定する。値は解析済みの要求バージョンに従いHTTP/1.0またはHTTP/1.1とする。応答側のHTTP/1.0固定値で上書きせず、空文字列・省略にはしない |
| CGI-34 | GATEWAY_INTERFACE | CGI起動時に必ずGATEWAY_INTERFACEを設定し、値は固定でCGI/1.1とする。要求のHTTPバージョンによって変えず、空文字列・省略にはしない。追加のConfig項目は設けない |
| CGI-35 | SERVER_PORT | CGI起動時に必ずSERVER_PORTを設定する。値はその接続を受け付けたlistenのポート番号を10進数の文字列にしたものとし、標準ポートでも省略しない。空文字列・省略にはせず、追加のConfig項目は設けない |
| CGI-36 | REMOTE_ADDR | CGI起動時に必ずREMOTE_ADDRを設定する。値はacceptで取得した接続元のIPv4アドレスをドット区切りの10進数で表したものとする。ポート番号は含めず、空文字列・省略にはしない。逆引きDNSは行わず、X-Forwarded-For等のHTTPヘッダーで値を上書きしない |
| CGI-37 | SERVER_NAME | CGI起動時に必ずSERVER_NAMEを設定する。有効なHostがあればポート部分を除いたホスト名またはIPアドレスを使い、名前の大小文字は保持する。HTTP/1.0でHostがなければgetsocknameで取得した接続のサーバー側IPを使う。空文字列・省略にはせず、listenの0.0.0.0や接続元IPは代用しない。DNS照会やserver_name設定項目は追加しない |
| CGI-38 | SERVER_SOFTWAREの省略 | 最小構成ではSERVER_SOFTWAREをCGIへ渡す環境変数に含めない。固定文字列の設定・生成処理と、そのためのConfig項目は実装しない |
| CGI-39 | HTTP_*の変換 | 転送対象の要求ヘッダー名をASCII大文字に変換し、-を_へ置き換え、先頭にHTTP_を付けて環境変数名にする。値はHTTPパーサーで前後のSP/HTABを除去したものを使い、大小文字を保持する。転送対象・除外・重複はCGI-40〜49・75〜83に従う。名前をASCII英数字と-に限定するCGI-47により、異なる名前の変換後の衝突を避ける |
| CGI-40 | 専用変数と重なるヘッダーの除外 | Content-TypeとContent-LengthはCGI-31・32の専用変数だけで渡す。HTTP_CONTENT_TYPEとHTTP_CONTENT_LENGTHは設定しない。専用変数の値と設定・省略の規則は既存の合意を維持する |
| CGI-41 | Transfer-Encodingの除外 | 要求のTransfer-EncodingはHTTP_*へ転送せず、HTTP_TRANSFER_ENCODINGを設定しない。CGIには復号後の本文を渡し、本文長とstdinのEOFは合意済みの規則に従う |
| CGI-42 | Connectionの除外 | 要求のConnectionヘッダー自体はHTTP_*へ転送せず、HTTP_CONNECTIONを設定しない。要求の値によらず、1接続1要求・応答後に切断する既存方針を維持する |
| CGI-43 | その他の通信制御用ヘッダーの除外 | Keep-Alive・TE・Upgrade・Proxy-ConnectionはHTTP_*へ転送せず、HTTP_KEEP_ALIVE・HTTP_TE・HTTP_UPGRADE・HTTP_PROXY_CONNECTIONを設定しない。ヘッダー名による除外とし、この合意によって各ヘッダーの機能を新たに実装するものではない |
| CGI-44 | Connectionに列挙されたヘッダーの除外 | Connectionの値に列挙されたヘッダー名もHTTP_*への転送対象から除外する。名前はカンマで区切り、前後のSP/HTABを除去して大小文字を区別せず照合する。この除外で本文の受信処理や専用環境変数の規則は変更しない |
| CGI-45 | Proxyの除外 | 要求のProxyヘッダーはCGIへ転送せず、HTTP_PROXYを設定しない。Proxyヘッダーがあることだけを理由に要求を拒否する処理は追加しない |
| CGI-46 | 認証用ヘッダーの除外 | Authorization・Proxy-AuthorizationはCGIへ転送せず、HTTP_AUTHORIZATION・HTTP_PROXY_AUTHORIZATIONを設定しない。これらの存在だけを理由に要求を拒否する処理や、認証情報を解釈する処理は追加しない |
| CGI-47 | HTTP_*へ転送する名前の文字範囲 | HTTP_*へ転送するヘッダー名はASCII英数字と-だけで構成されるものに限定し、合意済みの除外規則を適用する。_やその他の記号を含む名前はCGIへの転送だけを省略し、HTTP要求としての受付は既存のtoken検証を維持する。転送対象の同名ヘッダーの重複はCGI-48に従う |
| CGI-48 | HTTP_*へ転送する同名ヘッダーの重複 | CGIへの転送対象となる同名ヘッダーが複数行あれば400を返し、CGIを起動しない。名前の大小文字を区別せず、値が同じ場合も重複として扱う。結合・先勝ち・後勝ちの処理は実装しない。これはチームの受理範囲の制限とする |
| CGI-49 | HTTP_*の空値 | HTTP層の検証を通った転送対象ヘッダーでも、前後のSP/HTABを除去した結果が空なら対応するHTTP_*を設定しない。空値はヘッダーなしと同じ省略扱いとし、個別ヘッダーの既存の検証規則は維持する |
| CGI-50 | CGI用環境の構築 | 合意済みの11種類の環境変数と転送対象のHTTP_*だけから、execveへ渡すCGI用環境を構築する。各変数の設定・省略規則を維持し、親プロセスの環境変数はコピーしない。追加の環境変数用Config項目は設けない |
| CGI-51 | stderr | CGIのstderrはWebservのstderrをそのまま引き継ぐ。stdoutへ混ぜず、専用のパイプ・監視・収集処理は追加しない。stderrへの出力だけでは502にせず、終了状態とCGI出力は既存の規則で検証する |
| CGI-52 | CGI専用応答ヘッダーの重複 | CGI出力のContent-Type・Status・Locationは、それぞれ最大1行とする。同名ヘッダーが複数行あれば、名前の大小文字を区別せず、同じ値でも不正出力として502にする。結合・先勝ち・後勝ちの処理は実装しない |
| CGI-53 | CGI出力ヘッダーの折り返し・終端 | CGI出力のヘッダー部分では、SP/HTABで始まる空でない行を502とし、継続行の結合や字下げ除去は行わない。空白だけの行も502とし、完全に空の行だけをヘッダー終端とする。本文の空白や改行は保持する |
| CGI-54 | CGI出力ヘッダーのコロン直前の空白 | CGI出力のヘッダー名とコロンの間にSP/HTABがあれば、不正出力として502にする。空白を除去して受理する処理は実装しない |
| CGI-55 | CGI出力ヘッダー名の文字範囲 | CGI出力のヘッダー名は空でないASCIIのtokenとして検証し、条件を満たさなければ不正出力として502にする。英数字とtokenで許可された記号を受理し、名前の補正は行わない |
| CGI-56 | CGI出力ヘッダーの基本構文 | 各ヘッダー行を最初のコロンで名前と値に分ける。コロンがない行、ヘッダー終端の完全に空の行がないままEOFになった出力は502とする |
| CGI-57 | CGI出力ヘッダーの改行詳細 | 行ごとにLFまたはCRLFを受理し、混在も許容する。LFを伴わない単独CRは502とする。本文の改行は変更しない |
| CGI-58 | CGI出力ヘッダー値の空白・文字 | 値の前後SP/HTABを除去し、内部の空白・タブと大小文字は保持する。HTAB以外の制御文字とDELを含む値は502とし、非ASCIIのバイトは文字コード変換せず保持する。各ヘッダーの値の構文検証も行う |
| CGI-59 | CGI出力ヘッダーの空値 | 前後SP/HTAB除去後に空ならヘッダー省略と同じ扱いとする。ただし重複検査を先に行う。通常の文書応答でContent-Typeが空なら必須ヘッダー欠落として502とする |
| CGI-60 | CGI出力のContent-Type検証 | type/subtypeと、付いている場合のパラメーターの構文を検証し、不正なら502とする。未知の種類も構文が正しければ受理する。本文の形式推測・文字コード変換・登録一覧との照合は行わない |
| CGI-61 | CGI出力のStatus検証 | 非空のStatusはASCII数字3桁のコード、区切りのSP、説明文の形式とし、説明文は空でもよい。コードの受理範囲は100〜599とする。構文不正・範囲外は502。説明文は検証後にHTTPステータス行へ使い、区切りSPは末尾空白除去前に確認する。1xxだけで終了するCGIはCGI-74に従い502とする |
| CGI-62 | Locationの値・受け渡し | 非空のLocationは任意のフラグメント付きスキーム付き絶対URIとして一般構文を検証する。HTTP/HTTPSには限定しない。空白・不正な%表記等は502。前後空白除去後の値をそのまま使い、URLデコード・パス正規化・DNS照会・宛先への接続確認は行わない |
| CGI-63 | Location単独の応答 | Location以外の有効なヘッダーがなく本文が空なら、Locationを伴う302へ変換する。空値の省略と重複検査は既存規則に従う |
| CGI-64 | Locationと文書を伴う応答 | Location・Status・Content-Typeを要求し、対応するStatusは301・302・303・307・308に限定する。指定コードと本文を返し、本文は空でもよい。その他の応答ヘッダーはCGI-69〜72の規則で扱う |
| CGI-65 | Location応答の不完全な組み合わせ | Locationを含む応答で、Location単独またはLocation・Status・Content-Typeの形式に合わなければ502とする。欠けたヘッダーやStatusを補完しない |
| CGI-66 | 内部・相対リダイレクトの出力 | /other・other・//example.com/等、スキームのない非空Locationは502とする。サーバー内部で再処理せず、絶対URIへの補完や外部向け302への変換も行わない |
| CGI-67 | CGI出力のContent-Length数値 | 非空値はASCII数字だけの非負整数として検証する。0・先頭ゼロは受理し、符号・カンマ・内部空白・数値の桁あふれは502とする。宣言値とCGI本文の実長を照合し、不一致も502とする |
| CGI-68 | HTTP応答のContent-Length生成 | CGIのContent-Lengthを直接コピーせず、Webservが実際に送る本文のバイト数から生成する。本文を送れないステータスはCGI-73に従う |
| CGI-69 | CGI出力全ヘッダーの重複 | CGI出力のすべてのヘッダーを名前の大小文字を区別せず最大1行とする。同値・空値を含む重複も502とし、結合・先勝ち・後勝ちは実装しない。複数のSet-Cookieも最小構成では対象外とする |
| CGI-70 | CGI出力の接続用ヘッダー | Connection・Keep-Alive・TE・Upgrade・Proxy-ConnectionとConnectionに列挙されたヘッダーを転送しない。ただしStatus・Content-Type・Location・Content-Lengthが列挙されていれば矛盾として502。WebservがConnection: closeを付ける |
| CGI-71 | CGI出力のTransfer-Encoding | 非空のTransfer-EncodingをCGIが出したら502とする。CGI出力をchunkedとして復号する処理は実装しない。入力要求のchunked復号は既存方針を維持する |
| CGI-72 | その他のCGI出力ヘッダー | 専用の処理・除外対象以外は、構文と値の検証を通ったヘッダーを値を保持して転送する。Webserv側で同名ヘッダーを二重に追加しない。エラー本文置換時はCGI-131の除去・再生成を適用する |
| CGI-73 | 本文を送れないCGI応答 | 204・205・304でCGI本文があれば502。本文が空なら204・304にはHTTPのContent-Lengthを付けず、205にはContent-Length: 0を付ける。CGIの長さ指定があれば先に通常の実長照合を行う |
| CGI-74 | CGIの1xx応答 | Statusのコードの数値検査は100〜599を維持するが、1xxだけで終了するCGI出力は最終応答にできないため502とする。中間応答と最終応答を複数回扱う仕組みは実装しない |
| CGI-75 | 入力ヘッダーの転送対象の確定 | 合意済みの名前・除外・重複・空値の規則を満たす要求ヘッダーをHTTP_*へ転送する。Accept等の名前を個別登録する許可一覧は作らない |
| CGI-76 | Trailer・Expectの除外 | 要求のTrailer・ExpectをCGIへ転送せず、HTTP_TRAILER・HTTP_EXPECTを設定しない。HTTP/1.1のExpectは既存の417でCGIを起動せず、HTTP/1.0では値によらず無視する |
| CGI-77 | 要求trailerの扱い | chunked本文の後のtrailerはHTTP層で読み取り・構文検査するが、CGIの環境変数へ追加せず、先に届いたヘッダーも上書きしない |
| CGI-78 | 要求Connectionの検証・重複 | 値をカンマ区切りのヘッダー名として検証し、不正な非空の名前は400、空の要素は無視する。複数行なら全行に列挙された名前をCGI-44のHTTP_*除外対象とする。本文受信・専用環境変数の規則は維持する |
| CGI-79 | 要求ヘッダー値の共通文字検査 | 要求ヘッダー共通の検査としてHTAB以外の制御文字とDELを400にする。非ASCIIのバイトは変換しない。この検査を本文へ適用しない |
| CGI-80 | CGI要求のContent-Type検証 | CGIへの要求のContent-Typeは最大1行で、同値・空値を含む重複は400とする。値は専用変数へ渡し、本文の形式やパラメーターの解釈はCGIに任せる。空値はCONTENT_TYPEを空文字列で設定し、ヘッダーなしなら省略する |
| CGI-81 | 転送除外対象の重複 | CGIへ転送しないという理由だけで、除外対象ヘッダーの重複を一律に400にしない。Host・Content-Length等の個別検証と、Connection・Content-Typeの規則は維持する |
| CGI-82 | 要求のContent-Encoding | Content-Encodingは既存の転送規則を満たせばHTTP_CONTENT_ENCODINGへ渡す。gzip等の内容符号化をWebserv側で展開せず、CGI本文はchunked等の転送符号化の復号だけを済ませて渡す |
| CGI-83 | CGI環境へ渡す要求情報のNUL | 環境変数へ渡す要求情報に実NULバイトが残る場合はCGI起動前に400とする。未デコードのクエリの文字列%00はそのまま渡せる。URLパス中の%00を400にする既存規則と、本文をバイト列として渡す方針は維持する |
| CGI-84 | CGIファイルパスの結合 | Configで解決済みの絶対root/aliasと、C-24・C-25で1回デコード・正規化したURLパスを使う。rootはURI全体を追加し、結合境界で重なる/を1個にする。aliasはlocation prefixを置換し、C-09のprefix・alias値の末尾/必須を維持する |
| CGI-85 | スクリプトとPATH_INFOの検証範囲 | CGI-25の左からの探索で、設定拡張子に一致する最初の実在する通常ファイルを実行対象とする。以後のPATH_INFOはファイルパスとして存在・権限検査しない |
| CGI-86 | CGI領域のファイル種別 | statで種別を確認する。ディレクトリはスクリプトとして実行せず、既存のディレクトリ・index処理へ進む。通常ファイルでもディレクトリでもないFIFO等は403とする |
| CGI-87 | スクリプトの権限検査 | access(path, R_OK)で読み取り可能かを確認する。Pythonが読み込むため、スクリプトのX_OKは要求せず、内容の構文検査・試し実行は行わない |
| CGI-88 | Pythonの要求時検証 | Config起動時の通常ファイル・X_OK検査(C-40)を維持し、要求ごとのPythonの事前検査は追加しない。実際のexecve失敗は既存の502とする |
| CGI-89 | ファイル検査後の変化 | 検査後のファイル消失・権限変更等でも再検査・再起動によるリトライは追加しない。起動処理の失敗・Pythonの異常終了は既存のCGIエラー規則に従う。子のchdir等の失敗検出方法はCGI-120〜127の現時点の採用案に従う |
| CGI-90 | stat/access失敗の分類 | 失敗直後のerrnoでENOENT・ENOTDIRは404、EACCESは403、ENAMETOOLONGは414、その他は500とする。read/write(recv/send)後のerrno禁止と区別する |
| CGI-91 | CGIの担当境界 | rysatoが要求検証・実行対象とcwdの解決・環境変数内容の生成・出力解析とHTTP応答生成を担当する。tasugiyaが接続情報取得・起動・pollでのパイプI/O・上限と期限・子の終了と回収を担当する |
| CGI-92 | CGI実行要求の型 | CgiRequestはPython絶対パス・スクリプト絶対パス・cwd・NAME=valueの環境変数一覧・復号済み本文を保持する。プロセス側でHTTP/Configを再解釈せず、argvはPythonとスクリプトの2つから生成する |
| CGI-93 | 接続情報の受け渡し | ConnectionInfoで接続元IPv4、実際のサーバーIPv4、実際のlistenポートをtasugiyaからrysatoへ渡す。IPv4は4オクテットの十進表記とし、DNSやinet_ntop/inet_ntoaは追加しない |
| CGI-94 | 実行結果の型 | CgiResultはstdout全体とerrorStatusを保持する。0は実行成功、非0はサーバー側のエラーコード。子の終了状態はtasugiya側で解釈し、失敗時の部分出力をHTTPへ送らない。出力の妥当性はrysato側が別途検証する |
| CGI-95 | Routerの受け渡しAPI | route(request, config, connection, response, cgi)はRESPONSE_READYまたはCGI_REQUIREDを返し、対応する出力引数だけを有効とする。parseCgiOutput(output, response)は成功0・不正出力502を返し、成功時だけresponseを有効とする |
| CGI-96 | プロセス起動・結果取得API | start(const CgiRequest&, CgiProcess&)は開始成功0・失敗時はエラーコードを返し、CGI完了を待たない。takeResult(CgiResult&)は未確定false、確定結果を一度だけtrueで返す。所有権はCGI-99〜106、後始末はCGI-107〜118、失敗検出は9/10の現時点の代替案に従う |
| CGI-97 | 要求ヘッダー全行の公開 | HttpRequest::getHeaders()は小文字名・前後SP/HTAB除去後の値の組を全行保持したvectorへのconst参照を返す。重複を辞書の上書き・結合で失わず、転送対象の検証とConnection全行処理に使う |
| CGI-98 | CGIのStatus説明文の反映API | HttpResponse::setStatus(int, const std::string&)で検証済みのCGI説明文を空も含め保持する。setStatus(int)はサーバー生成応答の標準文言に使う |
| CGI-99 | CGI実行要求の寿命 | EventLoopの一時的なCgiRequestをrouteからstartが返るまで保持する。startは親側にrequestの参照・ポインターを残さず、返却後は要求を破棄できる |
| CGI-100 | CGI本文の所有権 | startが本文をCgiProcessのstringへコピーする。入力全体とsize_tの書き込み位置を持ち、部分書き込みでは位置を進める。書き込み完了・打ち切りで不要になった本文バッファを空stringとのswapで解放する |
| CGI-101 | argv/envpの管理 | CgiRequestの文字列を先に完成させ、tasugiyaがfork前にPython・script・NULLのargvと、NAME=valueのポインター一覧・末尾NULLのenvpを作る。配列作成後に元の文字列/vectorを変更せず、文字列ごとのnew[]・手動解放を追加しない |
| CGI-102 | 起動情報の親子での寿命 | 子はfork時のパス・cwd・環境をexecve成功または起動失敗による終了まで保持する。親はstartが返れば一時データを解放でき、CgiProcessに環境変数のコピーを保持しない |
| CGI-103 | stdoutと結果の所有権 | CgiProcessがstdoutを蓄積し、takeResult成功時にswapでCgiResultへ渡す。失敗時は部分出力を破棄しstdoutDataを空にする。falseなら出力引数を変更せず、trueなら既存内容を置き換え、一度だけ取得できる |
| CGI-104 | CGI結果とHTTP応答の寿命 | EventLoopがCgiResultをparseCgiOutputの間保持する。HttpResponseは生成したヘッダー・本文を自分の値として保持し、解析後はCgiResultを破棄できる。ヘッダーのconst参照はroute内で環境変数へコピーし、CGI実行中に残さない |
| CGI-105 | ClientとCGIの所有関係 | EventLoopがClientとCgiProcessを所有する。CgiProcessは対応先Clientへの非所有ポインターを持ち、Client破棄前に関連付けをNULLへ外す。fdだけで返却先を探さず、切断後の結果を解析・送信しない |
| CGI-106 | CgiProcessのコピー禁止 | C++98のprivateな未定義コピーコンストラクター・代入演算子でコピー不可とし、pid/fdの所有者を増やさない。EventLoopの管理表にはポインターを保持し、CgiProcessがClientを削除する処理は作らない |
| CGI-107 | 正常終了時の後始末 | 親側stdinパイプは入力完了で閉じる。stdoutはEOFまで読み、子の終了・回収も確認してから結果を返す。子が先に終了しても未読stdoutを捨てず、後始末と結果の引き渡し/破棄後にCgiProcessを削除する |
| CGI-108 | CGI中止時の後始末 | 切断検出・タイムアウト・上限超過・I/O異常では、最初の中止/失敗理由を保持し、パイプ監視を外して閉じ、未回収の直接の子を必要に応じSIGKILLで停止・回収する。切断後は応答せず、残る接続には回収後に確定済みエラーを返す |
| CGI-109 | 子の回収とpidの管理 | EventLoopだけがpid>0の直接の子にwaitpid(pid, &status, WNOHANG)を呼ぶ。0なら次のループへ進み、ブロッキング待機・SIGCHLDハンドラー/自動回収は追加しない。回収済みpidをkillせず、停止成功後のkillを繰り返さない |
| CGI-110 | 回収・closeの失敗処理 | waitpid失敗のEINTRは次のループで再試行、ECHILDは回収対象なしとして以後killせず管理エラー500、その他は500を保持して再確認する。既存の失敗理由を上書きしない。killのESRCHだけで回収済みとしない。Linuxのcloseは保持fdを-1にして一度だけ呼び、再試行しない |
| CGI-111 | 回収待ちとpoll周期 | CGI実行中・回収待ちは単一EventLoopのpoll待ち時間を最大100 msとし、I/O通知がなくても終了・期限を定期確認する。別pollループやデストラクターでの待機を追加しない |
| CGI-112 | CGIパイプのイベント処理 | stdoutのPOLLHUPでもpoll後に読んでreadの0をEOFとする。各対象fdに1回ずつI/Oし、残りは次のpollへ戻す。read負・未送信本文へのwriteが0以下・POLLERR/POLLNVAL等は502で打ち切り、read/write後のerrno分岐・再試行はしない |
| CGI-113 | CGI待機中の受信EOF | 要求完成後のrecvの0だけで完全切断と判断せず、受信EOFを記録してPOLLIN監視を止め、CGIを継続する。POLLHUP/POLLERR・送信失敗等で接続が使えないと分かった時に中止する |
| CGI-114 | CGI待機中の追加受信 | WAITING_CGI中もクライアントのpoll通知を処理する。POLLIN時の追加データは固定サイズの一時バッファで読み捨て、次の要求として蓄積・解析しない。1接続1要求を維持する |
| CGI-115 | 古いpoll通知と削除時期 | イベント処理前に管理オブジェクトと現fdの有効性を確認し、閉じたものをスキップする。Client/CgiProcessの削除は現在のpoll結果の処理後まで遅らせ、解放済みオブジェクト・再利用fdを古い通知で操作しない |
| CGI-116 | CGIのfd継承 | listen・acceptソケットと全CGIパイプに生成時F_SETFD/FD_CLOEXECを設定する。子はstdin/stdoutをdup2で0/1へ接続し元の不要端を閉じ、0/1/2のFD_CLOEXECを解除して標準入出力・stderrを継承する。不要fdはexecve成功時に閉じ、設定失敗は成功扱いにしない |
| CGI-117 | 親子のシグナル設定と対象 | 親のSIGPIPE無視を維持し、子はexecve前にSIGPIPEをSIG_DFLへ戻す。SIGCHLDをSIG_IGNにしない。管理対象はWebservが直接起動したCGIの子とし、孫プロセス列挙・プロセスグループ管理は追加しない |
| CGI-118 | SIGINT終了時のCGI回収 | 新規受付・CGI起動を止め、listen/Clientを閉じて関連付けを外し、実行中CGIを打ち切る。同じEventLoopの終了段階で全直接の子を回収してからWebservを終了し、未回収のままループを抜けない |
| CGI-119 | 使用不可の関数 | ユーザー確認により_exitとclock_gettimeはともに使用不可とする。子の終了・時間計測はこの制約の下でCGI-120〜127の現時点の採用案に従う |
| CGI-120 | 起動・計時の代替案の位置付け | CGI-121〜127は現時点で採用する代替案とし、実装・実行環境・動作確認の結果によって方法や具体的な処理を見直し得る。動作確認済み・最終固定とは扱わず、変更時は理由・影響・検証結果を関連資料に揃えて記録する。_exit/clock_gettime使用不可の制約は維持する |
| CGI-121 | 子の起動失敗とmainへの戻り方(現時点の採用案) | 子のSIGPIPE復元・dup2・fd設定/close・chdir・execve等の失敗では、子フラグを用いてmainまでreturnで戻る。isChildProcessを各呼び出し元が最初に確認し、HTTP応答・残りのイベント処理・親用cleanupへ進まない。起動時の一時fdと自動変数を解放し、mainでCGI-123の子専用解放後にreturn 127で終了する |
| CGI-122 | 起動失敗の親への通知(現時点の採用案) | 親はwaitpidで終了状態を確認し、非0の終了コード・シグナル終了は502とする。既存の504等を上書きしない。Python自身の127も502でよいため、起動失敗専用の通知パイプは追加しない |
| CGI-123 | return終了時の明示的な解放とバッファ(現時点の採用案) | 子の起動失敗ではEventLoop::cleanupInChild()で子に複製された管理対象fd・Client/CGI/listen管理データを解放し、mainでEventLoopをdeleteしてからreturn 127する。kill/waitpid/shutdown・親用cleanup・応答/ログ処理は実行しない。所有データと一時資源をfork前に追跡可能にし、デストラクターは停止・回収を行わず資源解放だけとする。fork前にC++出力バッファを空にし、子では追加確保・例外脱出・ログ出力を行わない |
| CGI-124 | 計時の取得元(現時点の採用案) | Linuxの/proc/uptimeの最初の非負小数をopen/read/closeで取得し、有限のdoubleへ変換する。毎回開き直して固定サイズ256バイトで読み、lseek・時刻関数・外部コマンド・追加Config項目を設けない。Linuxで/procが読めることを前提とする |
| CGI-125 | 10秒の判定(現時点の採用案) | fork直前の取得値を開始値とし、EventLoopの各周1回の現在値を全CGIで共有して差が10.0秒以上なら504とする。途中のI/Oで延長しない。小数2桁の表示精度と最大100 msのpoll待ちに従って検出し、10秒ちょうどの実時間保証とはしない |
| CGI-126 | 計時失敗(現時点の採用案) | 開始前の取得失敗は500で起動せず、実行中の取得失敗・値の逆行は未完了CGIを500で打ち切る。既存の失敗理由を保持し、取得不能時に無期限で続けるフォールバックは追加しない |
| CGI-127 | fork前の準備と失敗(現時点の採用案) | 本文コピー・argv/envp・パイプ・親側ノンブロッキング/CLOEXEC設定はfork前に準備する。親側pipe/fork/fcntl失敗は500とし、作成済み資源を後始末する。子が作成済みなら8/10に従い回収する |
| CGI-128 | 受け渡しの順序 | rysato側のrouteが即時応答を返した場合はCGIを起動しない。tasugiya側のstart失敗はそのコードでmakeErrorを生成する。非同期のCgiResultはerrorStatusが非0なら部分stdoutを解析せずmakeErrorへ渡す。0ならrysato側のparseCgiOutputで全出力を検証し、解析失敗は502、成功はCGIの応答として扱う。失敗時のresponse出力引数は使用しない。 |
| CGI-129 | 失敗コード | スクリプト未発見404・読み取り不可403・パス長414等の既存分類を維持する。親側のpipe/fork/fcntl・計時・管理の失敗は500、子側の準備/exec失敗・非0終了/シグナル終了・パイプI/O失敗・出力上限超過・不正出力は502、10秒経過は504。複数の原因が発生しても最初に確定したコードを維持する。正常に返されたStatus: 404等はCgiResultの実行失敗ではない。 |
| CGI-130 | error_pageの適用 | 出力検証後の最終コードが400〜599なら、CGIが正常に返したエラーも含め、指定ファイルの本文へ置き換える。未指定・読み取り失敗なら内蔵HTMLにする。元のコードを維持し、有効なCGIのStatus説明文は空文字列も保持する。200〜399には適用しない。1xxは既決定のCGI-74により先に502となる。不正出力をerror_pageで覆って元のCGIコードのまま受理しない。 |
| CGI-131 | 置換する本文とヘッダー | エラーページはHTMLとして扱い、Content-Typeはtext/html、Content-Lengthは置換後のバイト数で生成する。元本文の情報を残さないため、CGI由来のContent-*、ETag、Last-Modified、Digest、Repr-Digestを大小文字を区別せず除去する。この一律除去は最小構成のチーム方針。その他の既存転送対象ヘッダー(Allow、WWW-Authenticate、Set-Cookie等)は保持する。200〜399の応答にはこの除去を行わない。 |
| CGI-132 | エラーページの読み取り | 設定済みのファイルパスをそのまま使い、実行時に通常ファイルであることを確認して読む。空ファイルも読み取り成功なら空本文として採用する。失敗時は元コードの内蔵HTMLへ一度だけフォールバックし、再ルーティング・CGI実行・再帰的なエラー生成を行わない。CGIの8 MiB制限は元のstdoutに適用し、エラーページに新しいCGI専用上限は追加しない。 |
| CGI-133 | 内部の共通処理 | HttpResponseにvoid applyErrorPage(const ServerConfig& conf)を追加し、400〜599の場合だけ上記の本文置換を行う。makeErrorは同じ処理を内部で利用する。EventLoopはparseCgiOutputが成功した応答に一度だけapplyErrorPageを呼び、makeErrorの結果には再適用しない。ファイル読み取りと内蔵HTMLへのフォールバックをCGI専用に重複実装しない。 |
| CGI-134 | 応答を返せない場合 | クライアント切断時は停止・回収だけを行い、応答を生成しない。送信失敗後に別のエラー応答を追加送信しない。listen側のCLOEXEC設定失敗は起動失敗、acceptしたfdの設定失敗はその接続を閉じて監視へ登録しない。CGIパイプの設定失敗は上記の親500/子502の分類を使う。 |
| CGI-135 | 結合の完了条件 | 04のCGI受入項目を実装後に満たすことを条件とする。正常なGET/POST・chunked入力、入出力の並行処理、出力検証・エラーコード・上限・期限、切断/半閉鎖/SIGINT、fd継承・子の回収・寿命、CGI実行中の静的配信を確認する。加えて正常なCGIの404/599の置換、200の非置換、エラーページ未指定/読取失敗/空ファイル、元本文の符号化等の除去とAllow等の保持、不正な404出力が502になることを確認する。9/10の代替案は実環境で検証し、必要なら方法を見直す。10項目の合意完了と、未実装の結合試験合格は別に扱う。 |

詳しい規則の説明は [03 CGI](reports/03_config_cgi_upload.md#cgi-details)、実装後の確認項目は [04 検証計画](reports/04_testing_and_workplan.md) を参照する。

## 合意状況と次の設計判断

- ConfigはC-01〜42まで合意済み。詳細・設定例・公開APIは [05](reports/05_config_agreement.md#config-decisions) にまとめる。
- CGIはCGI-01〜135まで合意済み。CGI-120〜127の起動・計時方法は現時点の採用案で、実装・実際の動作で見直し得る。子の失敗時の明示的な資源解放と、親用の停止・回収処理を実行しない方針は維持する。
- HTTPの1/10〜6/10はHTTP-01〜44で合意済み。7/10はstd::removeによる削除を含めHTTP-45〜50で合意済み。8/10の応答生成はHTTP-51〜58で合意済み(Date省略は暫定案)。9/10のHTTP API/データの保持はHTTP-59〜64で合意済み。10/10のネットワーク境界・計時・通常ファイルI/OはHTTP-65〜74で合意済み(採用案を含む)。10項目の動作方針の判断は完了し、削除手段も確定した。Dateは暫定、接続情報取得失敗時の扱いは未確定、採用案の実環境検証は未実施。残件は [04の10項目](reports/04_testing_and_workplan.md#remaining-design) で管理する。

ここでの「合意済み」は設計判断の状態を示す。実装・受入試験の完了とは区別する。

## Config合意事項

Configの決定表C-01〜42、詳細規則、設定例、型・API・寿命の正本は [05 Configの設計と合意事項](reports/05_config_agreement.md#config-decisions) とする。本書には同じ本文を再掲しない。

root/aliasの区別、設定ファイルを基準とした相対パス、限定文法、既定値、起動時検証は合意済み。error_pageは100〜599の設定を受理し、本文置換は400〜599だけに適用する。HTTPの未決定事項をConfigParserの処理として追加しない。

---

# クラス設計(全13クラス+2表)

Configの型・parse API・getter・所有関係はC-42で確定済み。HTTPの責務・公開API・保持はHTTP-59〜64で合意済み。I/O・資源管理はHTTP-65〜74に従い、検証後に見直し得る採用案と未確定の内部実装を区別する。CGI受け渡しの公開型・シグネチャ・担当境界はCGI-91〜98、所有権・寿命はCGI-99〜106、後始末はCGI-107〜118で確定済み。起動失敗検出・計時はCGI-120〜127を現時点の代替案として採用し、実装・動作確認で見直し得る。モジュール一覧の `CgiHandler` はCGI処理の総称で、この案では `CgiExecutor` / `CgiProcess` 等に分ける。

## 一覧と責務

### 設定ドメイン(rysato担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ConfigParser` | ファイル→トークン化→構文解析→既定値補完/パス解決/検証。Configを値で返し、設定エラーはruntime_errorをmainへ伝える(C-42) | 起動時1回・使い捨て |
| `Config` | パース結果の最上位入れ物 | 起動時1回・終了まで不変 |
| `ServerConfig` | server ブロック1個分のデータ | 同上 |
| `LocationConfig` | location ブロック1個分のデータ | 同上 |

※ Config / ServerConfig / LocationConfigはprivateメンバーとconst getterを持つ読み取り専用データクラス。設定値の解析・検証はConfigParserが行う(C-42)。

### ネットワーク/イベントドメイン(tasugiya担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ServerSocket` | socket→setsockopt→bind→listen→ノンブロッキング化(listen先のhost:port 1組分) | 起動時・listen先数分 |
| `EventLoop` | poll 本体。fd 登録簿、イベント配送、タイムアウト見回り、終了時の後始末 | 起動時1個 |
| `Client` | 接続1本の状態機械(バッファ・状態・各段階の開始時刻、HTTP-65〜70) | accept 時に生成 / close 時に破棄 |
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
| `CgiExecutor` | pipe/fork/dup2/chdir/execveによる起動と環境変数の受け渡し。tasugiyaがプロセス制御、rysatoが環境変数の内容を担当する(CGI-91〜98)。所有権はCGI-99〜106、子の失敗時の明示的解放はCGI-123。起動失敗検出等は9/10の現時点の代替案に従う |
| `Logger` | `info/warn/error`。レベルフィルタ。ファイル出力なし |
| MIME タイプ表 | 拡張子→Content-Type。static 関数1個 |
| ステータスコード表 | code→文言。HttpResponse 内の static 関数 |

## 所有関係

```
main
 └─ Config(不変・全員が const 参照で覗く)
 └─ EventLoop
     ├─ Router(1個、要求ごとのデータを保持し続けない)
     ├─ ServerSocket × listen先数
     ├─ Client × 接続数          ← accept 時 new / close 時 delete
     │   └─ HttpRequest(値として内包)
     │   └─ send_buffer(serialize 済み Response)
     └─ CgiProcess × 実行中CGI数  ← EventLoopが生成・所有 / 後始末・結果受け渡し後に破棄
         └─ 対応先Clientへの非所有ポインター(Client破棄前にNULLへ外す、CGI-105)
```

HttpResponseは生成したヘッダー・本文・ステータスを自分の値として保持する(CGI-98・104、HTTP-64)。Routerの寿命と結果はHTTP-63、CgiExecutorの担当・コピーと寿命は03の合意に従う。残る内部実装は以下の設計案と区別する。

## 主要インターフェース

### HttpRequest

```cpp
class HttpRequest {
public:
    enum State { INCOMPLETE, COMPLETE, ERROR };

    explicit HttpRequest(std::size_t maxBodySize);

    // Client(tasugiya側)が呼ぶ
    void   appendData(const char* data, std::size_t len); // recv断片をバイト数付きで投入
    State  getState() const;
    int    getErrorCode() const;      // ERRORの確定コード、それ以外は0

    // Router(rysato側)が COMPLETE 後に呼ぶ
    const std::string& getMethod() const;
    const std::string& getPath() const;         // query分離/1回デコード/正規化済みURLパス
    const std::string& getQueryString() const;  // ? の後ろ、生のまま
    const std::string& getVersion() const;     // 受信したHTTP/1.xの表記(HTTP-04)
    std::string        getHeader(const std::string& key) const;  // 大小無視
    const std::string& getBody() const;         // un-chunk 済み
    const std::vector<std::pair<std::string, std::string> >& getHeaders() const;
                                               // 全行・重複を保持(CGI-97)
};
```

- 外部の状態はHTTP-60の3つ、内部の要求行/ヘッダー/本文/chunked解析段階は非公開。生成時の本文上限を変更しない。COMPLETE/ERROR後は結果を変えない(HTTP-61)
- エラー判定(400/413/417/501等)はパース時点でHttpRequest自身が行い、ソケットI/Oや応答生成は行わない
- ヘッダー完了時に受信バージョン別のHost・Transfer-Encoding・Expectを検証する。拒否する場合はボディを待たずERRORへ遷移し、Clientがエラー応答を送る
- ClientはappendDataのたびに状態を確認する。ERRORになった要求のボディ受信を続けてCOMPLETEを待たない
- ヘッダーキーは内部で小文字化して格納。getHeadersは全行を保持し、単一値getterのために重複を上書きして失わない(CGI-97)
- chunked デコードはこのクラスに閉じる。HTTP/1.1要求でのみ受理し、CGIには復号済みボディを渡す

### HttpResponse

```cpp
class HttpResponse {
public:
    HttpResponse();
    void setStatus(int code);   // reason phrase は内部表で自動解決
    void setStatus(int code, const std::string& reason); // CGI説明文を空も保持(CGI-98)
    void setHeader(const std::string& key, const std::string& value);
    void setBody(const std::string& body, const std::string& contentType);
                                // 通常の本文付き応答のContent-Length / Content-Typeを設定
    std::string serialize() const;
                                // HTTP-51〜58: HTTP/1.0・Connection: close。Serverを生成せずDate生成も暫定的に省く
                                // CGIの204/304は本文・Content-Lengthなし、205は本文なし・長さ0(CGI-73)
    void applyErrorPage(const ServerConfig& conf); // 400〜599だけ本文・関連ヘッダーを置換(CGI-133)
    static HttpResponse makeError(int code, const ServerConfig& conf);
                                // error_page 指定 or 内蔵デフォルトHTML
};
```

- `\r\n` の組み立ては serialize() に封じ込める
- Transfer-Encodingは出力しない。本文を送れないステータスはserialize時にも検証し、setStatus/setBodyの呼び出し順にかかわらず正しい形式にする
- 送信完了後はClientをCLOSINGへ遷移させる。次の要求を受けるための再初期化は行わない
- サーバー側エラー生成はmakeErrorに一元化。HttpRequestはコードだけを返し、Client/EventLoop側が応答を作る。Routerの即時拒否もmakeErrorを使う。makeErrorは内部でapplyErrorPageを使う。正常なCGI応答はparseCgiOutput成功後にEventLoopがapplyErrorPageを一度だけ呼ぶ。コード・CGI説明文を保持し、400〜599だけ本文を置換する(CGI-128〜133)
- serialize()は送信開始前に1回だけ呼び、生成文字列をClientのsendBufferへswapで渡す。Clientは送信済みオフセットを保持し、元のHttpResponseの寿命に依存しない(HTTP-64)
- Routerの即時応答/CGI要求の戻り値と非同期結果の受け渡しはCGI-91〜98で確定済み。所有権・寿命はCGI-99〜106で確定済み。後始末はCGI-107〜118で確定済み。起動失敗検出・計時は03の9/10の現時点の代替案(CGI-120〜127)を採用し、動作確認で見直し得る。

### CGIの共有型・受け渡しAPI

ConnectionInfo / CgiRequest / CgiResult / RouteResultと、Router::route / parseCgiOutput、CgiExecutor::start、CgiProcess::takeResultはCGI-91〜98で合意済み。宣言と担当境界は [03の共有API](reports/03_config_cgi_upload.md#cgi-api) を参照する。

### Config 系

Config / ServerConfig / LocationConfigの型、getter、ConfigParser::parse、エラー処理、寿命はC-42で合意済み。公開APIは [05の型とAPI](reports/05_config_agreement.md#config-api) を参照する。

最長前方一致・マッチなし404は確定済み。findLocation等はRouter内部の実装詳細で、必要な公開APIを追加する未決定事項とは扱わない。

### EventLoop / Client / CgiProcess(tasugiya側内部)

```cpp
class EventLoop {
public:
    EventLoop(const Config& conf);
    ~EventLoop(); // fd・所有データの解放のみ。kill/waitpid/shutdownは行わない
    void cleanupInChild(); // CGI起動失敗の子専用。CGI-123の現時点の採用案
    void run();   // SIGINT後は受付停止・CGI停止/回収を終えてから戻る(CGI-118)
private:
    void rebuildPollfds();          // 毎周再構築(決定事項)
    void handleListenEvent(int fd);
    void handleClientEvent(Client& c, short revents);
    void handleCgiEvent(CgiProcess& p, short revents);
    void checkTimeouts();           // 受信/送信60秒・切断待ち1秒 / CGI起動から10秒、延長なし
    void cleanup();                 // 親専用: CGI停止・非ブロッキング回収を含む(CGI-107〜118)
                                    // 子はこの経路へ入らずcleanupInChildで複製資源だけを解放
};

class Client {
public:
    enum State { READING_REQUEST, PROCESSING, WAITING_CGI,
                 WRITING_RESPONSE, DRAINING, CLOSING };
    // fd, state, sendBuffer, sendOffset, HttpRequest(値), 計時情報,
    // recv断片はHttpRequestへ渡し、要求全体の別バッファは持たない(HTTP-62)
    // 所属 const ServerConfig* を参照(所有しない、C-42)
};

class CgiProcess {
public:
    CgiProcess(); // 無効pid・fd=-1・Client=NULL・空バッファで初期化
    ~CgiProcess(); // 所有fd・バッファ解放のみ。kill/waitpid/shutdownは行わない
    bool takeResult(CgiResult& result); // CGI-96、結果を一度だけ取得
private:
    CgiProcess(const CgiProcess&);            // 未定義、CGI-106
    CgiProcess& operator=(const CgiProcess&); // 未定義、CGI-106
    // pid, stdinFd(書込), stdoutFd(読取), 残り書込ボディ,
    // 出力蓄積バッファ, 開始時刻, stdoutEOF, 子終了状態, 失敗理由,
    // 対応先Clientへの非所有ポインター(CGI-105)
    // 読み取り中にヘッダー8 KiB・stdout全体8 MiBの上限を検査
};
```

### CgiExecutor / Logger

CgiExecutorのstart APIは [03](reports/03_config_cgi_upload.md#cgi-api)、isChildProcessは同書の項目9/10を参照する。Loggerの形式はHTTP-73で合意済み。以下の宣言は内部実装案で、暦日時の取得・書式化関数は追加しない。

```cpp

class Logger {
public:
    enum Level { DEBUG, INFO, WARN, ERROR };
    static void setLevel(Level l);
    static void debug(const std::string& msg);
    static void info (const std::string& msg);
    static void warn (const std::string& msg);
    static void error(const std::string& msg);
    // 形式: [LEVEL] message (HTTP-73)
};
```
