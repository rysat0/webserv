# webserv 要件・決定事項一覧

[課題文（Version 24.1）](webserv.pdf)の要件と、チームで採用した設計を記録します。実装に着手する場合は、[実装の始め方](reports/06_implementation_guide.md)から読むと、必要な規則やAPIにたどり着けます。

- §1〜13には課題の要件をまとめています。
- 「チーム決定事項」以降には、担当、実装順、HTTP・CGIの方針、クラス間の契約を記載しています。日付付きの更新で変わった箇所は、新しい契約に従います。
- 実装後に確かめる動作と、検証結果に応じて見直す採用案は[試験項目と進行計画](reports/04_testing_and_workplan.md#remaining-design)を参照してください。

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

<a id="submission-readme"></a>

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

以下はチーム全体の結合順です。HTTP担当の単体実装は、ネットワーク側の準備と並行して進められます。[実装の始め方](reports/06_implementation_guide.md)

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
| 例外方針 | 通常の実行時失敗は戻り値方式。接続処理の例外は捕捉して資源を回収する。2026-11-15合意によりstd::bad_allocは500の再生成/追加確保による復旧をせず当該接続をcloseし、ほかの接続を継続する。捕捉位置と回収経路は実装設計で具体化する |

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

<a id="autoindex-incremental-design"></a>

#### autoindex生成の段階分割(2026-11-19合意)

autoindex生成は、列挙・ソート・HTML生成の途中状態を保持し、各段階で一定量を処理したらEventLoopへ戻る方式を採用する。Routerの1回の呼び出しで一覧全体を完成させる構成から、後続の周へ継続する構成へ変更する。

| 段階 | 保持・処理する内容 |
|---|---|
| 列挙 | 開いているディレクトリと取得済み項目名を保持し、1回のstepで最大32回のreaddirを進める採用案とする。既存の除外項目とHTML完成長の検査を適用し、正常終端後にclosedir成功を確認してソートへ進む |
| ソート | 取得した元の名前をHTTP-28の符号なしバイト列の昇順に並べる。ボトムアップのマージソートで途中位置を保持し、比較/移動を分割する。具体的な処理量は次に決める |
| HTML生成 | ソート済みの名前から表示名とリンクを生成し、出力途中の位置とHTML本文を保持して一定量ずつ進める |

HTTP-27〜32の表示対象・名前/リンクの変換・8 MiBの完成HTML上限・失敗分類を適用する。列挙中から完成時のHTML長を確認し、超過時は打ち切って既存の回収/500経路へ進む。全列挙・並べ替え・HTML生成・closedirの成功後に200応答を生成する。

途中状態は下記のClient所有のAutoindexTaskへまとめる。各段階の1周あたりの処理量、RouterとClientの受け渡しAPI、Client状態、完了/失敗結果、期限、キャンセルとCGI子専用解放は次に具体化する。ソートだけを一括実行して分割完了とすることは避け、ソートの比較と移動の処理量も設計対象とする。HTML本文全体を完成後に送る方針に沿って、保持メモリとserializeの負荷も検証する。

大量の項目を持つディレクトリの一覧生成と小さいGET/CGIを並行させ、各段階でEventLoopへ戻ること、期限確認と他接続が進むこと、失敗時のディレクトリ/保持データの回収を受入条件とする。単発のopendir/readdir/closedirによる待ち時間も実環境で測定する。

<a id="autoindex-task-ownership"></a>

#### AutoindexTaskの所有と担当(2026-11-19合意)

Clientは専用クラスAutoindexTaskを値メンバーとして所有する。Clientと一緒に生成・破棄し、タスクのためのnew/deleteを追加する必要はない。EventLoopは既存のClientへのポインター管理を使う。

AutoindexTaskが開いているディレクトリ、取得済み項目名、ソート途中状態、HTML生成位置、生成中のHTML本文と段階を保持する。列挙・並べ替え・HTML生成の進行と、ディレクトリ/保持データの後始末をタスクへまとめる。ソート方式は下記のボトムアップのマージソートを採用する。具体的なコンテナ、各周の処理量、資源メンバー名は次に定める。

Routerは既存のlocation/root/alias/index/autoindex設定に従う対象の解決と、一覧生成へ進む判断を担当する。共有型とRouteResultは下記のAutoindexRequest/AUTOINDEX_REQUIREDを採用する。開始APIは下記のstartを採用する。進行APIは下記のstepを採用する。結果取得は下記のtakeBodyを採用する。回収は下記のcancelを採用する。Client状態は下記のGENERATING_AUTOINDEXを採用する。Client.hppには値メンバーの完全な型定義としてAutoindexTask.hppをincludeする。

コピー禁止は下記の2026-11-20合意に従う。初期状態とcancel契約も合意済み。具体的な初期資源値、CGI子専用解放、期限と異常経路への組み込みは後続の設計で具体化する。表示/並び順/上限/失敗分類はHTTP-27〜32に従う。

<a id="autoindex-route-api"></a>

#### Routerの一覧生成待ちの受け渡し(2026-11-19合意)

RouteResultにAUTOINDEX_REQUIREDを追加し、対応するAutoindexRequestだけを有効な出力とする。指示は対象ディレクトリの絶対パスpathと、リンク生成用の正規化済みURLパスurlPathを値で持つ。公開宣言の正本は[03の共有API](reports/03_config_cgi_upload.md#cgi-api)とする。

Router::routeとprivateなhandleGetの末尾にAutoindexRequest& autoindexを追加する。GETのディレクトリ処理で、既存の末尾スラッシュ補完・index優先・autoindex設定に従って一覧生成を選んだ場合に返す。Clientは所有するAutoindexTaskを開始し、タスクがディレクトリのopen/列挙/ソート/HTML生成/closeを担当する。指示自体はfdやDIR*を持たず、タスクは必要情報を自分の値として保持する。

指示の一時変数を破棄した後も処理を続けられるよう、タスクに指示への参照・ポインターを残さない。完了時にRouterを再実行せず、生成結果から応答を作る。開始APIと開始失敗コードは下記で合意済み。進行APIも下記で合意済み。結果取得は下記のtakeBodyを採用する。回収は下記のcancelを採用する。Client状態と基本遷移は下記で合意済み。本文は下記のsetBodyBySwapで応答へ移す。

<a id="autoindex-start-api"></a>

#### AutoindexTaskの開始API(2026-11-19合意)

```cpp
int start(const AutoindexRequest& request);
```

対象ディレクトリの絶対パスpathとリンク生成用の正規化済みURLパスurlPathをタスク自身の値として保持し、opendirでディレクトリを開く。成功時は0を返す。開いたDIR*をタスクが所有し、呼び出し元の一時的な指示を破棄した後も処理を継続できるようにする。

opendir失敗はHTTP-31に従い、ENOENT/ENOTDIRは404、EACCES/EPERMは403、ENAMETOOLONGは414、その他は500を返す。タスクはHttpResponseを生成せず、呼び出し側が返却コードでmakeErrorを生成し、共通送信準備へ渡す。std::bad_allocは通常の開始失敗コードへ変換せず、既存の接続終了方針に従う。

0は一覧生成を開始する準備ができたことを表す。readdirによる列挙・ソート・HTML生成は後続のstepで進め、開始時には完成した一覧や200応答を返さない。空ディレクトリの完了も後続の進行処理で確認する。startの状態別契約は下記に従う。通常の開始失敗時の保持情報の解放は下記で合意済み。期限とCGI子専用解放への組み込みは次に定める。

<a id="autoindex-step-api"></a>

#### AutoindexTaskの進行API(2026-11-19合意)

```cpp
// AutoindexTaskのpublicな列挙型と操作
enum StepResult { IN_PROGRESS, COMPLETED, FAILED };
StepResult step();
```

Clientは共通のstepを呼び、AutoindexTaskが内部で列挙・ソート・HTML生成の現在段階を管理する。各段階で一定量を進めたらIN_PROGRESSを返し、EventLoopの次の周へ処理を継続する。列挙は最大32回のreaddirという採用案に従う。段階切り替えは下記の契約を使い、ソートは合計256操作、HTML生成は追加最大8 KiBの採用案に従う。

全列挙・HTTP-28によるソート・HTML生成・closedirのすべてに成功したら、下記の完了時解放を済ませてDONEへ進み、COMPLETEDを返す。空ディレクトリも列挙終端後にclosedir成功を確認し、正常な空一覧のHTMLを生成して完了する。完了後のHTML本文は下記のtakeBodyで受け取り、Client側で200/text/htmlの応答を組み立てる。

列挙途中の失敗・closedir失敗・完成HTML8 MiB超過など、HTTP-31/32で500とする通常の生成失敗はFAILEDを返し、呼び出し側が500応答を共通送信準備へ渡す。部分一覧を成功応答として使用しない。std::bad_allocはFAILEDへ変換せず、既存の接続終了方針に従う。

内部状態は下記の7種類のPhaseを採用する。startの状態別契約も下記で合意済み。stepの状態別契約も下記で合意済み。通常の生成失敗時の回収順序は下記で合意済み。期限とEventLoopへの組み込みは下記の2026-11-20合意に従う。

<a id="autoindex-take-body"></a>

#### AutoindexTaskの完成本文取得API(2026-11-19合意)

```cpp
bool takeBody(std::string& body);
```

全列挙・ソート・HTML生成・closedirに成功し、完成本文が未取得であれば、タスクが所有するHTML本文を出力引数へswapで渡してtrueを返す。取得は完了後の初回だけとし、未開始・開始/生成失敗・生成途中・取得済みでは出力引数を維持してfalseを返す。

成功時は出力引数が完成本文を所有する。タスクへの参照・ポインターを出力へ残さず、タスクの破棄後も本文を利用できる。出力引数の以前の内容がswapでタスク側へ移った場合、その不要な旧内容を解放する。空ディレクトリでも、生成した正常な一覧HTMLを成功結果として渡す。

Client側は取得した本文から200応答を組み立て、Content-Typeをtext/htmlとしてprepareResponseへ渡す。最終本文のバイト数からContent-Lengthを生成する既存の応答規則を適用する。AutoindexTaskは完成HTMLの生成と受け渡しを担当し、HttpResponseの組み立てはClient側で行う。cancel後は未開始としてtakeBodyが出力不変でfalseを返す。Client状態と基本遷移は下記で合意済み。本文の格納は下記のswapによる引き渡しに従う。

<a id="autoindex-directory-close"></a>

#### AutoindexTaskのディレクトリ解放時期(2026-11-19合意)

readdirの正常終端を確認した直後、ソートへ進む前にclosedirを行う。HTTP-31に従ってreaddir直前にerrnoを0にし、NULL時に正常終端と列挙失敗を区別する。正常終端後にclosedirが成功したら、タスクが保持する項目名を使ってソート・HTML生成へ進む。

closedirを呼ぶ前にDIR*をタスクの所有状態から外してNULLへ戻し、呼び出しは一度だけとする。ソート・HTML生成中は開いたディレクトリを保持しない。空ディレクトリも同じcloseの順序を使う。

closedir失敗はFAILEDとし、取得済み項目一覧など不要な保持データを破棄して、呼び出し側が500応答を共通送信準備へ渡す。ソート・HTML生成へ進まず、同じDIR*へのclosedirを再試行しない。後続の回収・キャンセル・破棄でも二重closedirを防ぐ。生成途中の一覧を成功応答へ渡さない。

キャンセル時の回収は下記のcancelを採用する。列挙途中の失敗/上限超過時の回収順序とCGI子専用解放の詳細は後続設計で具体化する。完了条件は全列挙・closedir・ソート・HTML生成の成功とする。

<a id="autoindex-enumeration-budget"></a>

#### AutoindexTaskの列挙量(2026-11-19採用案)

列挙段階では、1回のstepにつきreaddirの呼び出しを最大32回とする。採用した回数は並行負荷試験で検証し、各周の処理時間と一覧生成の所要時間を基に見直し得る。

.と..の取得、NULLによる正常終端の確認、列挙失敗となった呼び出しも回数へ含める。表示対象の取得件数で数え直さず、除外項目を読み飛ばす場合も同じ上限に従う。正常終端・失敗・完成HTML上限超過を確認した時点では、残り回数を使い切る前に列挙を止める。

32回の呼び出しで正常終端を確認できなければ、取得済み項目と開いているディレクトリを保持してIN_PROGRESSを返す。次のstepで列挙を継続する。正常終端を確認した場合は、既存のclosedir成功後にソートへ進む契約に従う。closedir成功後はソート開始状態としてIN_PROGRESSを返し、ソートの実処理は次の周から行う。

列挙中のHTML完成長の検査と、HTTP-27〜32の対象・失敗分類を適用する。大量項目、除外項目、32回目付近の終端/失敗/上限超過を試験し、小さいGET/CGIと期限確認が進むこと、CPU使用率とメモリ量を測定する。個々のreaddirやclosedirによる待ち時間も検証対象とする。

<a id="autoindex-stage-transition"></a>

#### AutoindexTaskの段階切り替え(2026-11-19合意)

1回のstepで列挙・ソート・HTML生成のいずれか一つの段階を進める。段階の完了を確認したら、次の段階の開始状態へ切り替えてIN_PROGRESSを返し、EventLoopへ戻る。次の段階の実処理は次の周のstepから行う。

列挙の正常終端を確認したstepではclosedirまで行い、成功したらソート開始状態にして戻る。ソートを完了したstepではHTML生成開始状態にして戻る。空一覧や1項目の一覧も、この段階切り替えの契約に従う。ソート処理を必要としない件数の場合も、ソート段階のstepで完了を判定してからHTML生成を次の周へ回す。

HTML生成を完了したstepは、先行段階とclosedirの成功を確認したうえでCOMPLETEDを返す。通常の失敗を検出した場合は、その段階で既存の失敗/回収経路へ進みFAILEDを返す。次の段階へ処理を連鎖させず、各段階の処理量を個別に管理する。

段階切り替えは同一タスクの継続として扱う。各段階に移るたびにRouterの再実行やタスクのstartを行わず、タスク全体の開始と段階ごとの進行を区別する。ソート/HTML生成は下記の処理量の採用案に従う。内部状態は下記のPhaseを採用する。期限への組み込みは後続の設計で具体化する。

<a id="autoindex-merge-sort"></a>

#### AutoindexTaskの分割ソート方式(2026-11-19合意)

ソート段階はボトムアップのマージソートを採用する。短い整列済み区間から隣接区間をマージし、区間幅を段階的に広げて全体を整列する。再帰を使わず、区間幅・マージ対象範囲・左右の読み取り位置・出力位置・作業領域など、継続に必要な途中状態をAutoindexTaskが保持する。

比較はHTTP-28の元の名前の符号なしバイト列による昇順とし、同じ接頭辞なら短い名前を先にする。比較と要素移動を一定量進めたらIN_PROGRESSを返し、次の周のstepで途中位置から継続する。一方の区間を読み切った後の残り要素の移動も処理量の対象に含める。

作業領域の準備と初期化も分割設計の対象とする。ソート段階へ入った際に項目数に応じた全要素のコピー/初期化をまとめて実行してから分割を始める構成は避け、準備の途中状態と1回のstepの処理量を具体化する。ソート対象は下記の名前を参照する番号とする。比較/移動の処理量は下記の合計256操作を採用案とする。コンテナと作業領域の具体形は次に決める。

全区間の整列後は既存の段階切り替え契約に従い、HTML生成開始状態にしてIN_PROGRESSを返す。名前の変換は表示/リンク生成の段階で行い、ソートは元のバイト列を使う。空/1項目も既存の段階切り替えを適用する。大量項目・逆順・同じ接頭辞・非ASCIIを含む名前で順序と継続位置を検証し、小さいGET/CGIとの並行動作とピークメモリを測定する。

<a id="autoindex-sort-indices"></a>

#### AutoindexTaskのソート対象と名前の保持(2026-11-19合意)

項目名は列挙順の一覧として保持し、ソートでは各名前を参照する番号を並べ替える。名前一覧の位置に対応する0, 1, 2, ...の番号列を準備し、HTTP-28に従って対応する元の名前を比較して、番号をボトムアップのマージソートで整列する。

ソート中は名前一覧の並びと内容を維持し、マージ元と作業領域の間で番号を移動する。名前の文字列をソート用に複製しない。左右の番号から名前を参照して比較し、片側区間の残りの移動も番号の移動として処理量に含める。

HTML生成は整列済み番号列の先頭から順に名前一覧を参照し、表示名のHTMLエスケープとリンクのURLエンコードを行う。名前一覧と番号列はタスクが所有し、HTML生成で必要な期間を通して番号から名前へ到達できる状態を保つ。

番号列の準備と作業領域の初期化も一定量ずつ進める。比較と番号移動は下記の合計256操作の採用案に従う。名前一覧は下記のstd::deque<std::string>を採用する。番号列とマージ用作業領域は下記のstd::vector<std::size_t>を使う。領域の準備は下記のreserveと段階的なpush_backの契約に従う。追加の番号列と作業領域を含むピークメモリを受入試験で測定する。

<a id="autoindex-sort-budget"></a>

#### AutoindexTaskのソート処理量(2026-11-19採用案)

ソート段階の1回のstepは、名前の比較と番号の書き込みを合計256操作以内で進める。名前の比較1回を1操作、番号の書き込み1回を1操作として数える。比較後に番号を書き込む場合は合計2操作とする。

番号列の準備と作業領域の初期化も、番号を1個書き込むごとに1操作として同じ上限へ含める。一方の区間を読み切った後の残りの番号の移動も、各書き込みを数える。1回のstep内で準備からマージへ進む場合も、その呼び出しの合計操作数を引き継ぎ、途中で回数をリセットしない。

上限に達してソートが未完了なら、準備位置・マージ位置・必要な比較結果などを保持してIN_PROGRESSを返す。比較と書き込みの間で中断する場合も、次のstepから整列を正しく継続できるようにする。ソート完了時は既存の段階切り替えに従ってHTML生成を次の周へ回す。

256は並行負荷試験で検証する採用値とする。名前の比較1回に含まれるバイト比較数は名前と共通接頭辞の長さに依存するため、長い共通接頭辞・非ASCII・逆順・大量項目でstepの処理時間と他接続の応答時間を測定する。作業領域の確保・コンテナの拡張に伴う処理も、具体的な領域設計と負荷測定に含める。

<a id="autoindex-html-budget"></a>

#### AutoindexTaskのHTML生成量(2026-11-19採用案)

HTML生成段階では、1回のstepで完成本文へ追加する量を最大8 KiB(8,192バイト)とする。固定HTML・表示名・hrefを含むすべての追加バイトを数える。HTMLエスケープとURLエンコードは、変換後の出力バイト数で上限を判定する。

上限に達して本文が未完成なら、現在の項目番号・生成部分・入力位置と必要な変換途中状態を保持してIN_PROGRESSを返す。次の周のstepでその位置から継続する。固定HTMLや1項目の途中でも戻れ、文字列全体を先に変換してから8 KiBずつ本文へ移す構成による一括生成を避ける。変換結果の一部を次のstepへ持ち越す場合は、その出力位置も保持して重複/欠落を防ぐ。

空一覧の固定HTMLも同じ上限へ含める。8 KiB目の追加でHTML全体が完成した場合は、既存の先行段階とclosedirの成功を確認してCOMPLETEDを返す。未完成ならIN_PROGRESSとする。完成HTML全体の8 MiB上限はHTTP-32に従って列挙中から検査し、本文生成時も整合を確認する。

8 KiBは並行負荷試験で検証する採用値とする。長い名前/URL、HTML特殊文字、percent-encoding対象のバイト、固定HTMLと項目の境界、8 KiB前後、空一覧と大量項目を対象に、完成HTMLの正しさと並行GET/CGIの応答を確認する。本文領域の拡張・保持メモリと1回のstepの処理時間も測定する。

<a id="autoindex-cancel-api"></a>

#### AutoindexTaskのキャンセルと破棄(2026-11-19合意)

```cpp
void cancel();
```

保持中のDIR*があれば所有状態から外してNULLにし、closedirを一度だけ呼ぶ。closedirが失敗しても同じDIR*を再試行せず、ソート/HTML生成前にすでに閉じたディレクトリを重ねて閉じない。

名前一覧・番号列・マージ用作業領域・生成途中/完成後のHTML本文・保持パスとURL・継続用の位置情報を解放または初期化し、未開始状態へ戻す。どの段階や終端状態からも繰り返し呼べる。cancel後のtakeBodyは出力引数を維持してfalseを返し、すでに取得済みで呼び出し側が所有する本文は維持する。

cancelは資源回収を担当し、HTTP応答を生成しない。デストラクターも同じ回収処理を使い、例外を外へ出さない。cancelの繰り返しと続く破棄で二重closedirを防ぐ。ディレクトリ内のファイルの削除や変更は行わない。

内部状態は下記のPhaseを採用する。startの状態別契約は下記で合意済み。stepの状態別契約も下記で合意済み。通常の生成失敗は下記の回収後にFAILEDを返す。closedir失敗時のログ方針、例外と期限/切断/CGI子専用経路への具体的な組み込みは後続の設計で定める。

<a id="autoindex-phase"></a>

#### AutoindexTaskの内部状態(2026-11-19合意)

```cpp
private:
    enum Phase {
        IDLE, ENUMERATING, SORTING, BUILDING,
        DONE, ERROR, TAKEN
    };
    Phase phase_;
```

| Phase | 意味と遷移 |
|---|---|
| IDLE | 初期状態とcancel後の未開始状態 |
| ENUMERATING | start成功後。開いているディレクトリから項目を列挙する |
| SORTING | 列挙終端とclosedir成功後。番号列の準備と分割マージソートを進める |
| BUILDING | ソート完了後。整列済み番号から名前を参照してHTMLを生成する |
| DONE | 全工程が成功し、完成HTML本文を未取得 |
| ERROR | 通常の開始失敗または生成失敗後 |
| TAKEN | takeBodyが初回成功し、完成本文を渡した後 |

生成段階と終端状態を一つのPhaseで管理し、別のタスク状態の列挙型や完了/失敗/取得済みのboolを重ねて持たない。ソートとHTML生成それぞれの継続位置は、該当段階で必要な処理データとして保持する。

ENUMERATINGからSORTING、SORTINGからBUILDINGへ切り替えたstepは既存契約に従ってIN_PROGRESSを返す。BUILDINGの完了でDONEへ進みCOMPLETEDを返す。takeBody成功でTAKEN、cancelはどの状態からもIDLEへ戻す。通常の開始/生成失敗はERRORへ進め、例外時は既存の接続終了方針に従う。

公開StepResultとprivateなPhaseは役割を分け、startの状態別契約は下記で合意済み。stepの状態別契約も下記で合意済み。通常の生成失敗時の回収は下記の契約に従う。DIR*を含む資源は実際の所有状態で管理し、終端後の二重closedirを防ぐ。

<a id="autoindex-start-state"></a>

#### AutoindexTask::startの状態別契約(2026-11-19合意)

startはIDLEで一覧生成の開始処理へ進む。ENUMERATING/SORTING/BUILDING/DONE/ERROR/TAKENで呼ばれた場合は500を返し、現在のPhase・DIR*・名前一覧・番号列・作業領域・HTML本文・保持情報と、入力のAutoindexRequestをすべて維持する。opendirや現在のディレクトリのclose、本文の破棄を行わない。

再呼び出し拒否の500によって、内部PhaseをERRORへ変更しない。IDLEからの通常の開始失敗は、既存の分類のコードを返してERRORへ進む契約を使う。タスクの再利用はcancelで回収してIDLEへ戻してからstartを呼ぶ。開始失敗後や生成失敗後も同じ再利用手順とする。

<a id="autoindex-step-state"></a>

#### AutoindexTask::stepの状態別契約(2026-11-19合意)

| 呼び出し時のPhase | 動作と返却値 |
|---|---|
| ENUMERATING / SORTING / BUILDING | 該当段階を既存の処理量・段階切り替え契約に従って進め、IN_PROGRESS/COMPLETED/FAILEDを返す |
| DONE | 状態と完成本文を保持してCOMPLETEDを返す |
| ERROR | 状態と保持データを維持してFAILEDを返す |
| IDLE / TAKEN | 状態と保持データを維持してFAILEDを返す |

生成中以外での呼び出しはreaddir・ソート・HTML生成・closedirを行わず、完成本文やその他の保持データを変更しない。IDLE/TAKENでFAILEDを返しても内部PhaseをERRORへ変更しない。

DONEでCOMPLETEDを繰り返し返しても、完成本文はtakeBodyの初回成功で一度だけ渡す。cancel後はIDLEの契約で扱う。失敗時の回収を終えた後のstepで、同じDIR*やデータを重ねて回収しない。

<a id="autoindex-step-failure"></a>

#### AutoindexTaskの通常の生成失敗時の回収(2026-11-19合意)

ENUMERATING/SORTING/BUILDINGで通常の生成失敗を検出した場合、stepはタスク内で次の回収を済ませてERRORへ進み、FAILEDを返す。列挙失敗・closedir失敗・完成HTML上限超過など、既存のHTTP-31/32に従う失敗を対象とする。

1. 保持DIR*があれば所有状態から外してNULLにし、closedirを一度だけ行う。すでにcloseを試みたDIR*は、close失敗後も再試行しない。
2. 名前一覧・番号列・マージ用作業領域・生成中HTML・保持パスとURLを解放し、各段階の継続位置などを初期化する。
3. PhaseをERRORにしてFAILEDを返す。

Clientはこの結果を受けてmakeError(500)を共通送信準備へ渡す。回収中のclosedir失敗でも元の生成失敗に対応する500を維持する。後続のstepはERRORのままFAILEDを返し、cancel/破棄でも回収済み資源を重ねて処理しない。

cancelはIDLEへ戻す契約であり、生成失敗の結果を返す際はERRORとする。共通化する内部回収処理の具体形、closedir失敗時のログ方針と例外時の回収は後続の設計で定める。std::bad_allocは既存の接続終了方針に従い、通常のFAILEDへ変換しない。

<a id="autoindex-start-failure"></a>

#### AutoindexTaskの通常の開始失敗時の回収(2026-11-20合意)

IDLEからのstartが通常の開始失敗となった場合、取得済みDIR*があれば所有状態から外してNULLにし、closedirを一度だけ行う。close失敗時も再試行せず、後続のcancel/破棄による二重closedirを防ぐ。

タスクが保持したパス・URL・名前一覧・番号列・作業領域・HTML本文を解放し、各段階の継続位置などを初期化する。回収後はPhaseをERRORとし、HTTP-31に従う元の開始失敗コードを返す。回収中のclosedir失敗で開始失敗コードを変更しない。

入力のAutoindexRequestは全フィールドを維持する。ERRORとなったタスクはcancelでIDLEへ戻してから再利用する。IDLE以外でstartが500を返す場合は、既存の再呼び出し拒否の契約で状態と資源を維持する。

std::bad_allocは通常の開始失敗コードへ変換せず、既存の接続終了方針に従う。回収処理の具体的な共通化、closedir失敗時のログ方針と例外時の解放手順は後続の設計で具体化する。

<a id="autoindex-noncopyable"></a>

#### AutoindexTaskのコピー禁止(2026-11-20合意)

```cpp
private:
    AutoindexTask(const AutoindexTask&);
    AutoindexTask& operator=(const AutoindexTask&);
```

C++98のprivateなコピーコンストラクターとコピー代入演算子を宣言し、定義しない。DIR*と各段階の保持データを所有するタスクをコピーで増やさず、二重closedirを防ぐ。

ClientがAutoindexTaskを値メンバーとして所有し、Client自身のコピー禁止とEventLoopのClientポインター管理に従う。タスクはClientと一緒に生成・破棄する。指示用のAutoindexRequestはDIR*を持たず、path/urlPathを値で受け渡す既存の契約に従う。

<a id="client-autoindex-state"></a>

#### Clientの一覧生成中の状態と基本遷移(2026-11-20合意)

ClientのStateにGENERATING_AUTOINDEXを追加する。

| 契機 | 処理と遷移 |
|---|---|
| PROCESSINGでAUTOINDEX_REQUIRED | Client所有のAutoindexTaskを開始し、startが0ならGENERATING_AUTOINDEXへ進む |
| startの通常の開始失敗 | 返却されたHTTPコードでmakeErrorを生成し、prepareResponseへ渡す |
| GENERATING_AUTOINDEXでIN_PROGRESS | 同状態を維持し、タスク内の現在段階から次のstepを継続する |
| GENERATING_AUTOINDEXでCOMPLETED | takeBodyで完成HTMLを取得し、setBodyBySwapで200/text/html応答へ移してprepareResponseBySwapへ渡す |
| GENERATING_AUTOINDEXでFAILED | タスク内の回収済み結果を受け、makeError(500)をprepareResponseへ渡す |

成功応答は既存の共通送信準備を経てWRITING_RESPONSEへ進む。エラー応答も共通経路で扱い、設定されたerror_pageを読む場合は既存のREADING_FILEへ進む。Routerの再実行や一覧生成の再開始を行わない。

ClientのGENERATING_AUTOINDEXは一覧生成タスクの待機を表し、列挙/ソート/HTML生成の段階はAutoindexTaskのPhaseで管理する。期限とEventLoopの進行順序は下記の共通規則に従う。切断/SIGINT/例外時の具体的な回収経路、COMPLETED後のtakeBodyが契約に反してfalseだった場合の扱いは次に具体化する。

<a id="autoindex-loop-timeout"></a>

#### AutoindexTaskの期限とEventLoopへの組み込み(2026-11-20合意)

AutoindexTaskにもファイル読み取り/保存の期限・進行規則を適用する。Clientはstart直前に開始時刻を記録し、列挙・ソート・HTML生成の全段階を通して60秒の固定期限を採用案とする。opendirと開始準備を含め、処理の進行や段階切り替えで期限を延長しない。

start成功直後に期限を再確認し、期限内ならGENERATING_AUTOINDEXへ進む。開始した周はfileTaskStartedThisRound_をtrueとし、最初のstepを次の周へ回す。以後は各周最大1回のstepを、ソケット/CGIイベント処理後に実行する。進行可能なautoindexタスクがあればpoll待ち時間を0とし、返却イベント0件でも期限確認と対象タスクの進行を行う。

step前に期限確認し、超過ならstepを呼ばずcancelして500をprepareResponseへ渡す。step復帰後はCOMPLETEDを優先して成功経路へ進み、FAILEDは回収済みの失敗結果として500へ進む。IN_PROGRESSは復帰後の時刻で再確認し、超過ならcancel後500、期限内なら次周へ戻す。start失敗は既存の返却コードを使う。

一覧生成の500でerror_pageを読む場合は、error_pageのstart直前から新しい60秒を計る。error_pageの期限超過は元の500を保持した内蔵HTMLへ進む。送信バッファ準備後の送信期限も既存の60秒を適用する。

autoindexの計時に必要な時刻取得/解析の失敗・逆行を検出したら、cancelで資源を回収してCLOSINGへ進み、既存のfd回収/関連付け解除と周末削除を使う。開始時刻取得に失敗した場合はstartを呼ばない。追加のHTTP応答やerror_page生成を開始しない。

EventLoop全体は周頭フラグリセット、poll、全体期限確認、ソケット/CGIイベント、対象タスクの進行、周末削除の既存順序を使い、GENERATING_AUTOINDEXも対象へ含める。段階ごとの処理量は列挙32回/ソート256操作/HTML追加8 KiBの採用案に従う。並行負荷試験で処理時間・期限検出・CPUとメモリ・資源回収を検証する。

<a id="autoindex-success-release"></a>

#### AutoindexTaskの完成時の保持データ解放(2026-11-20合意)

HTML全体が完成し、先行段階とclosedirの成功を確認した時点で、stepは完成HTML本文を保持したまま、名前一覧・番号列・マージ用作業領域・保存していたディレクトリパスとURL・不要な継続情報を解放または初期化する。この整理を済ませてDONEへ進み、COMPLETEDを返す。

DONEでは完成HTML本文と本文未取得を表す状態を保持し、takeBodyの初回成功で本文を渡してTAKENへ進む。DONEでstepを再度呼んでも、解放済みデータを使ってソート/HTML生成をやり直さず、既存契約に従ってCOMPLETEDを返す。後続のcancel/破棄は残る本文を回収し、取得済み本文は呼び出し側が所有する。

空/1項目/大量項目でも同じ完了処理を適用する。完成本文の8 MiB上限、表示/リンク/並び順を維持し、応答生成/送信中に不要な名前一覧やソート領域を残さない。大量項目の保持データ解放にかかる時間も含めて、最後のstepの処理時間と並行GET/CGIの応答、ピークメモリと完了後の回収量を測定する。

解放経路でのstd::bad_allocは既存の接続終了方針に従う。具体的なコンテナと解放処理、例外捕捉/回収への組み込みは後続の内部設計で具体化する。

<a id="autoindex-name-container"></a>

#### AutoindexTaskの名前一覧コンテナ(2026-11-20合意)

名前一覧はstd::deque<std::string>で保持する。列挙で採用した名前を文字列として所有し、取得順に末尾へ追加する。ソート中も名前一覧の並びと内容を維持する。

番号列の各番号からdequeの対応位置の名前を参照して比較し、HTML生成も整列済み番号を通して同じ名前を参照する。名前を別のソート用文字列一覧へ複製しない。名前一覧はHTML生成で必要な期間を通して保持し、生成成功/失敗/キャンセル時は既存の保持データ解放契約に従う。

末尾追加で既存の名前全体をコピーする処理を抑える目的で採用する。dequeの領域確保・管理領域の拡張・文字列コピーと解放にかかる時間は並行負荷試験で測定する。AutoindexTask.hppでは<string>と<deque>をincludeする。

<a id="autoindex-index-containers"></a>

#### AutoindexTaskの番号列とマージ用作業領域(2026-11-21合意)

マージ元と出力先に、それぞれstd::vector<std::size_t>を使う。各番号は列挙順で保持する名前一覧の位置を表す。必要な領域は列挙を終えて項目数が確定した後に準備する。

番号列の初期化と作業領域への番号の書き込みは、いずれも1個につき1操作として、名前の比較と合わせて1回のstepにつき最大256操作の採用案に従う。準備位置を保持し、複数のstepに分けて初期化する。

列挙とclosedirの成功後、ソート段階の準備で項目数Nを使って空の2本のvectorにそれぞれreserve(N)する。reserve後も両方のsizeは0で、要素の追加は後続の処理で行う。マージ元には0からN-1までの番号をpush_backで順に追加し、1回のstepの操作数に応じて中断・再開する。最初のマージでは出力先に結果の番号をpush_backし、番号を1個追加するごとに1操作と数える。未追加の位置には添字でアクセスしない。

<a id="autoindex-merge-buffer-reuse"></a>

#### マージのパス間での作業領域の再利用

各パスで全N個の番号を書き終えたら、2本のvectorをswapして出力結果を次のパスのマージ元にする。最初のパス完了時点で両方のsizeがNに達する。2パス目以降はsizeをNに保ち、出力位置を0から進めながら既存要素に添字で上書きする。中断時は出力位置を保持して次のstepで再開する。上書き1件を1操作として、比較を含む1 stepあたり256操作の上限に含める。

最後のパスでもswapを行い、整列済みの番号はマージ元からHTML生成に渡す。空または1項目の場合はマージのパスを実行せず、既存の段階切り替えに従う。

両方のvectorの領域確保にかかる時間とピークメモリを、並行負荷試験で測定する。

<a id="autoindex-response-body-swap"></a>

#### 完成HTMLからHttpResponseへの本文引き渡し(2026-11-21合意)

```cpp
// HttpResponseのpublic関数
void setBodyBySwap(std::string& body, const std::string& contentType);
```

ClientはtakeBodyで取得した完成HTMLをsetBodyBySwapへ渡し、swapで応答に本文の所有権を移す。成功後は引数bodyを空にし、応答が以前に保持していた本文の領域を解放する。新しい本文の実バイト数と指定された種類を、既存のsetBodyと同じ規則でContent-LengthとContent-Typeに反映する。

autoindex成功時は200とtext/htmlの応答をprepareResponseBySwapへ渡す。本文を応答へ格納するときと、応答をpendingResponse_へ渡すときの本文コピーを省く。送信時も[ヘッダーと本文の分離送信](#split-response-send)で本文の連結コピーを省く。メモリ確保失敗は既存の接続終了方針に従う。

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
| HTTP-41 | 保存の完了と失敗 | 短い正のwriteでは書き込み位置を進め、残りは次のEventLoop周で書く。各周最大1回・最大8 KiBとし、0以下なら500。write後のerrno分岐をせず、close失敗も500。open時の同名対象は409、権限不足/読み取り専用は403、保存名によるパス長制限超過は400、保存先消失/その他のopen失敗は500。保存とcloseが成功するまで成功応答にしない |
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
| HTTP-53 | 405のAllow(2026-11-16更新) | locationのメソッド禁止による405は、そのlocationの許可メソッドをGET, POST, DELETEの固定順でカンマ＋SP区切りに列挙する。CGI有効/POST許可のlocationで実在する対象または選択indexがCGI非対象の場合も405とし、この対象で利用できるメソッドだけを列挙する。GET許可ありならAllow: GET、POSTだけ許可なら空のAllowを必ず出力する。後者の405へ処理不能なPOSTを載せず、禁止GET/DELETEを追加しない。CGI由来Allowの転送規則は維持する |
| HTTP-54 | Webservのリダイレクト | ConfigのreturnとディレクトリGETの末尾スラッシュ補完による301/302は、既決定のLocation、空本文、Content-Length: 0で応答する。移動先を説明する追加HTMLは生成しない。CGIのLocation・本文の規則は変更しない |
| HTTP-55 | Server | Webserv自身ではServerヘッダーを生成しない。RFCで任意のためBare minimumとして省く。CGI由来ヘッダーの既存の転送規則は維持する |
| HTTP-56 | Date(暫定案) | 使用可能な暦日時の取得手段が確定するまで、Webserv自身ではDateを生成しない。固定日時・ファイル更新日時で代用しない。HTTP/1.0でも推奨、RFC 9110では時計を持つサーバーの2xx/3xx/4xxに必須であり、関数制限だけを理由に免除されたとは説明しない。課題向けの暫定的な制限として記録し、使用可能な取得手段が確認できたら見直す。CGI由来ヘッダーの転送規則は維持する |
| HTTP-57 | エラー判定の順序 | 全コードの一律優先順位表を作らず、要求行・ヘッダー・本文の検証で拒否が確定したら通常処理へ進めない。正常な要求はlocation選択→メソッド許可→Configのreturn→ファイル/アップロード/CGI処理。確定したエラーを後始末の失敗で上書きしない。既決定の早期拒否とExpectより先に確定した400/413等の優先を維持する |
| HTTP-58 | error_page適用後の応答 | 400〜599の本文を指定ファイルで差し替え、未指定/読み取り失敗なら内蔵HTMLへ一度だけ戻す。元のステータス、CGIの検証済み説明文、必要なAllow等の保持を維持し、置換後の本文からCLを生成する。200〜399へは適用せず、再ルーティング・CGI実行・再帰的なエラー生成を追加しない(C-20・CGI-130〜133) |

例: DELETE禁止のlocationに正常なDELETE要求が届いた場合は、ファイルの有無を調べる前に405＋Allowを返す。対象がないことを理由に404へ変更しない。エラーページ読み取り失敗でも元の404等を500へ変更しない。本文の長さは文字数ではなくバイト数で数える。

根拠: [課題PDF](webserv.pdf)の正確なHTTPステータスとデフォルトエラーページ、[RFC 1945 §10.4](https://www.rfc-editor.org/rfc/rfc1945.html#section-10.4)・[§10.6](https://www.rfc-editor.org/rfc/rfc1945.html#section-10.6)、[RFC 9110 §6.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-6.6.1)・[§8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6)・[§10.2.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.4)・[§15.5.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.6)。必要なヘッダーに限定する構成・空本文のWebservリダイレクト・エラー処理順はチーム方針。Dateの暫定的省略を含め、RFC全体への完全準拠とは説明しない。実装・受入試験は未完了。

<a id="http-api"></a>

### HTTP APIとデータの保持の合意事項(HTTP 9/10・2026-10-07)

本節がHTTPクラス間の責務・API・保持方針の正本。具体的なHttpRequest/HttpResponseの宣言は本書の主要インターフェースを参照し、共有宣言は03を正本とし、2026-11-15のroute拡張を反映する。CGIプロセス側のシグネチャは維持する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-59 | 責務 | Clientがソケットの受信/送信、HttpRequestがバイト列の解析と完成/エラー判定、Routerが完成した要求とConfigによる処理選択、HttpResponseがステータス/ヘッダー/本文と送信用バイト列の生成を担当する。HttpRequestはrecv/sendを呼ばない |
| HTTP-60 | HttpRequestの生成と状態 | explicit HttpRequest(std::size_t maxBodySize)でConfigの本文上限を生成時に渡し、途中では変更しない。外部へ公開するStateはINCOMPLETE/COMPLETE/ERRORの3つ。要求行/本文/chunkedの内部解析段階は非公開。appendData(const char*, std::size_t)のたびClientが状態を確認し、途中なら受信継続、完成ならRouterを1回だけ呼び、ERRORならエラー応答へ進む |
| HTTP-61 | エラーコードと終端状態 | getErrorCode()はINCOMPLETE/COMPLETEなら0、ERRORなら最初に確定したHTTPコード。COMPLETE/ERROR後にappendDataしても結果を変更せず、余剰バイトを次の要求へ使わない。要求の再利用・途中の上限変更を行わない |
| HTTP-62 | 要求の保持とgetter | ClientがHttpRequestを値として保持し、要求全体を別の受信バッファへ重複保存しない。Routerへは完成済みHttpRequestをconst参照で渡す。getPathはquery分離・1回デコード・正規化済みのURLパスで、Routerが再デコードしない。getQueryStringは未デコード、getBodyはchunked復号済み。メソッド/受信版/単一ヘッダーgetterを維持し、getHeadersは重複を含む全行を保持する。参照はHttpRequestの生存中だけ使う |
| HTTP-63 | Routerの寿命と結果 | EventLoop内のRouterを1個使い回し、要求ごとのデータを保持し続けない。2026-11-15の合意でrouteにFILE_READ_REQUIREDとFileReadRequest出力引数を追加し、2026-11-16にFILE_WRITE_REQUIREDとFileWriteRequest出力引数も追加した。RESPONSE_READYはresponse、CGI_REQUIREDはcgi、FILE_READ_REQUIREDはfile、FILE_WRITE_REQUIREDはwrite、AUTOINDEX_REQUIREDはautoindexだけが有効。Routerは配信対象の解決・権限確認まで行い、ファイルのopen/read/closeはClientが所有するFileReadTaskへ任せる。CGI用データのコピー・寿命とプロセス側APIは維持する。共有宣言の正本は03、詳細は本書の「通常ファイル読み取りの設計修正」 |
| HTTP-64 | 応答と送信データの保持 | HttpResponseがヘッダー/本文/ステータスを値で保持する。共通送信準備ではステータス行とヘッダーを送信用に生成し、保持済みの本文を別に送る。ヘッダーを送り切ってから本文へ移り、短いsendでは該当部分の送信位置から続ける。詳細は[分離送信の契約](#split-response-send)を参照する。I/O量と全体メモリはHTTP-65・71・72に従う |

課題/RFCはこのクラス構成やAPIを指定していない。上の責務分担はチーム方針であり、バイト列としての解析は[RFC 9112 §2.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-2.2)、完成/未完了の区別は[§6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3)・[§8](https://www.rfc-editor.org/rfc/rfc9112.html#section-8)を踏まえる。実装・受入試験は未完了。

<a id="getsockname-failure"></a>

### 接続先情報の取得失敗時の契約(2026-11-23採用)

EventLoopはaccept直後、Clientの生成とpollへの登録より前にgetsocknameで接続先のローカルIPv4とポートを取得する。成功したら取得値とacceptで得た接続元IPv4をConnectionInfoに入れてClientを登録する。Routerは有効なConnectionInfoを受け取る。

getsocknameが失敗したら、EventLoopが新規接続のfdを一度だけcloseする。Clientの生成、poll登録、要求解析、HTTP応答の生成は行わず、既存の接続と待ち受けの処理を続ける。この分岐はacceptした接続の情報取得に適用する(C-28・CGI-35・37・93)。

<a id="http-network-boundary"></a>

### ネットワーク境界・計時・通常ファイルI/Oの合意事項(HTTP 10/10・2026-10-07)

課題PDFを再確認して採用する最小構成。計時・切断待ち時間・独自の全体資源上限を追加しない構成は、実装と負荷試験で見直し得る現時点の採用案とする。2026-11-15に通常ファイルの読み取りを各周に分割する方式へ変更し、Routerの結果を3種類へ拡張した。ClientにREADING_FILEを追加し、静的ファイル読み取りの基本遷移も合意した。HttpRequestの公開3状態、ConfigとCGIプロセス側のAPIは維持する。error_page生成APIや読み取り期限など、変更に伴う未決定事項は下記で区別する。

| ID | 項目 | 決定 |
|---|---|---|
| HTTP-65 | 通常ソケットのI/O | 単一pollの準備通知に従い、接続ごと・各周にrecv/sendをそれぞれ最大1回、各最大8 KiBとする。ソケットはノンブロッキング。sendは残量との小さい方を渡し、正の戻り値だけ送信オフセットを進める。recvの負値、送るデータがあるsendの0以下は接続を閉じ、errnoで再試行・待機へ分岐しない。送信失敗後に追加のHTTP応答を作らない。CGIパイプの既決定は維持する |
| HTTP-66 | 受信EOF | 受信中の未完了要求のrecv=0はHTTP-19の破棄/close。正常な要求の完成後、または早期エラー応答へ進んだ後は受信EOFを記録しPOLLIN監視を外すが、生成・送信中の応答を続ける。EOFを根拠に次の要求を作らない。通常ソケットのPOLLHUPだけで未読データを破棄せず、残りをrecvして0を確認する。POLLERR/POLLNVALやI/O失敗はclose。CGI待機中の切断判定・停止/回収はCGI-113等を変更しない |
| HTTP-67 | 応答後の切断(採用案) | 正常応答も413/417等の早期応答も、送信中の追加受信は8 KiBの固定バッファで読み捨て、解析・保存しない。全応答のsend完了後にshutdown(SHUT_WR)し、受信EOF確認済みならclose、未確認なら読み捨てを続けEOF/エラーまたは送信完了から1秒でcloseする。期限は受信で延長せず、shutdown失敗もclose。内部ClientにDRAININGを設け、待機中は送信データがないのでPOLLOUTを外す。1秒は応答到達・ACKを保証する値ではなく、実際の早期拒否/負荷試験で見直す |
| HTTP-68 | 要求受信の期限 | accept成功から60秒以内に要求を完成させる。途中のバイト到着では延長しない。未完了のまま期限になった場合、1バイト以上受信済みなら408の応答経路へ進み、未受信の空接続なら応答なしでcloseする。未完了要求の保存/削除/CGI起動をしない。受信失敗/EOFが先に確定した場合はその切断方針を維持する |
| HTTP-69 | 応答送信の期限 | 最終応答の送信バッファが準備できた時から60秒。部分sendで延長せず、未送信のまま期限になったらcloseし、別のエラー応答を送らない。CGI待機中にこの期限を適用せず、CGIは起動から10秒・延長なし・504を維持する。全送信後はHTTP-67の1秒を使う。これらは処理がEventLoopへ戻ってから検出する期限であり、同期ファイル処理の停止を防ぐ保証ではない |
| HTTP-70 | 共通の経過時間(採用案) | CGI-124〜126のLinux /proc/uptimeの取得方法を通常接続にも使い、各周1回の有効な現在値を共有する。CGI起動直前の追加取得は維持。単一pollの待ち時間は通常接続だけの時も最大100 ms。通常の受信/送信/切断待ち、およびファイルタスクの計時で取得・解析失敗や値の逆行を検出したら、その接続を閉じ、計時なしで無期限に継続しない。CGI起動前/実行中の計時失敗は既存の500と停止/回収規則を維持する。使用可能な時計APIと断定せず、利用可否・精度・追加I/O負荷を実環境で検証する |
| HTTP-71 | 通常ファイル処理(2026-11-16に保存方式も変更) | 静的ファイル/error_pageはClientが所有するFileReadTaskで読み、全量の読み取り・close成功後に応答を生成する。EventLoopの各周で各タスクに最大1回・最大8 KiBのreadを行い、未完了なら次の周へ戻す。readの正値は本文へ追加、0はEOF、負値は失敗。read後にerrnoで分岐せず、通常ファイルをpollへ登録しない。FileReadTaskとerror_pageの合意済みAPI・残件は下記を参照する。2026-11-16の合意でアップロード保存も全検証後にClient所有のFileWriteTaskで分割し、各周に各タスクのwriteを最大1回・最大8 KiB進める。未完了ならEventLoopへ戻す。全量のwriteとclose成功後だけ200を生成する。短い正のwriteは残量へ進め、0以下は失敗とし、write後のerrno分岐は行わない |
| HTTP-72 | メモリと資源(採用案・2026-11-15更新) | 要求/CGI/一覧の既存上限を維持し、静的ファイル専用サイズ上限・独自の接続数上限・全体メモリ上限・件数による503を追加しない。HTTP-62〜64の保持・参照・swapを使い、確定した本文は送信完了まで必要な領域を保持し、その後に回収する。残る要求/送信バッファ等は不要時または接続破棄時に解放する。serialize時は元本文と生成文字列が一時的に共存し、全体メモリの一定上限は保証しない。接続処理でstd::bad_allocが起きたら、その接続の処理を中止し、fd・保持本文・送信バッファを回収してcloseする。500の再生成や追加確保を伴う復旧を行わず、ほかの接続を継続する。関連CGIは既存の停止/回収へ進める。他の例外を一律に同じ扱いへ変更する合意ではない。これだけでOSによる強制終了を防げるとはせず、負荷試験で資源量と回収を確認する |
| HTTP-73 | ログ形式 | Loggerは[LEVEL] messageとし、暦日時を出力しない。ファイル出力なし・レベルによる絞り込みを維持する。/proc/uptimeをDateや暦日時へ流用せず、DateのHTTP-56の暫定案とは区別する |
| HTTP-74 | 受入条件と見直し | 04の通常接続/ファイル/負荷試験は未実施。部分送受信、EOF/RST、早期拒否後の応答受信、期限、計時失敗、ファイル処理中の他接続/CGIの進行、終了後のfd/所有メモリ回収を確認する。評価項目の空ページGETのSiege -bでAvailabilityが99.5%を超え、長時間の継続でメモリが増え続けず、接続がハングしないことを確認する。一括処理でイベントループが滞る、資源量/精度/可用性が基準を満たさない場合は採用案を見直し、影響するAPI/規則/試験を関連資料に揃えて記録する |

根拠: [課題PDF](webserv.pdf) p.8〜9は常にノンブロッキング・単一poll・read/write後のerrnoによる分岐禁止・無期限ハング禁止・負荷耐性を要求し、通常のディスクファイルだけは準備通知を不要としている。[RFC 9112 §9.6](https://www.rfc-editor.org/rfc/rfc9112.html#section-9.6)の段階的な切断を参考にする。[Linux poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)では通常ファイルは常に準備完了となるため、pollへ登録してもディスクI/Oが非同期になるわけではない。経過時間の取得元は[proc_uptime(5)](https://man7.org/linux/man-pages/man5/proc_uptime.5.html)。8 KiB・60秒・1秒・100 ms・クラス構成・独自の全体上限を省く選択は課題/RFCの指定値ではなくチーム方針である。

[42EvalHubの評価項目](https://www.42evalhub.com/common/webserv)(2026-10-07確認)には通常ファイルの同期I/Oがイベントループを停滞させないことと負荷試験が含まれる。ディスクファイルのpoll免除を、イベントループの停止や無制限の資源消費を認める根拠にはしない。現時点で合格・完全ノンブロッキングを実証したとは扱わない。DELETEの削除手段はHTTP-50で確定済み。Date(HTTP-56)は暫定案として残る。

<a id="file-read-design"></a>

### 通常ファイル読み取りの設計修正(2026-11-15合意・実装未着手)

HTTP-63・71とCGI-95の更新内容をまとめる。従来の「Routerの1回の呼び出しでファイルを読み切る」案を、接続ごとの読み取りを各周で進める方式へ変更する。読み取り方式・所有者・Routerの受け渡しと、ClientのREADING_FILEの基本遷移を合意済み。以下の未決定事項を完了扱いにしない。

#### 合意した読み取り方式と所有者

- 静的ファイルは全量を読み、EOFとclose成功を確認してから200の応答を生成する。途中からクライアントへ本文を送る方式は採用しない。失敗時の部分本文を200で送らない既存方針を維持する。
- Clientは専用クラスFileReadTaskを値メンバー`FileReadTask fileTask_;`として所有する。FileReadTaskがファイルfd・読み取り済み本文・読み取り状態を保持し、open/read/closeと後始末を担当する。Clientと一緒に生成・破棄し、タスク自体のnew/deleteは行わない。未開始時はfd=-1、本文は空とし、本文用の大きなバッファを先行確保しない。
- FileReadTaskとClientはコピー禁止とする。C++98のprivateなコピーコンストラクター・コピー代入演算子を宣言し、定義しない。fdの所有者をコピーで増やさず、二重closeを防ぐ。EventLoopはClientへのポインターを管理し、Clientをコンテナへ値としてコピーしない。FileReadRequestはfdを持たない読み取り指示の値型のままとする。
- Routerは配信対象のパス解決・種別/権限確認までを担当する。通常ファイルの読み取りが必要ならFileReadRequestを返す。FileReadRequestは読み取り指示の値であり、fdや途中の本文を持たない。FileReadTaskは必要な情報を自分で保持し、一時的なFileReadRequestへの参照・ポインターを残さない。
- EventLoopが各周で各タスクの読み取りを最大1回・最大8 KiB進める。正のreadは本文へ追加し、未完了なら他の接続・CGIを処理できるようループへ戻る。通常ファイルはpoll対象にせず、ソケット/CGIパイプは既存どおり単一pollの準備通知に従う。
- FileReadTaskの結果は「未完了・完了・失敗」を区別する。操作はstart/step/takeBody/cancelの4つに分け、役割は下記で合意済み。startはbool start(const FileReadRequest& request)、stepはStepResult step()とする。StepResultはIN_PROGRESS/COMPLETED/FAILEDの3種類。takeBodyはbool takeBody(std::string& body)とする。cancelはvoid cancel()とし、状態によらず繰り返し呼べる。静的ファイルのopen/read/close失敗は500へ進み、error_pageの読み取り失敗は元コードを保持して内蔵HTMLへ戻す。error_pageの読み取りも同じFileReadTaskを使う方針とし、組み込みAPIは別途決める。

Routerの戻り値に対応する出力引数だけを使う。

| 戻り値 | 有効な出力 | 呼び出し側の処理 |
|---|---|---|
| RESPONSE_READY | response | 有効な応答を共通のerror_page準備経路へ渡し、本文確定後に送信準備へ進む |
| CGI_REQUIRED | cgi | 既存のCgiExecutorで起動する |
| FILE_READ_REQUIRED | file | Clientが所有するFileReadTaskを開始する |
| FILE_WRITE_REQUIRED | write | Clientが所有するFileWriteTaskを開始する |
| AUTOINDEX_REQUIRED | autoindex | Clientが所有するAutoindexTaskを開始する |

FileReadRequestは絶対パスのpathと配信用contentTypeを持つ。RouteResult・FileReadRequest・routeの正式な共有宣言は[03の共有API](reports/03_config_cgi_upload.md#cgi-api)を正本とする。FileReadRequestを返しただけで読み取り成功やHTTP応答完成とは扱わない。読み取り完了時にrouteを再実行せず、すでに選んだ対象の結果から応答を作る。

<a id="client-file-read-state"></a>

#### ClientのREADING_FILEと基本遷移(2026-11-15合意)

Clientに通常ファイル読み取り中を示すREADING_FILEを追加する。HttpRequestの公開State(INCOMPLETE/COMPLETE/ERROR)とFileReadTaskの結果は別のものであり、これらへClientの状態名を追加しない。

完成した要求をPROCESSINGでRouterへ1回だけ渡し、結果に応じて次へ進む。

| Routerの結果 | Clientの処理・次の状態 |
|---|---|
| RESPONSE_READY | 有効な応答を共通のerror_page準備経路へ渡す。読み取り待ちが不要で本文確定済みならserializeし、WRITING_RESPONSEへ進む。error_page読み取り時の詳細遷移は別途設計 |
| CGI_REQUIRED | 既存のCGI起動経路へ進み、開始成功後はWAITING_CGIで待つ。開始失敗は既存のエラー処理へ進む |
| FILE_READ_REQUIRED | Clientが所有するFileReadTaskを開始し、READING_FILEへ進む。openなどの開始失敗も下記の読み取り失敗として扱う |

| READING_FILEでの結果 | Clientの処理・次の状態 |
|---|---|
| 未完了 | READING_FILEを維持し、EventLoopへ戻って次の周で続きを読む |
| 完了 | EOFとclose成功を確認済みの本文と配信用Content-Typeで静的ファイルの200応答を作り、serialize・送信バッファ準備後にWRITING_RESPONSEへ進む |
| 失敗 | 途中の本文を破棄し、500の応答生成経路へ進む。部分本文の200を送らず、routeを再実行しない |

この完了/失敗の表は通常の静的ファイル読み取りを対象とする。error_pageもREADING_FILEで処理し、ClientのFileReadPurposeで用途を区別する。error_pageは元の応答を保持し、読み取り成功/失敗をfinishErrorPageで確定して送信へ進む。具体的な応答の保持方法と共通処理の宣言は別途決める。読み取り失敗から500を生成する経路も、常にその場でWRITING_RESPONSEへ進めるとは確定しない。例外・切断・期限超過・SIGINT時の遷移と後始末、FileReadTaskの正式APIは未決定。

<a id="file-read-task-operations"></a>

#### FileReadTaskの4操作(2026-11-15合意)

開始・進行・本文取得・中止を次の4操作に分ける。表の呼び出し表記は役割を示すもので、正式なC++宣言は下記のstartから順に具体化する。4操作の宣言、StepResult、start/stepの状態別の呼び出し契約と、privateなPhaseによる5状態の内部表現は合意済み。

| 操作 | 呼び出し表記 | 合意した役割 |
|---|---|---|
| 開始 | start(request) | 読み取りに必要な情報を自分で保持し、対象ファイルを開く。呼び出し元の一時的な指示への参照は残さない |
| 進行 | step() | 最大8 KiBのreadを1回だけ行い、未完了・完了・失敗を区別して返す。EOFを検出したらcloseも確認し、成功した場合だけ完了とする |
| 本文取得 | takeBody(body) | 完了した本文をswapで呼び出し側へ渡す |
| 中止 | cancel() | 保持しているfdを閉じ、途中の本文を解放する |

startの引数と戻り値は次の宣言で合意済み。

```cpp
bool start(const FileReadRequest& request);
```

FileReadRequestはconst参照で受け取り、必要な情報をタスク自身で保持する。呼び出し元の一時的なrequestへの参照・ポインターは残さない。trueは対象ファイルを開けて読み取り開始の準備ができたこと、falseは開始失敗を示す。trueでも本文の読み取り完了や200応答の送信可能を意味しない。FileReadTaskはHTTPステータスコードを返さず、呼び出し側が用途に応じて静的ファイルなら500生成、error_pageなら元コードを保持して内蔵HTMLへ戻す判断を行う。

stepの戻り値はFileReadTaskのpublicな列挙型StepResultとし、引数なしの次の宣言で合意済み。

```cpp
enum StepResult { IN_PROGRESS, COMPLETED, FAILED };
StepResult step();
```

| StepResult | 意味 |
|---|---|
| IN_PROGRESS | 読み取り途中。ClientはREADING_FILEを維持し、次の周で続ける |
| COMPLETED | EOFとclose成功を確認済み。完了した本文を取得できる |
| FAILED | 読み取り中ならread/close失敗を示す。未開始・本文取得済みでは進行できない呼び出しを示し、状態は変更しない。失敗結果の本文を成功応答に使わない |

正のreadでは戻り値の分を本文へ追加する。要求した8 KiBより短くてもEOFと判断せず、readの0とclose成功を確認してCOMPLETEDとする。結果はHTTPコードではなくタスクの進行結果であり、用途に応じたHTTP応答の判断は呼び出し側が行う。

takeBodyの宣言と取得ルールは次のとおり合意済み。

```cpp
bool takeBody(std::string& body);
```

- 完了後の初回だけ本文をswapで出力引数へ渡し、trueを返す。空ファイルの正常な0バイト本文もtrueとする。
- 未開始・未完了・失敗・取得済みの場合は出力引数を変更せず、falseを返す。本文の空判定を取得可否の判断に使わない。
- 出力引数に以前の内容があれば、取得成功時に完了本文で置き換える。swapでタスク側へ移った以前の内容は解放する。
- 本文取得成功時はPhaseをTAKENへ進め、同じ本文を二度渡さない。取得済みboolを別に重ねて持たない。

cancelの宣言と中止ルールは次のとおり合意済み。

```cpp
void cancel();
```

- 保持fdが有効なら一時変数へ取り出し、保持値を-1にしてからcloseを1回だけ呼ぶ。close失敗でも再試行しない。
- タスク内の本文を解放し、以後takeBodyでは取得できないようにする。すでに呼び出し側へ渡した本文は変更しない。
- 未開始・読み取り中・完了・失敗・取得済みのいずれでも呼べる。繰り返し呼んでも二重closeしない。
- デストラクターも同じ解放処理を使い、例外を外へ出さない。デストラクターの前にcancelを呼んだ場合も、同じfdを再度closeしない。
- cancel自体はHTTP応答を生成しない。中止後の接続やHTTP応答の判断は呼び出し側が行う。

同じFileReadTaskを再利用する場合は、必ずcancelで未開始状態へ戻してからstartする。

| startを呼ぶ時点 | 契約 |
|---|---|
| 未開始 | 開始できる。準備成功ならtrue、開始失敗ならfalse |
| 読み取り中・完了・失敗・本文取得済み | falseを返し、現在のfd・本文・状態・保持情報を変更しない。別ファイルを開かない |
| cancel後 | 未開始に戻っているため、新しい読み取りを開始できる |

読み取り失敗後に同じタスクでerror_pageを読む場合も、cancelで片付けてから新しい指示でstartする。startが暗黙に既存のタスクを中止・上書きする方式は採用しない。IDLEからの開始処理で通常の失敗が起きたら、取得済みfdを一度だけ閉じ、本文・保持情報を解放してERRORとしfalseを返す。IDLE以外からのstartは内容/状態を変えない拒否と区別する。再度開始する場合もcancelを挟む。std::bad_allocは通常失敗に変換せず、接続の資源を回収してcloseする。

stepを呼んだ時点の状態に応じた動作も合意済み。

| タスクの状態 | stepの動作 |
|---|---|
| 読み取り中 | 最大8 KiBのreadを1回進め、結果に応じてIN_PROGRESS/COMPLETED/FAILEDを返す |
| 完了・本文未取得 | I/Oを行わずCOMPLETEDを返す。本文は保持する |
| 失敗 | I/Oを行わずFAILEDを返す |
| 未開始・本文取得済み | I/Oを行わず、状態を変えずFAILEDを返す |

未開始・本文取得済みのFAILEDは「進行できない呼び出し」を表し、それ自体でタスクを失敗状態へ変えない。cancel後は未開始の規則に従う。完了/失敗後にstepを繰り返してもread/closeを繰り返さない。

FileReadTaskの内部状態はprivateなPhaseを1つ持ち、複数のboolで表現しない。

```cpp
enum Phase { IDLE, READING, DONE, ERROR, TAKEN };
Phase phase_;
```

| Phase | 意味・既存の操作との対応 |
|---|---|
| IDLE | 未開始。コンストラクターの初期状態とcancel後の状態 |
| READING | start成功後の読み取り中。stepで読み取りを進める |
| DONE | EOFとclose成功を確認済み。本文はまだ取得されていない |
| ERROR | 処理失敗。IDLEからの通常の開始失敗は後始末してこの状態へ進む。std::bad_alloc時はタスク再利用で復旧せず接続を閉じる |
| TAKEN | takeBodyで本文を取得済み |

Phaseは外部へ公開しない。呼び出し側にはboolまたはStepResultを返す。DONEは公開結果COMPLETED、ERRORは公開結果FAILEDに対応するが、IDLE/TAKENでのstepも状態を変えずFAILEDを返すため、戻り値をそのまま内部状態へ代入しない。cancelはどのPhaseからもIDLEへ戻す。独立した中止済み状態や完了/失敗/取得済みのboolを追加しない。

std::bad_allocはfalse/FAILEDへ変換せず、接続の資源を回収してcloseする。具体的な捕捉位置と、切断・期限超過・SIGINT・CGI子専用解放の各経路からcancelへつなぐ手順は未決定。


<a id="error-page-prepare"></a>

#### error_pageの準備とClientへの受け渡し(2026-11-15合意)

エラーページ処理は「元の応答の準備」「ファイル読み取り」「本文の確定」に分ける。Clientが置換対象のHttpResponseを保持し、FileReadTaskによる読み取りが終わってから、rysato側の本文確定処理をその応答へ適用する。読み取り完了時に別の応答へ作り直さず、元のステータス・CGIの検証済み説明文(空も含む)・Allow等の必要ヘッダーを保持する。本文確定まではserialize/送信へ進まない。

HttpResponseに次の準備APIを置く。

```cpp
bool prepareErrorPage(const ServerConfig& config, FileReadRequest& file);
```

| 条件 | 応答と出力引数の扱い | 戻り値 |
|---|---|---|
| 200〜399 | 応答を変更しない。fileは使用しない | false |
| 400〜599でerror_page未指定 | 元コードの内蔵HTMLで本文を確定する。fileは使用しない | false |
| 400〜599でerror_page指定あり | file.pathへ設定済み絶対パス、file.contentTypeへtext/htmlを設定する。Clientが元の応答を保持して読み取りへ進む | true |

trueは読み取り指示が有効でClient側の読み取りが必要であることを示し、読み取り成功を意味しない。falseはこの処理による読み取り待ちが不要であることを示し、HTTP処理の失敗を意味しない。fileはtrueのときだけ有効とする。この関数はファイルを開かず、読み取りの判断と指示の作成までを担当する。通常ファイルの確認と読み取りの具体的な接続方法は、残るAPIの設計で決める。

指定ファイルの読み取り成功時は、その本文を使って保持中の応答の本文・本文関連ヘッダーを確定する。空ファイルも成功した空本文として扱う。読み取り失敗時は元コードの内蔵HTMLへ一度だけ切り替え、元コードを500へ変更しない。本文置換時は既存のCGI-131に従って元の本文関連ヘッダーを除去し、Content-Typeをtext/html、Content-Lengthを最終本文のバイト数とする。error_pageの再選択・再ルーティング・CGI実行・再帰的なエラー生成は行わない。

rysato側はerror_pageの選択・内蔵HTML生成・本文/ヘッダーの置換を担当し、Client側は応答の保持・FileReadTaskの開始/進行・読み取り結果の受け渡しを担当する。共同部分は準備APIと結果を受け取る本文確定APIの契約とする。

本文確定APIはHttpResponseに次の宣言で置く。

```cpp
void finishErrorPage(bool readSucceeded, const std::string& body);
```

| readSucceeded | 本文の確定方法 |
|---|---|
| true | bodyを採用する。空文字列も正常な空本文として扱う |
| false | bodyを使用せず、保持中の元コードの内蔵HTMLを生成する |

どちらの経路でも保持中の応答のステータス・説明文・Allow等の必要ヘッダーを維持する。CGI-131に従って元の本文関連ヘッダーを除去し、Content-Typeをtext/html、Content-Lengthを最終本文のバイト数に揃える。ファイル読み取りやerror_pageの再選択を行わない。対象はerror_pageを適用する400〜599の応答とし、通常応答へ適用するためのAPIとは扱わない。

Clientは保持中の応答へfinishErrorPageを適用してからserialize・送信へ進む。読み取り成功時はFileReadTaskから取得した完成本文を渡し、失敗時はfalseで内蔵HTMLへ切り替える。読み取り結果と本文を別の引数にすることで、正常な空ファイルを読み取り失敗と混同しない。prepareErrorPageがfalseを返して本文確定済みの場合に、さらに読み取り後の確定処理を重ねる経路は作らない。

makeErrorは設定を受け取らず、次の宣言へ変更する。

```cpp
static HttpResponse makeError(int code);
```

makeErrorはサーバーが生成する400〜599のコードと標準説明文を持つ応答を作る。Config参照、error_page選択、内蔵HTML生成、ファイル読み取りは行わない。本文は後続の準備/確定経路で決め、makeErrorからprepareErrorPageを呼ばない。

Routerが生成したエラー、要求解析の失敗、CGIの開始/実行/出力解析の失敗はいずれもmakeError(code)で応答を作り、その後に同じerror_page準備経路へ渡す。正常に返されたCGIの404等は元の検証済み応答をその経路へ渡し、makeErrorでCGI説明文や必要ヘッダーを失わせない。prepareErrorPageは内蔵HTMLで確定するか読み取り指示を作り、読み取りを開始した経路だけfinishErrorPageで本文を確定する。準備/確定前にはserializeしない。

これに伴い、RouterのRESPONSE_READYは有効なHttpResponseが返ったことを示すが、エラー本文の確定・即時送信まで保証しない。呼び出し側で共通のerror_page準備経路へ進める。FILE_READ_REQUIREDは従来どおり静的ファイルの指示だけが有効であり、error_pageを読む際の元応答と指示を同時に返すために、この戻り値の契約を変更しない。

prepareErrorPageの呼び出しはClient側の共通送信準備処理へ集約し、応答ごとに1回だけ行う。RouterのRESPONSE_READY、要求解析エラー、CGIの応答/エラーなどからこの処理へ渡す。RouterやCGI出力パーサーはprepareErrorPageを個別に呼ばず、makeErrorも準備処理を呼ばない。

| 共通送信準備での結果 | 次の処理 |
|---|---|
| prepareErrorPageがfalse | error_pageの読み取り待ちは不要。本文確定済みの応答をserializeして送信準備へ進む |
| prepareErrorPageがtrue | Clientが元の応答を保持し、指示をFileReadTaskへ渡して読み取りを進める |
| 上記の読み取り成功/失敗 | finishErrorPageで本文を確定し、直接serialize・送信準備へ進む。共通送信準備やprepareErrorPageへ戻さない |

error_page読み取りの開始失敗もfinishErrorPage(false, body)による内蔵HTMLへの切り替えへ進める。別のmakeErrorや再準備を呼んで元コードを上書きしない。静的ファイル読み取り成功から組み立てた200応答も共通送信準備へ渡せ、prepareErrorPageはその応答を変更しない。具体的な共通処理のメソッド名・引数は未決定。読み取り用途は下記のClient内の列挙型で区別する。

旧applyErrorPage(const ServerConfig&)とreadErrorPage(path, body)は廃止する。error_pageの選択はprepareErrorPage、ファイル読み取りはFileReadTask、読み取り後の本文/関連ヘッダーの置換はfinishErrorPageへ移す。HttpResponseにはファイル読み取りや所有fdを残さない。buildDefaultErrorBodyとremoveBodyHeadersはprivateな共通補助関数として残し、内蔵HTML生成と本文関連ヘッダーの除去に使う。

ClientはprivateなFileReadPurposeで読み取り用途を区別する。

```cpp
enum FileReadPurpose { STATIC_FILE, ERROR_PAGE };
```

| 用途 | 読み取り成功時 | 読み取り失敗時 |
|---|---|---|
| STATIC_FILE | 完成本文から200応答を作り、共通送信準備へ渡す | 部分本文を破棄し、makeError(500)で元応答を作って共通送信準備へ渡す |
| ERROR_PAGE | 保持中の応答へfinishErrorPage(true, body)を適用し、serialize/送信へ進む | 保持中の応答へfinishErrorPage(false, body)を適用し、内蔵HTMLで確定後にserialize/送信へ進む |

用途はClientのREADING_FILE中だけ参照する。読み取りを開始する際に用途を設定し、「用途なし」の列挙値は追加しない。FileReadTaskは用途やHTTP応答を知らず、同じ読み取りAPIで処理する。FileReadRequestへ用途を重複追加せず、Client側で結果の渡し先を選ぶ。error_pageの開始失敗もERROR_PAGEの失敗処理へ進め、500の再生成やerror_pageの再準備を行わない。

Clientは置換対象のHttpResponseを値メンバーとして保持する。

```cpp
HttpResponse pendingResponse_;
```

共通送信準備へ渡された応答をpendingResponse_に保持し、error_pageの読み取り中も元のステータス・説明文・必要ヘッダーを維持する。読み取り成功/失敗後はこの応答にfinishErrorPageを適用し、本文確定後のpendingResponse_をserializeして送信バッファへ渡す。応答自体のnew/deleteは行わず、Clientと一緒に生成・破棄する。呼び出し元の一時的なHttpResponseへの参照を読み取り待ちの間に残さない。値メンバーなのでClient.hppではHttpResponse.hppをincludeする。

Client側の共通送信準備APIは次の宣言で置く。

```cpp
void prepareResponse(const HttpResponse& response);
```

1. responseを値メンバーpendingResponse_へコピーする。引数の参照を保持せず、呼び出し後に元の応答を破棄できる。
2. pendingResponse_.prepareErrorPageを所属ServerConfigと読み取り指示を使って1回呼ぶ。
3. falseなら本文は確定済みなのでbeginWritingResponseで分離送信を準備する。
4. trueなら用途をERROR_PAGEに設定し、FileReadTaskを開始してREADING_FILEへ進む。開始失敗はpendingResponse_.finishErrorPage(false, body)で内蔵HTMLへ切り替え、再準備せず送信準備へ進む。

読み取り後のfinishErrorPageは保持中の応答へ適用し、prepareResponseへ戻らずserialize/送信へ進む。prepareResponseを呼ぶのは新しく用意した応答を共通準備へ渡す時とし、error_page本文確定後に同じ応答を再投入しない。静的ファイル読み取りの成功/失敗で新しく用意した200/500応答もこのAPIへ渡す。本文確定後のserialize/送信準備は下記のbeginWritingResponseへまとめる。std::bad_alloc時は資源を回収して接続を閉じ、500の再生成は行わない。

<a id="response-ownership-handoff"></a>

#### Clientへの応答の所有権移譲(2026-11-21合意)

```cpp
// Clientのpublic関数
void prepareResponseBySwap(HttpResponse& response);
```

autoindexの成功経路では、Clientが使い終えたローカルの応答をこの操作に渡す。最初にステータス・説明文・ヘッダー・本文をまとめてpendingResponse_とswapし、応答本文をコピーせずに保持する。渡した応答は交換前のpendingResponse_の内容を受け取り、その場で破棄できる。呼び出し側は引き渡し後の元の応答を送信に使わない。

格納後は、コピー経路と同じ共通処理でprepareErrorPageを1回呼び、本文確定後の送信準備へ進む。autoindexの200では本文確定済みの経路を通る。既存のprepareResponse(const HttpResponse&)はコピーで受け取る入口として利用できる。送信準備中のメモリ確保失敗は既存の接続終了方針に従う。

本文確定後の送信準備は、Clientのprivateな次の関数へまとめる。

```cpp
void beginWritingResponse();
```

<a id="split-response-send"></a>

2026-11-21合意で、既存の共通送信準備にヘッダーと本文の分離送信を採用する。

1. 確定済みの応答から、ステータス行・ヘッダー・終端の空行を送信用の文字列に生成する。最終本文からContent-Lengthを決め、本文の有無は既存のHTTP応答規則に従う。
2. 生成したヘッダーと保持済みの本文を別々に扱う。本文をヘッダーへ連結するコピーを省く。送信に使う本文は完了または打ち切りまでClientのpendingResponse_に保持する。
3. 送信準備が整った時点から既存の60秒を計り、WRITING_RESPONSEへ進む。POLLOUTの通知後に各周最大1回・最大8 KiBのsendを行い、ヘッダーを送信し終えてから本文へ進む。部分送信は実際に送ったバイト数だけ位置を進める。本文が空または送信対象外なら、ヘッダーの完了で送信完了とする。部品の切り替えや進行で期限を延長しない。

<a id="response-body-send-lifetime"></a>

#### 分離送信中の本文の保持と回収(2026-11-23合意)

Clientは確定済み本文をpendingResponse_に保持し、送信時はその本文を直接参照する。ヘッダーを生成してから最後の送信が終わるまで、本文を変更せず、pendingResponse_のresetも行わない。

送信完了または打ち切りが確定したら、送信ヘッダーとpendingResponse_の保持領域を回収する。空本文や本文を送らない応答では、ヘッダー送信の完了を起点に回収する。正常完了後にClientが切断待ちへ進む場合も、この時点で送信データを解放する。メモリ確保失敗時は接続終了と破棄による回収に従う。

<a id="response-send-accessors"></a>

#### HttpResponseの送信用API(2026-11-23合意)

```cpp
std::string serializeHeaders() const;
const std::string& getBody() const;
```

serializeHeaders()はステータス行・ヘッダー・終端の空行を生成する。通常応答のContent-Lengthは確定した本文の実バイト数とし、本文を送らない応答では既存のHTTP-51〜58とCGI-73に従う。204/304は本文とContent-Lengthを省き、205は本文を省いてContent-Lengthを0とする。

getBody()は送信対象の保持本文をconst参照で返す。本文を送らないステータスでは、有効な寿命を持つ空文字列を参照として返す。これによりClientは応答コードによる本文の要否を再判定せず、ヘッダーの送信後に本文の残量を確認できる。参照は応答の変更・reset・破棄までに使い切る。送信位置の表現は実装設計で具体化する。

prepareResponse内でprepareErrorPageがfalseを返した場合と、error_pageの開始失敗/読み取り成功/失敗に対してfinishErrorPageで本文を確定した場合の両方から、この関数を呼ぶ。beginWritingResponseはprepareErrorPageやprepareResponseを呼び直さず、error_pageの選択/読み取りを行わない。ソケットへのsendも行わず、実際の送信は単一pollの準備通知後に進める。serialize/reset等のstd::bad_allocは下記の接続終了方針に従う。計時失敗など他の失敗の具体的な後始末は別途決める。

<a id="response-reset-and-allocation-failure"></a>

#### 保持応答の解放とメモリ確保失敗(2026-11-15合意)

HttpResponseに次の公開APIを追加する。

```cpp
void reset();
```

resetは本文・ヘッダー等の保持領域を解放し、ステータスと説明文をコンストラクターと同じ200/OKへ戻す。空に見せるだけで大きな本文の保持領域を残す処理とはしない。送信バッファ・送信オフセット・Clientの状態や期限は変更しない。pendingResponse_自体は値メンバーのまま残す。分離送信では送信完了または打ち切りが確定してから保持領域を回収する。

接続処理でstd::bad_allocが起きた場合は、その接続の処理を中止し、ファイルfd・保持本文・送信バッファ等を回収して接続を閉じる。対象には応答コピー、本文蓄積、error_pageの内蔵HTML生成/本文置換、serialize、resetなどを含む。FileReadTaskの通常のopen/read失敗をfalse/FAILEDで扱う規則とは区別し、std::bad_allocを通常のI/O失敗へ変換して500や内蔵HTMLを生成し直さない。捕捉後の回収は追加確保を伴う復旧をせず、Clientの破棄とFileReadTaskの既存の解放経路へつなぐ。関連するCGIがあれば既存の停止/回収も行い、ほかの接続の処理は継続する。

bad_alloc後にresetで200/OKへ戻して復旧する処理や、reset失敗時に再度resetする処理は行わない。送信データの回収中にresetが失敗した場合は、接続を閉じて破棄による回収へ進む。各呼び出し階層の具体的な捕捉位置・所有資源の接続方法は実装設計で確認するが、メモリ不足時に500を再生成する旧方針は採用しない。

元の応答をClientが値メンバーpendingResponse_で保持する方針、makeError/prepareErrorPage/finishErrorPageの宣言と役割分担、Client側への共通送信準備の集約、FileReadPurposeによる用途区別、旧applyErrorPage/readErrorPageの廃止・上記の動作は合意済み。prepareResponse/beginWritingResponseの宣言と処理順序、std::bad_alloc時の接続終了方針も合意済み。その他の異常時の詳細は未決定。RouterのRESPONSE_READYとFILE_READ_REQUIREDはそれぞれ有効な出力が1つという既存契約を維持し、両出力が有効とは扱わない。

#### 変更に伴う未決定事項

| 対象 | 次に具体化する内容 |
|---|---|
| Client | error_page読み取り時と例外・切断・期限超過・SIGINT時の遷移/後始末。FileReadTaskの値メンバー保持、Clientのコピー禁止、READING_FILEと静的ファイル読み取りの基本遷移は合意済み |
| FileReadTask | bad_allocの具体的な捕捉位置と資源回収経路、切断・期限超過・SIGINT・CGI子専用解放への組み込み。bad_allocでその接続を閉じる方針は合意済み。通常の開始失敗時の後始末とERRORへの遷移、4操作の宣言と状態別の呼び出し契約、privateなPhase(IDLE/READING/DONE/ERROR/TAKEN)、StepResultの3種類と結果の意味、本文の一度だけのswapによる引き渡し、cancelの再呼び出し安全性と未開始への復帰、cancelを挟む再利用、closeを再試行しない解放、デストラクターで同じ解放処理を使う方針、未開始時のfd=-1/空本文、コピー禁止は合意済み |
| EventLoop | 2026-11-18に進行可能なファイルタスクがある間のpoll待ち時間0を採用。各タスクを各周最大1回進める。poll復帰後はソケット/CGIイベントを先に処理し、その後に生存中の接続のファイルタスクを期限確認付きで進める。ソケットのPOLLOUTでファイル読み取りを駆動しない |
| 読み取りの期限 | 2026-11-17に進行で延長しない固定期限方式を採用。起点は各タスクのstart直前。2026-11-18に60秒を採用案とした。静的読み取りの期限超過はcancel後に500を共通送信準備へ渡す。詳細は下記のファイル処理の期限方式を参照する |
| error_page | 計時失敗等の異常時の詳細遷移。bad_alloc時の接続終了は合意済み。送信中の本文はpendingResponse_に保持し、完了または打ち切り時に回収する。prepareResponse/beginWritingResponseの宣言/処理順序、pendingResponse_の値メンバー保持、FileReadPurposeの2用途、makeError/prepareErrorPage/finishErrorPageの役割/宣言/動作は合意済み。通常の読み取り失敗時に元コードを保持して内蔵HTMLへ戻す規則と、メモリ不足の接続終了は区別する |
| fd継承 | 2026-11-18にopen直後のF_SETFD/FD_CLOEXEC設定と、設定失敗時の回収を採用。CGI子の起動失敗時は複製fdとメモリを解放する。保存途中ファイルを子側から削除しないFileWriteTask::cleanupInChild()を2026-11-19に採用。呼び出し口はClient::cleanupInChild()を2026-11-19に採用 |

アップロード保存は次節の2026-11-16合意で分割する。autoindexは2026-11-19に列挙/ソート/HTML生成の段階分割を採用し、Client所有のAutoindexTaskを採用し、API/処理量などを次に具体化する。大きい本文のserializeの分割と資源量も今後の設計対象である。

#### 効果と限界

改善対象は、1接続のファイルを読み切るまでEventLoopへ戻らないことによる他接続の待ち時間である。各readやopen/stat/close自体のディスク待ちを非同期にする設計ではなく、完全なノンブロッキングを保証しない。本文全量の保持とserialize時のコピーも残る。静的ファイル専用サイズ上限・全体メモリ上限を決めたものではない。Linux実環境で他接続/CGIの進行・期限検出・資源回収を検証するまで評価合格とは扱わない。

<a id="file-write-design"></a>

### アップロード保存の分割と受け渡し(2026-11-16合意)

- HTTP要求とmultipart全体の構文・単一ファイル・保存名を検証した後に保存を開始する。受信中のファイル作成・保存は行わず、全量保持の方針を維持する。
- Routerは検証済みの保存先絶対パス・保存名・保存データをFileWriteRequestへ設定し、FILE_WRITE_REQUIREDを返す。Router内でファイルを作成・書き込みせず、Clientが保存タスクを開始する。指示を返した時点では保存成功・応答完成とは扱わない。
- ClientはFileWriteTaskを値メンバーとして所有する。タスクがfd・保存先パス・保存データ・書き込み位置などの途中状態を管理し、open/write/closeと途中ファイルの後始末を担当する。開始・進行・保存名取得・キャンセルの4操作は下記で合意済み。内部状態は下記のPhaseを採用する。具体的な資源メンバー名は未決定。
- 保存本文はFileWriteRequest::bodyからタスク所有のstd::stringへswapで移し、タスク開始のための本文コピーを追加しない。引き渡し後の指示側bodyは空とし、以後の保存にはタスク所有の本文と書き込み位置を使う。タスクは指示側bodyへの参照・ポインターを残さない。開始APIは下記の合意に従い、保存先・保存名の保持とopenに成功してから本文を移す。開始失敗なら指示側の本文を残す。元のHttpRequestが保持する要求本文や、multipartからFileWriteRequestを作る際のコピーは、このswapで解放・削減されたとは扱わない。
- 新規作成は既存のO_CREAT/O_EXCL/O_WRONLY、mode 0600を維持する。同名409、open失敗の分類、write後のerrno分岐禁止もHTTP-40・41に従う。
- EventLoopの各周で各タスクのwriteを最大1回・最大8 KiB行う。短い正のwriteでは位置を進めて未完了なら次の周へ戻り、0以下は失敗とする。空ファイルも含め、保存全量とcloseの成功を確認した後だけ200と保存名の結果HTMLを生成し、共通送信準備へ渡す。Routerを再実行しない。
- 保存失敗時はfdの後始末後、今回新規作成した途中ファイルだけをstd::removeで削除する。削除失敗でも元の500を保持し、パスをログへ記録する。既存対象は削除しない。成功応答HTMLは下記の分担に従ってHTTP側が生成する。結果取得は下記のtakeFilenameを採用する。切断・期限超過・例外時の詳細な回収経路は未決定。
- 共有宣言の正本は[03](reports/03_config_cgi_upload.md#cgi-api)。routeの末尾にFileWriteRequest& writeを追加し、handlePostにも同じ出力を追加する。GET/DELETEには書き込み指示を追加しない。戻り値に対応する出力だけが有効であり、CGI用の型とプロセス側APIは維持する。
- 分割は個々のopen/write/closeのディスク待ちを非同期化しない。大きいアップロードと小さいGET/CGIを並行させ、期限確認と他接続が進むことを実環境で検証する。multipart全体検証や本文のコピーにかかる時間・全量保持のメモリも残る。

本文のswapによる引き渡しと、引き渡し後にタスクが本文を所有することは合意済み。開始成功時だけ本文を移し、開始失敗時は指示側へ残す契約も合意済み。進行APIは下記のstepを採用した。保存名の結果取得APIも下記で合意済み。キャンセルと破棄時の方針も下記で合意済み。内部状態はPhaseの5種類を採用した。startの状態別の呼び出し契約も合意済み。stepの状態別の呼び出し契約も合意済み。通常の開始失敗時の保持情報の解放も下記で合意済み。コピー禁止も2026-11-17に合意済み。ClientのWRITING_FILEと基本遷移も2026-11-17に合意済み。ファイル読み書きの固定期限方式も2026-11-17に合意済み。2026-11-18に期限を各60秒の採用案とした。静的読み取り/保存の期限超過は回収後500とすることも合意済み。poll待ち時間0の方針も2026-11-18に合意済み。ソケット/CGIイベントを先に処理する順序も合意済み。新規タスクのstepを次の周から行うことも合意済み。Clientの本体削除を周末まで遅らせる方針も合意済み。今周開始フラグによるstep対象管理も2026-11-19に合意済み。全体の期限確認をpoll復帰後・イベント前に置くことも2026-11-19に合意済み。未決定なのは、周末除去の具体形、Client::cleanupInChild()採用後の解放実装と異常時の回収である。これらを実装済み・設計完了とは扱わない。

<a id="file-write-start"></a>

#### FileWriteTaskの開始API(2026-11-16合意)

```cpp
int start(FileWriteRequest& request);
```

非const参照で受け取り、成功時は0、通常の開始失敗時はHTTP-41の分類に沿ったHTTPエラーコードを返す。同名対象409、権限不足/読み取り専用403、保存名によるパス長制限超過400、保存先消失/その他500とする。タスク自身はHttpResponseやエラー本文を生成せず、呼び出し側がmakeError(code)から共通送信準備へ進める。

保存先パス・保存名をタスク自身の値として保持し、既存の新規作成条件でopenする。その両方に成功してから、指示のbodyをタスク所有の空の文字列へswapで移して書き込み位置を0にする。開始失敗時は本文を移さず、request.bodyを変更しない。開始成功後は一時的なFileWriteRequestを破棄でき、タスクはそのパス・名前・本文への参照やポインターを残さない。

戻り値0は保存開始の準備完了だけを表し、writeや全量保存・closeの成功を表さない。空本文も保存完了は後続の進行処理で確認する。std::bad_allocは通常の開始失敗コードへ変換せず、既存の接続終了方針に従う。startの状態別の呼び出し契約は下記に従う。通常の開始失敗でERRORへ進む方針は合意済み。通常の開始失敗時の解放は下記に従う。例外時の具体的な回収手順は別途定める。

<a id="file-write-step"></a>

#### FileWriteTaskの進行API(2026-11-16合意)

```cpp
// FileWriteTaskのpublicな列挙型と操作
enum StepResult { IN_PROGRESS, COMPLETED, FAILED };
StepResult step();
```

start成功後の保存を進める。各呼び出しでwriteを最大1回行い、要求するバイト数は残量と8 KiBの小さい方とする。正の戻り値だけ書き込み位置を進め、残量があればIN_PROGRESSを返してEventLoopへ戻す。同じ呼び出し内で残りを繰り返し書かない。

最後のデータを書き終えたらcloseし、成功を確定してDONEへ進め、保持本文と保存先パスを解放してCOMPLETEDを返す。保存名はtakeFilenameまで保持する。空本文はwriteを呼ばず、close成功でCOMPLETEDとする。完了の判断は読み取りのEOFではなく、保持本文の全バイトを書き終えたことに基づく。

writeの0・負値、またはclose失敗はFAILEDとする。write直後のerrnoで分岐しない。WRITING中の通常の保存失敗では、下記の失敗時回収をstep内で済ませてからFAILEDを返し、呼び出し側が500応答を共通送信準備へ渡す。タスクはHttpResponseを生成しない。COMPLETEDになる前に成功応答を生成・送信しない。

保存中以外のstepの返却契約は下記の状態別契約に従う。内部状態とcancelの契約も合意済み。closeと削除を重複させない資源メンバーの具体形と、異常経路での回収手順は別途定める。

<a id="file-write-success-response"></a>

#### 保存完了後の応答生成の分担(2026-11-16合意)

FileWriteTaskは保存結果と検証済みの保存名を呼び出し側へ渡し、HTTPステータス行・ヘッダー・結果HTML・HttpResponseを生成しない。COMPLETEDは全量の書き込みとcloseの成功を表し、この成功を確認してから、rysato側のHTTP応答生成APIが保存名をHTMLエスケープした結果HTMLと200応答を作る。Content-Typeはtext/htmlとし、Content-Lengthは既存の応答生成規則に従う。公開URLやLocationを自動推定しない。

Clientは生成された応答を既存のprepareResponse(const HttpResponse&)へ渡して共通送信準備を行う。Router::routeを再実行せず、保存完了後にファイルを再保存しない。失敗時は既存どおり、開始失敗ならstartのエラーコード、保存途中の通常の失敗なら500から共通送信準備へ進む。

成功応答の生成には下記のHttpResponse::makeUploadSuccessを採用する。保存名は下記のtakeFilenameで完了後の初回だけ取得する。保存タスクの完了と、その後の応答生成・送信の成功は別に扱い、例外や切断時の詳細な回収は状態/キャンセル設計で定める。

<a id="upload-success-api"></a>

#### アップロード成功応答の生成API(2026-11-16合意)

```cpp
// HttpResponseのpublicなstatic関数
static HttpResponse makeUploadSuccess(const std::string& filename);
```

検証済みの保存名を受け取り、標準説明文による200 OKと、保存名をHTMLエスケープして表示する結果HTMLを持つHttpResponseを値で返す。Content-Typeはtext/htmlとし、最終本文のバイト数に対応するContent-Lengthは既存のserialize規則に従う。保存名の空白・非ASCIIを保持し、表示用のHTMLエスケープを保存先パスや実際のファイル名へ適用しない。

本関数はファイル操作・保存の成否判定・Config参照・公開URL/Locationの自動生成・error_page準備を行わない。Clientがタスクの保存成功を確認した後に呼び、戻り値をprepareResponseへ渡す。入力filenameへの参照を応答に残さず、返却後に入力を破棄しても応答を利用できる。メモリ確保失敗は既存のbad_alloc時の接続終了方針に従う。

<a id="file-write-take-filename"></a>

#### FileWriteTaskの保存名取得API(2026-11-16合意)

```cpp
bool takeFilename(std::string& filename);
```

全量の保存とcloseに成功し、まだ保存名を取得していないときだけ、タスクが保持する検証済み保存名を出力引数へswapで渡してtrueを返す。取得は一度だけとする。未開始・保存中・開始/保存失敗・取得済みの場合はfalseを返し、出力引数を変更しない。

取得成功後は呼び出し側の文字列が保存名を所有し、タスクの寿命に依存せずmakeUploadSuccessへ渡せる。出力引数に以前から内容があった場合は、その内容がswapでタスク側へ移るため、不要な旧内容をタスク内に保持し続けない。成功ファイルの削除や再保存をこの操作で行わず、ファイルへのI/Oも行わない。保存先パスを結果として追加で返す構造体は設けない。

保存結果の成否はstepのStepResult、成功時の保存名はtakeFilenameで受け取る。タスクはHttpResponseを生成せず、Clientが取得した保存名をHttpResponse::makeUploadSuccessへ渡し、その応答をprepareResponseへ渡す。cancel後は未開始に戻るため、takeFilenameは出力不変でfalseを返す。内部状態の具体的な列挙値は下記のPhaseを使う。

<a id="file-write-cancel"></a>

#### FileWriteTaskのキャンセルと破棄(2026-11-16合意)

```cpp
void cancel();
```

保存途中であれば、保持fdを閉じた後に今回新規作成した途中ファイルだけをstd::removeで削除する。openに成功していない同名の既存対象は削除しない。全量の書き込みとcloseに成功したファイルは、保存名をまだ取得していなくても、応答生成・送信の前でも削除しない。完了後のクライアント切断や応答生成失敗によって保存済みファイルを巻き戻さない。close失敗は保存成功に含めず、途中ファイルの後始末へ進む。

保持fdはcloseを呼ぶ前に所有状態から外し、close失敗でも同じfdを再試行しない。途中ファイルの削除も一度の試行として管理し、step等ですでに後始末した資源をcancelで再度close・削除しない。削除失敗では対象パスをログへ記録する。通常の保存失敗の500を後始末で上書きせず、切断時など応答を返さない経路で新たなHTTP応答を生成しない。

本文・パス・保存名などの保持データを解放し、書き込み位置を含むタスクの状態を未開始へ戻す。どの状態でも繰り返し呼べ、二重close・二重削除を起こさない。cancel後は保存名を取得できず、takeFilenameは出力不変でfalseを返す。取得済みで呼び出し側が所有する保存名は変更しない。

デストラクターも同じ後始末を行い、例外を外へ出さない。cancelと破棄時の解放・削除の具体的な例外対策と、Client/EventLoopの切断・期限超過・SIGINT・CGI子専用解放への組み込みは、後続の状態/異常経路の設計で具体化する。

<a id="file-write-phase"></a>

#### FileWriteTaskの内部状態(2026-11-16合意)

```cpp
private:
    enum Phase { IDLE, WRITING, DONE, ERROR, TAKEN };
    Phase phase_;
```

| Phase | 意味と遷移 |
|---|---|
| IDLE | 初期状態とcancel後の未開始状態 |
| WRITING | start成功後。本文を所有し、保存を進めている状態 |
| DONE | 全量の書き込みとclose成功後。保存名は未取得 |
| ERROR | 通常の開始失敗または保存失敗後 |
| TAKEN | takeFilenameが初回成功し、保存名を渡した後 |

完了・失敗・取得済みを別々のboolで重複保持せず、このPhaseで区別する。保存開始でWRITING、通常の開始/保存失敗でERROR、全量保存とclose成功でDONE、保存名の取得成功でTAKENへ進む。cancelはどの状態からもIDLEへ戻す。DONE/TAKENは保存成功済みであり、cancelや破棄で保存済みファイルを削除しない。ERRORでの途中ファイルの後始末と二重処理防止は既存のcancel契約を維持する。

Phaseはprivateな内部状態であり、公開するstepのStepResultと区別する。startの再呼び出し契約は下記で合意済み。stepの状態別の再呼び出し契約も下記で合意済み。通常の開始失敗時の解放は下記で合意済み。コピー禁止の宣言は下記で合意済み。資源メンバーの具体形は別途決める。close済み・削除試行済みの資源はPhaseだけで推測せず、実際の所有状態と既存の二重処理防止契約に従って扱う。

<a id="file-write-start-state"></a>

#### FileWriteTask::startの状態別契約(2026-11-16合意)

startはIDLEのときに保存開始の処理へ進む。WRITING/DONE/ERROR/TAKENで呼ばれた場合は500を返し、現在のPhase・保持データ・fd・ファイル・入力のFileWriteRequestをすべて維持する。openや本文のswapは行わない。この500は呼び出し手順の誤りを表し、タスクをERRORへ変更する指示にはしない。

再利用する場合は、先にcancelで資源を整理してIDLEへ戻してからstartを呼ぶ。通常の開始失敗でERRORになった場合も同じ手順とする。cancel時のファイルの扱いは既存契約に従い、保存途中は今回作成したファイルを削除し、DONE/TAKENの保存済みファイルを残す。

<a id="file-write-step-state"></a>

#### FileWriteTask::stepの状態別契約(2026-11-16合意)

| 呼び出し時のPhase | 動作と返却値 |
|---|---|
| WRITING | 既存のstep契約に従って保存を進め、IN_PROGRESS/COMPLETED/FAILEDを返す |
| DONE | Phaseを維持してCOMPLETEDを返す。ファイルI/Oは行わない |
| ERROR | Phaseを維持してFAILEDを返す。ファイルI/Oは行わない |
| IDLE / TAKEN | Phaseを維持してFAILEDを返す。ファイルI/Oは行わない |

DONE/ERROR/IDLE/TAKENでの呼び出しでは、保持データとファイルを維持し、write・close・削除を繰り返さない。IDLE/TAKENで返すFAILEDによって内部PhaseをERRORへ変更しない。DONEでCOMPLETEDを繰り返し返しても、保存名の取得はtakeFilenameの初回成功だけとする。

<a id="file-write-start-failure"></a>

#### FileWriteTaskの通常の開始失敗時の解放(2026-11-16合意)

IDLEからのstartが通常の開始失敗となった場合、タスクが保持したパス・保存名・本文を解放し、書き込み位置を0へ戻す。取得済みfdがあれば所有状態から外して一度だけcloseし、fd=-1とする。closeの再試行と、後続のcancel/破棄による二重closeを防ぐ。内部状態はERRORへ進め、startは既存の分類によるエラーコードを返す。

入力のFileWriteRequestはパス・保存名・本文を含めて維持する。開始成功時だけ本文をswapする既存の順序に従う。今回新規作成した途中ファイルがあれば既存の後始末契約を適用し、作成していない既存対象を保護する。後始末に必要なパスは削除試行と失敗時のログ記録を終えてから解放する。

ERRORになったタスクの再利用はcancelでIDLEへ戻してから行う。WRITING/DONE/ERROR/TAKENでstartが500を返す場合は、既存の状態維持契約を適用する。std::bad_allocは既存の接続終了方針に従い、例外時の具体的な資源回収経路は別途定める。

<a id="file-write-noncopyable"></a>

#### FileWriteTaskのコピー禁止(2026-11-17合意)

```cpp
private:
    FileWriteTask(const FileWriteTask&);
    FileWriteTask& operator=(const FileWriteTask&);
```

C++98のprivateなコピーコンストラクターとコピー代入演算子を宣言し、定義しない。FileWriteTaskのfdと途中ファイルの後始末を担う所有者を一つに保ち、コピーによる二重close・二重削除を防ぐ。

ClientはFileWriteTaskを値メンバーとして所有する。Client自身のコピー禁止と、EventLoopがClientへのポインターを管理する既存の方針に従う。FileWriteTaskをClientと一緒に生成・破棄し、タスク用のnew/deleteを追加する必要はない。

<a id="client-file-write-state"></a>

#### Clientの保存中の状態と基本遷移(2026-11-17合意)

ClientのStateにWRITING_FILEを追加する。

| 契機 | 処理と遷移 |
|---|---|
| PROCESSINGでFILE_WRITE_REQUIRED | Client所有のFileWriteTaskを開始し、startが0なら元のHttpRequest本文を解放してWRITING_FILEへ進む |
| startの通常の開始失敗 | 返されたHTTPコードでmakeErrorを生成し、prepareResponseへ渡す |
| WRITING_FILEでIN_PROGRESS | WRITING_FILEを維持し、次のEventLoop周で進める |
| WRITING_FILEでCOMPLETED | takeFilenameで保存名を取得し、makeUploadSuccessの戻り値をprepareResponseへ渡す |
| WRITING_FILEでFAILED | タスク側で後始末済みの結果を受け、makeError(500)の戻り値をprepareResponseへ渡す |

成功応答は既存の共通送信準備を経てWRITING_RESPONSEへ進む。エラー応答も共通送信準備を使い、error_page設定があれば既存のREADING_FILE経路へ進む。保存完了後はRouterを再実行せず、保存処理の再開もしない。切断・期限超過・SIGINT・例外時の詳細な遷移と回収、COMPLETED後のtakeFilenameが契約に反してfalseだった場合の扱いは後続の異常経路設計で具体化する。

<a id="file-write-step-failure"></a>

#### FileWriteTaskの保存失敗時の回収(2026-11-17合意)

WRITING中のwriteが0・負値を返した場合、または最後のcloseが失敗した場合は、step内で次の順序の回収を済ませ、ERRORへ進んでFAILEDを返す。

1. 保持fdがあれば所有状態から外してfd=-1とし、closeを一度だけ呼ぶ。最後のcloseですでに所有状態から外したfdは、close失敗時も再試行しない。
2. 今回新規作成した途中ファイルをstd::removeで一度だけ削除する。削除失敗時は対象パスをログへ記録し、元の保存失敗に対応する500を維持する。
3. 削除試行と必要なログ記録後に、タスクの本文・パス・保存名を解放し、書き込み位置を0へ戻す。
4. PhaseをERRORとしてFAILEDを返す。

通常の失敗結果を受けたClientはmakeError(500)から共通送信準備へ進む。後続のstep/cancel/破棄は回収済み資源を再びclose・削除しない。ERRORからのstepは状態を維持してFAILEDを返す。IDLE/TAKENでstepがFAILEDを返す場合は、それぞれの状態維持契約を適用する。

削除失敗時のファイルはディスク上に残り得るため、失敗ログで対象を確認できるようにする。ログ記録や解放経路での例外の具体的な処理は、既存のbad_alloc時の接続終了とデストラクターの例外非送出の方針に沿って後続の異常経路設計で定める。

<a id="file-write-success-release"></a>

#### 保存成功時の本文とパスの解放(2026-11-17合意)

全量の書き込みとcloseに成功した時点で、stepはタスクの保存本文と保存先パスの保持領域を解放し、保存名だけをtakeFilenameまで保持する。COMPLETEDを返す前にこの解放を行い、応答生成・送信中のタスクに保存本文を保持し続けない。空ファイルも同じ完了処理を使う。

close成功による保存完了をDONEとして確定してから不要な保持情報を解放し、既存の成功ファイル保護契約に従う。解放経路で例外が発生しても、cancelや破棄では保存成功済みファイルを保護する。DONEでstepを再度呼んだ場合は既存どおりI/OせずCOMPLETEDを返し、保存名はtakeFilenameで初回だけ取得できる。

対象はFileWriteTaskが保持する本文とパスである。元のHttpRequestの要求本文は下記の開始成功後の解放方針に従う。本文解放APIは下記のreleaseBodyを採用する。

<a id="upload-request-body-release"></a>

#### アップロード元の要求本文の解放時期(2026-11-17合意)

FILE_WRITE_REQUIREDの処理でFileWriteTask::startが0を返し、保存本文の所有がタスクへ移ったことを確認した後、Clientが元のHttpRequestの要求本文の保持領域を解放する。multipart全体の検証と抽出はRouterで完了しており、以後の保存はタスク所有の本文を使う。タスクを進める前に元本文の解放を行う。

本文解放後、要求のメソッド・URL・ヘッダー・解析完了状態など、本文以外の情報は保持する。後続の保存処理と応答生成は解放済み要求本文への参照・ポインターを利用せず、Routerを再実行しない。解放操作には下記のreleaseBodyを使い、COMPLETE後にClientが呼ぶ。

開始失敗時は既存の契約に従ってFileWriteRequestを保持し、通常のエラー応答経路へ進む。本決定の解放タイミングはアップロード保存の開始成功時に適用する。CGIなど他の処理経路の要求本文の解放時期は個別に扱う。multipart抽出時からタスク開始までの本文の重複保持は引き続きピークメモリの測定対象とする。

要求本文の解放中にstd::bad_allocが発生した場合は既存の接続終了方針に従い、開始済みタスクの後始末にはcancelの契約を適用する。例外捕捉と呼び出し順序の具体的な組み込みは異常経路設計で定める。

<a id="request-release-body-api"></a>

#### HttpRequestの本文解放API(2026-11-17合意)

```cpp
// HttpRequestのpublic関数
void releaseBody();
```

解析がCOMPLETEになった後、Clientが本文を利用する処理への受け渡しを終えてから呼ぶ。アップロードではFileWriteTask::startが0を返した直後、分割保存を進める前に呼ぶ。

body_を空にし、空のstd::stringとのswap等で本文の保持領域を解放する。COMPLETE後は繰り返し呼べる。解放後にgetBody()を呼ぶと空文字列を参照する。要求のメソッド・URL・ヘッダー・解析完了状態・エラーコードなど、本文以外の保持情報は維持する。呼び出し側はそれまでに取得した本文への参照・ポインターを以後の処理で利用しない。

元の要求本文を再び蓄積する処理は行わず、COMPLETE後のappendDataには既存の終端状態の契約を適用する。INCOMPLETE/ERRORでreleaseBodyを呼んだ場合は、そのまま戻る。本文・保持領域・解析段階・状態・エラーコードを含む全保持データを維持する。INCOMPLETEならその後のappendDataで通常どおり解析を続け、ERRORなら確定済みエラーを保持する。戻り値はvoidのままとする。本文解放中のメモリ確保失敗は既存の接続終了方針に従う。

<a id="file-task-timeout-policy"></a>

### ファイル読み書きの期限方式(2026-11-17合意)

静的ファイル・error_pageの読み取りとアップロード保存には、開始時刻を基準とする固定期限方式を採用する。read/writeが正のバイト数を返して進行しても、期限を延長しない。途中まで進んでいても、期限に達したら超過時の処理へ進む。

起点は各タスクのstart直前とする。error_pageの期限は下記の2026-11-18合意に従う。2026-11-18に静的ファイル読み取り・保存・error_pageの期限を各60秒の採用案とした。静的ファイル読み取り/保存の期限超過は下記の回収後500の契約に従う。既存の受信60秒・送信60秒との接続、計時失敗は下記の回収/接続終了に従う。EventLoopで期限を確認する全体の順序は後続の共同設計で定める。

ClientはFileReadTask/FileWriteTaskのstartを呼ぶ直前に開始時刻を記録し、開始準備とopenを含めて経過時間を測る。開始前のRouterによる対象検証やmultipart全体の解析は、このファイル処理期限の計測区間より前に行う。起点の記録は進行ごとに更新しない。error_pageへ移る際も、その読み取りタスクのstart直前に新しい開始時刻を記録する。

各60秒はチームの採用案として、大きいファイルの配信・保存と複数接続の負荷試験で検証する。期限は各タスクのstart直前から計り、read/writeの進行で延長しない。error_pageへ進む場合は新しい60秒を計る。期限の値は実環境の検証結果に応じて見直し得る。

各open/read/write/closeは同期呼び出しであるため、期限の検出はEventLoopへ制御が戻った時点で行う。単発のシステムコール中に期限が来た場合も、その呼び出し自体の中断を前提にせず、復帰後の期限確認と処理順序を具体化する。

<a id="error-page-timeout-budget"></a>

#### error_pageの独立した固定期限(2026-11-18合意)

error_pageを読み取る際は、元の静的ファイル読み取り・アップロード保存などの処理に費やした時間にかかわらず、error_page用の新しい固定期限を設ける。Clientはerror_page用FileReadTaskのstart直前に開始時刻を記録し、読み取りが進行しても期限を延長しない。

error_pageの読み取りが期限に達した場合は、タスクをcancelしてfdと部分本文を解放し、保持中の元応答にfinishErrorPage(false, body)を適用する。元のステータス・説明文・保持対象ヘッダーを維持した内蔵HTMLで本文を確定し、beginWritingResponseへ進む。読み取り失敗時も既存の同じ内蔵HTML生成経路を使う。

この遷移でerror_pageの再選択や読み取りの再開始を行わず、送信準備へ進む。bad_alloc時は既存の接続終了方針に従う。error_pageも60秒を採用案とする。計時失敗は下記の回収/接続終了に従う。切断時の詳細とEventLoopでの期限確認順序は後続の設計で定める。元のファイル処理とerror_pageを順に行う接続では、それぞれの期限に加えて送信期限が適用されるため、接続全体の所要時間を受入試験で確認する。

<a id="file-task-timeout-response"></a>

### 静的ファイル読み取り・保存の期限超過(2026-11-18合意)

静的ファイル読み取りまたはアップロード保存の期限超過を検出したら、Clientは対象タスクをcancelして資源を回収し、HttpResponse::makeError(500)の戻り値をprepareResponseへ渡す。

静的読み取りは保持fdと部分本文を解放する。保存中は保持fdを閉じた後、今回作成した途中ファイルだけを削除し、保持データを解放する。削除失敗は対象パスをログへ記録して500を維持する。close・削除の再試行と二重処理は既存のcancel契約に従って防ぐ。保存とcloseに成功済みのファイルは既存の成功ファイル保護契約に従う。

500応答は共通送信準備を経由し、設定されたerror_pageがあれば自身のstart直前から新しい60秒の読み取り期限を設ける。error_page自体の期限超過は、元の500を保持した内蔵HTMLへの切り替えで処理する。送信バッファ準備後は既存の送信60秒を適用する。

本決定は期限超過を検出した場合の応答と回収を定める。step前後の期限確認と完了優先は下記に従う。start復帰時の確認は下記に従う。EventLoop全体の期限確認は下記のpoll復帰後・イベント前の契約に従う。計時失敗・切断・bad_allocなどの異常経路は既存方針と後続の共同設計に従う。

<a id="file-step-timeout-order"></a>

### ファイルタスクのstep前後の期限確認(2026-11-18合意)

静的ファイル読み取り・error_page読み取り・アップロード保存で、次の順序を使う。

1. stepを呼ぶ前に期限を確認する。期限に達していればstepを呼ばず、用途別の期限超過処理へ進む。
2. 期限内ならstepを一度呼ぶ。
3. COMPLETEDなら完了を優先し、既存の成功経路へ進む。最後の同期I/Oから戻った時点で期限を過ぎていても、完了した結果を採用する。
4. FAILEDなら既存の失敗経路へ進む。
5. IN_PROGRESSなら復帰後の時刻で期限を再確認する。期限に達していれば用途別の期限超過処理へ進み、期限内なら次の周へ戻す。

保存のCOMPLETEDは全量保存とclose成功を表し、保存済みファイルを保護して成功応答へ進む。静的読み取りは成功本文、error_pageは元の応答へ適用する取得本文を、それぞれ既存の完了経路で受け取る。いずれも期限に合わせて進行時刻を更新しない。

期限超過の静的読み取り/保存は回収後500、error_pageは回収後に元のステータスを保持した内蔵HTMLへ進む。同期システムコールの復帰に時間がかかった場合、60秒を超えてCOMPLETEDが採用され得ることを並行負荷試験で測定する。start復帰時の確認は下記に従う。計時失敗は下記の回収/接続終了に従い、EventLoop全体の進行順序は下記の全体期限確認の契約に従う。

<a id="file-start-timeout-order"></a>

### ファイルタスクのstart復帰時の期限確認(2026-11-18合意)

Clientはstart直前の開始時刻を基準に、start成功後すぐに時刻を再確認する。静的読み取り・error_page読み取り・保存のいずれも、開始準備とopenを含む経過時間が期限に達していたら、stepを呼ぶ前にタスクをcancelし、用途別の期限超過処理へ進む。空ファイルの開始成功もこの確認を行う。

期限内なら既存の開始成功経路へ進む。アップロードの元のHttpRequest本文の解放は、start成功後に行う既存の契約を維持し、期限内の保存ではstepを進める前に済ませる。開始成功直後に期限超過となった場合はタスクを回収してエラー応答経路へ進む。

start自体が失敗した場合は、復帰時の期限超過にかかわらず既存の開始失敗経路を使う。静的読み取りは500、保存はstartが返したHTTPコードを共通送信準備へ渡し、error_pageは元の応答を保持した内蔵HTMLで確定する。計時失敗は下記の回収/接続終了に従う。メモリ確保失敗時の具体的な回収経路は後続の異常経路設計で定める。

<a id="file-task-poll-timeout"></a>

### ファイルタスクがある間のpoll待ち時間(2026-11-18合意)

進行可能な静的ファイル/error_pageの読み取りタスク、アップロード保存タスク、またはAutoindexTaskがある間は、単一EventLoopのpoll待ち時間を0にする。各周でソケット・CGIのreadinessと期限を確認し、各ファイルタスクを最大1回ずつ進める。読み取りはread最大1回・最大8 KiB、保存はwrite最大1回・最大8 KiBという既存の処理量に従う。

進行可能なファイルタスクがなくなったら、HTTP-70/CGI-111の通常のpoll待ち時間へ戻す。通常時の最大100 msと各期限の確認を維持し、完了/失敗/取得済みのタスクを理由に待ち時間0を続けない。ファイルタスクの進行はEventLoopで管理し、通常ファイルのpoll登録やクライアントソケットのPOLLOUTを駆動条件に使わない。

ファイル処理中もソケット/CGIのI/Oは既存のpoll通知に従う。pollの返却イベントが0件でも、進行可能なファイルタスクの処理と期限確認を行う。ソケット/CGIイベントとファイルタスクの順序は下記に従う。新規タスクのstepは下記の契約に従って次の周から行う。EventLoop全体の期限確認は下記のpoll復帰後・イベント前の契約に従う。

待ち時間0による処理量とCPU使用率を、大きいファイルの配信・保存、小さいGETとCGIの並行処理で測定する。タスク終了後は通常待機に戻り、接続完了後のCPU使用率と資源回収を確認する。

<a id="file-task-event-order"></a>

### poll復帰後のイベントとファイル処理の順序(2026-11-18合意)

pollから戻ったら全体の期限確認を行い、その後に通知されたソケット・CGIイベントを処理する。その後、生存中でファイル処理を継続する接続を対象に、既存のstep前後の期限確認を行って各ファイルタスクを最大1回進める。

イベント処理で切断・終了が確定した接続は、既存の回収経路へ進め、後段のファイル処理対象から外す。通知の種類と切断判定は既存のHTTP-66/CGI-113等に従う。受信EOFによって応答処理を継続する接続などは、その判定結果に従って対象を判断する。

イベント処理には既存のソケット/CGIの処理量制限を適用し、各周でファイルタスクの段階へ進む。イベント0件でも、進行可能なファイルタスクの期限確認と処理を行う。イベント処理中に破棄されたClientを後段から参照しないよう、Client本体は下記の契約で周末に削除し、走査対象の具体的な管理は後続の内部設計で定める。

同じ周のイベント処理や別タスクの結果処理で新たに開始したファイルタスクのstepは、次の周から行う。受信/送信/CGI等を含む全体の期限確認は、下記のpoll復帰後・イベント前に行う。

<a id="file-task-first-step"></a>

### 新規ファイルタスクの最初のstep(2026-11-18合意)

イベント処理または別タスクの結果処理で新たにファイルタスクを開始した場合、最初のstepは次のEventLoop周から行う。開始した周ではstartとその直後の期限確認を行い、成功かつ期限内なら、次の周のファイル処理対象とする。start失敗または開始直後の期限超過は、その場で既存の失敗/超過経路へ進む。

静的読み取りや保存の失敗/期限超過からerror_pageを開始した場合も、error_pageの最初のstepを次の周へ回す。FileReadTaskの同じ値メンバーをcancel後に再利用した場合も、新たな開始として扱う。前の用途のstep実行後にerror_pageのstepを同じ周で連鎖させない。

次のpollは進行可能なファイルタスクがあるため待ち時間0とし、ソケット/CGIイベントを処理してから、継続する接続のタスクを期限確認付きで最大1回進める。次節のClientの今周開始フラグで、開始した周のタスクをstep対象から外す。走査中に終了したClientを再参照しない管理も合わせて具体化する。

<a id="client-deferred-destruction"></a>

### 終了したClientの周末での削除(2026-11-18合意)

接続の終了が確定した時点でClientをCLOSINGにし、進行中のファイルタスクをcancelするなど必要な資源回収を行う。クライアントfdは監視対象から外して閉じ、CGIとの関連付けを解除する。関連するCGIは既存の停止/回収規則に従い、CgiProcessの管理情報を子の回収まで保持する。

Client本体のdeleteと管理コンテナからの除去は、そのEventLoop周の最後にEventLoopがまとめて行う。後段のイベント処理・期限確認・ファイルタスクの走査はCLOSINGのClientを対象から外す。終了が確定した周で、新しい応答生成・send・ファイルタスクのstart/stepを行わない。

同じClientの終了処理が再度呼ばれても、資源の二重close・二重削除を起こさない。保存成功済みのファイルはcancelの既存契約で保護する。終了処理で外したCGIの関連付けを通じて古いClientへ結果を返さない。Clientのデストラクターは既存の再呼び出し安全な資源解放を行い、すでに回収した資源を重ねて処理しない。

現在のpoll結果に残る古い通知は、管理オブジェクトと現fdの有効性を確認する既存の規則で扱う。走査対象を作る具体的な方法と周末の除去手順は、各段階で有効なClientへの参照を維持するよう内部設計で具体化する。

<a id="file-task-clock-failure"></a>

### ファイル処理の計時失敗時の接続終了(2026-11-18合意)

静的ファイル読み取り・error_page読み取り・アップロード保存で、開始時刻の取得や期限確認に必要な時刻の取得/解析に失敗した場合、または値の逆行を検出した場合は、該当ClientをCLOSINGへ進める。進行中のファイルタスクをcancelし、fd・本文・途中ファイルなどを既存契約に従って回収する。開始時刻の取得前に失敗した場合はタスクを開始しない。

クライアントfdの監視解除とclose、関連CGIの関連付け解除/停止/回収を既存の接続終了経路へつなぎ、Client本体を周末に削除する。追加の500応答やerror_page読み取り、内蔵HTML生成を開始しない。error_page読み取り中の計時失敗にも同じ終了処理を使う。

保存とcloseに成功済みのファイルは保護する。完了後の応答生成/送信準備で計時失敗を検出した場合も保存済みファイルを保持し、接続終了へ進む。cancelと破棄を重ねても二重close・二重削除を防ぐ。共通時刻の取得失敗が複数のファイル処理中の接続に影響する場合は、それぞれに本契約を適用する。通常接続とCGI実行中の既存の計時失敗方針はHTTP-70/CGI-124〜126に従う。

<a id="file-task-cloexec"></a>

### ファイルタスクのfd継承防止(2026-11-18合意)

FileReadTask/FileWriteTaskはopen直後にfcntl(fd, F_SETFD, FD_CLOEXEC)を設定する。設定成功までstartの成功を返さず、FileWriteTaskは設定成功後に指示側本文をswapで受け取る。保持情報の準備・open・fd設定までを開始処理の計測対象に含める。

設定失敗時は取得済みfdを所有状態から外して一度だけcloseする。FileWriteTaskが今回新規作成した途中ファイルは既存契約に従って削除し、必要なログ記録後に保持情報を解放する。指示側の本文を維持し、保存開始の失敗コードは500とする。FileReadTaskはfalseを返し、静的読み取りなら500、error_pageなら元の応答の内蔵HTMLへ進む。タスクの通常の開始失敗状態と再利用契約に従う。

CGIのexecve成功時はFD_CLOEXECによりファイルタスクのfdを閉じる。子の起動失敗では既存のEventLoop::cleanupInChild経路で、子に複製されたタスクのfdと保持メモリを解放する。親は所有するタスクで処理を継続する。

子側の解放では、親が保存中の途中ファイルを削除しない。FileWriteTaskの通常のcancel/デストラクターは途中ファイルを削除する契約を持つため、子専用経路で削除責任を外してから破棄する必要がある。専用APIは下記のFileWriteTask::cleanupInChild()を採用する。呼び出し口は下記のClient::cleanupInChild()を採用する。子側では既存のCGI-123に従い、追加確保・ログ出力を行わない。

<a id="file-write-child-cleanup"></a>

### FileWriteTaskのCGI子専用解放API(2026-11-19合意)

```cpp
// FileWriteTaskのpublic関数
void cleanupInChild();
```

CGI子の起動失敗経路から呼び、forkで子に複製された保持fdとメモリを解放する。保持fdは所有状態から外してfd=-1とし、一度だけcloseする。close失敗でも再試行しない。本文・保存先パス・保存名の保持領域を解放し、書き込み位置を0へ戻してPhaseをIDLEとする。

子側の途中ファイル削除責任も解除し、std::removeを呼ばずに完了する。親の保存途中ファイルと保存成功済みファイルを保護する。親側のタスクの状態・保持データ・fdは親の所有管理で継続して扱う。

どのPhaseからも呼べ、繰り返し呼んでも二重closeを起こさない。呼び出し後のcancel/デストラクターではファイル削除を行わず、解放済み資源を重ねて処理しない。子側では追加確保・ログ出力・HTTP応答生成を行わず、例外を外へ出さない。解放処理の具体的な実装とメモリ検査はCGI-123の子専用経路の受入試験で確認する。

EventLoop::cleanupInChildから子に複製された各Clientを破棄する前に、この操作を到達させる。通常のFileWriteTask::cancelやデストラクターによる途中ファイル削除が先に走らないよう、呼び出し順序を管理する。EventLoopからは下記のClient::cleanupInChild()を呼ぶ。

<a id="client-child-cleanup"></a>

### ClientのCGI子専用解放API(2026-11-19合意)

```cpp
// Clientのpublic関数
void cleanupInChild();
```

CGI起動失敗の子専用経路で、EventLoop::cleanupInChildが各Clientの本体を破棄する前に呼ぶ。Clientは所有するFileReadTaskにcancel、FileWriteTaskにcleanupInChildを呼び、複製されたファイルfdとタスクの保持データを解放する。保存タスクの通常のcancel/デストラクターが走る前に、子側の削除責任を外す。

Clientが所有する複製クライアントfdも所有状態から外して一度だけcloseする。送信バッファ・要求・保持応答など、残るClient所有メモリは専用解放と続くClientの破棄で回収する。呼び出し後はCLOSINGとして、EventLoopの子専用経路がClientを破棄する。親の保存途中/成功済みファイルと親の接続・CGIを保護する。

通常の接続終了で行うCGIの停止/回収は親側の既存経路に従う。子専用経路ではkill/waitpid/shutdown・HTTP応答・ログ出力を行わず、追加確保と外へ出る例外を避ける。各資源の所有状態を解除してから解放し、繰り返し呼び出しと続くデストラクターで二重close/削除を防ぐ。

具体的な呼び出し順序は、EventLoopの子専用解放開始、各Client::cleanupInChild、各Clientの破棄、残る管理データの解放という既存の所有関係に沿って組み込む。起動中CGIや一時fdの解放とmainでのEventLoop破棄はCGI-123の契約に従い、子プロセスを対象にしたメモリ検査で確認する。

<a id="client-file-started-this-round"></a>

### Clientの今周開始フラグ(2026-11-19合意)

```cpp
// Clientのprivateメンバー
bool fileTaskStartedThisRound_;
```

EventLoopは各周の先頭で既存Clientのフラグをfalseへ戻す。その周に生成するClientもfalseで初期化する。イベント処理・期限処理・ファイル処理で新たにFileReadTask/FileWriteTask/AutoindexTaskを開始するとき、Clientはフラグをtrueにする。静的読み取り・error_page・アップロード保存・autoindexで共通のフラグを使う。

後段のファイルタスク走査では、CLOSINGと今周開始フラグがtrueのClientをstep対象から外す。開始した周ではstartと直後の期限確認をその場で行い、開始失敗/期限超過は既存の経路へ進む。同じFileReadTaskをcancelしてerror_page用に再開始した場合もtrueを設定し、その周のstepを連鎖させない。

次の周の先頭でfalseへ戻った後、生存中で進行可能なタスクを既存の期限確認付きで進める。フラグはstartを行った周を示し、読み取り/保存の内部Phaseや完了/失敗の状態と区別して管理する。開始した周のフラグがtrueでも、進行可能なファイルタスクがあれば次のpoll待ち時間は0とする。

各周の先頭でのリセットは、その周のイベント処理・期限処理・タスク開始より前に行う。開始後にフラグをリセットして同じ周でstepを許可しない。EventLoopからリセットと対象判定を呼ぶ具体的な操作APIは、既存のClientの責務に沿って内部設計で具体化する。

<a id="event-loop-timeout-order"></a>

### EventLoop全体の期限確認の位置(2026-11-19合意)

各周は先頭で今周開始フラグをリセットし、poll待ち時間を決めて単一pollを呼ぶ。poll復帰後の処理は次の順序とする。

1. 復帰後の現在時刻に基づき、受信・送信・切断待ち・CGI・進行中のファイル処理などの期限を確認する。期限超過は用途別の既存処理へ進める。計時失敗も既存の対象別契約に従う。
2. 通知されたソケット・CGIイベントを処理する。期限処理で無効になったClient/fdやCLOSINGを有効性確認で除外する。
3. 生存中で継続する接続のファイルタスクを進める。今周開始フラグがtrueならstepを省き、対象タスクには既存のstep前後の期限確認と完了優先を適用する。
4. CLOSINGのClientを周末に削除する。CGIの停止/非ブロッキング回収などは既存の所有管理と終了規則に従って継続する。

全体の期限処理で新たにerror_pageなどのタスクを開始した場合は、今周開始フラグをtrueにし、その周のstepを省く。期限処理で閉じたfdの古いpoll通知は、管理オブジェクトと現fdの有効性を確認して処理対象から外す。

ファイルタスクのstart直前/成功直後、step前/IN_PROGRESS復帰後の期限確認も既存の契約に従う。全体の期限確認後に時間のかかるイベント処理があっても、ファイルタスクには呼び出し時点の期限判定を適用する。COMPLETEDは同じstepからの復帰後に期限を超えていても完了経路を使う。

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
| CGI-10 | index | `/cgi/` のようなディレクトリ要求で、設定されたindexとして `index.py` が選ばれた場合も同じCGI実行判定を適用する。2026-11-16合意: CGI有効locationへの許可済みPOSTでも実ディレクトリなら設定indexを探し、CGI対象なら元のPOST本文を渡して実行する。POSTの末尾/補完リダイレクトや別locationへの再ルーティングは行わない。index未指定時にindex.pyを自動補完せず、Configで補完されたindex値を使う。CGI非該当ならHTTP-53の対象別Allowで405、実ディレクトリのindex不在は403とし、autoindexへ進まない |
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
| CGI-93 | 接続情報の受け渡し | ConnectionInfoで接続元IPv4、実際のサーバーIPv4、実際のlistenポートをtasugiyaからrysatoへ渡す。IPv4は4オクテットの十進表記とし、DNSやinet_ntop/inet_ntoaは追加しない。取得に失敗した接続はRouterへ渡さない。[取得失敗時の契約](#getsockname-failure)を参照する |
| CGI-94 | 実行結果の型 | CgiResultはstdout全体とerrorStatusを保持する。0は実行成功、非0はサーバー側のエラーコード。子の終了状態はtasugiya側で解釈し、失敗時の部分出力をHTTPへ送らない。出力の妥当性はrysato側が別途検証する |
| CGI-95 | Routerの受け渡しAPI | 2026-11-15の合意でroute(request, config, connection, response, cgi, file)へ拡張し、2026-11-16に末尾へFileWriteRequest& writeも追加した。2026-11-19にAutoindexRequest& autoindexも末尾に追加した。RESPONSE_READY/CGI_REQUIRED/FILE_READ_REQUIRED/FILE_WRITE_REQUIRED/AUTOINDEX_REQUIREDを返し、それぞれresponse/cgi/file/write/autoindexだけが有効。fileはpath/contentTypeを持つFileReadRequest、writeはpath/filename/bodyを持つFileWriteRequest。CGI実行要求・結果の型とプロセス側APIは維持する。parseCgiOutput(output, response)は成功0・不正出力502を返し、成功時だけresponseを有効とする |
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
| CGI-115 | 古いpoll通知と削除時期 | イベント処理前に管理オブジェクトと現fdの有効性を確認し、閉じたものをスキップする。Clientの削除はCLOSINGとしてその周の最後まで遅らせ、CgiProcessの削除は現在のpoll結果の処理後まで遅らせる。解放済みオブジェクト・再利用fdを古い通知で操作しない |
| CGI-116 | CGIのfd継承 | listen・acceptソケットと全CGIパイプに生成時F_SETFD/FD_CLOEXECを設定し、FileReadTask/FileWriteTaskの通常ファイルfdにもopen直後に設定する。子はstdin/stdoutをdup2で0/1へ接続し元の不要端を閉じ、0/1/2のFD_CLOEXECを解除して標準入出力・stderrを継承する。不要fdはexecve成功時に閉じ、設定失敗は成功扱いにしない |
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
| CGI-133 | 内部の共通処理 | 2026-11-15更新: makeError(code)はコード/標準説明文の元応答だけを作り、設定参照/本文生成/読み取りを行わない。生成したエラーと正常なCGIの検証済み応答を共通のprepareErrorPageへ渡し、読み取りを行う経路はfinishErrorPageで本文を確定する。prepareErrorPageをClient側の共通送信準備で1回だけ呼び、finishErrorPage後は再準備せず送信する。旧applyErrorPage/readErrorPageは廃止し、ファイル読み取りをHttpResponseから外す。共通の入口はClient::prepareResponse(const HttpResponse&)とする。本文置換動作と一度だけの適用は維持し、CGI専用の重複実装を作らない |
| CGI-134 | 応答を返せない場合 | クライアント切断時は停止・回収だけを行い、応答を生成しない。送信失敗後に別のエラー応答を追加送信しない。listen側のCLOEXEC設定失敗は起動失敗、acceptしたfdの設定失敗はその接続を閉じて監視へ登録しない。CGIパイプの設定失敗は上記の親500/子502の分類を使う。 |
| CGI-135 | 結合の完了条件 | 04のCGI受入項目を実装後に満たすことを条件とする。正常なGET/POST・chunked入力、入出力の並行処理、出力検証・エラーコード・上限・期限、切断/半閉鎖/SIGINT、fd継承・子の回収・寿命、CGI実行中の静的配信を確認する。加えて正常なCGIの404/599の置換、200の非置換、エラーページ未指定/読取失敗/空ファイル、元本文の符号化等の除去とAllow等の保持、不正な404出力が502になることを確認する。9/10の代替案は実環境で検証し、必要なら方法を見直す。10項目の合意完了と、未実装の結合試験合格は別に扱う。 |

詳しい規則の説明は [03 CGI](reports/03_config_cgi_upload.md#cgi-details)、実装後の確認項目は [04 検証計画](reports/04_testing_and_workplan.md) を参照する。

## 合意状況と次の設計判断

- ConfigはC-01〜42まで合意済み。詳細・設定例・公開APIは [05](reports/05_config_agreement.md#config-decisions) にまとめる。
- CGIはCGI-01〜135まで合意済み。CGI-120〜127の起動・計時方法は現時点の採用案で、実装・実際の動作で見直し得る。子の失敗時の明示的な資源解放と、親用の停止・回収処理を実行しない方針は維持する。
- HTTPの1/10〜6/10はHTTP-01〜44で合意済み。7/10はstd::removeによる削除を含めHTTP-45〜50で合意済み。8/10の応答生成はHTTP-51〜58で合意済み(Date省略は暫定案)。9/10のHTTP API/データの保持はHTTP-59〜64で合意済み。10/10のネットワーク境界・計時・通常ファイルI/OはHTTP-65〜74で合意済み(採用案を含む)。10項目の動作方針の判断は完了し、削除手段も確定した。Dateは暫定、接続情報取得失敗時は新規接続だけを閉じる方針を採用済み、採用案の実環境検証は未実施。残件は [04の10項目](reports/04_testing_and_workplan.md#remaining-design) で管理する。

ここでの「合意済み」は設計判断の状態を示す。実装・受入試験の完了とは区別する。

## Config合意事項

Configの決定表C-01〜42、詳細規則、設定例、型・API・寿命の正本は [05 Configの設計と合意事項](reports/05_config_agreement.md#config-decisions) とする。本書には同じ本文を再掲しない。

root/aliasの区別、設定ファイルを基準とした相対パス、限定文法、既定値、起動時検証は合意済み。error_pageは100〜599の設定を受理し、本文置換は400〜599だけに適用する。HTTPの未決定事項をConfigParserの処理として追加しない。

---

# クラス設計(全13クラス+2表)

Configの型・parse API・getter・所有関係はC-42で確定済み。HTTPの責務・公開API・保持はHTTP-59〜64で合意済み。I/O・資源管理はHTTP-65〜74に従い、検証後に見直し得る採用案と未確定の内部実装を区別する。CGI受け渡しの公開型・シグネチャ・担当境界はCGI-91〜98(2026-11-15のroute拡張を含む)、所有権・寿命はCGI-99〜106、後始末はCGI-107〜118で確定済み。起動失敗検出・計時はCGI-120〜127を現時点の代替案として採用し、実装・動作確認で見直し得る。モジュール一覧の `CgiHandler` はCGI処理の総称で、この案では `CgiExecutor` / `CgiProcess` 等に分ける。

## 一覧と責務

### 設定ドメイン(rysato担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ConfigParser` | ファイル→トークン化→構文解析→既定値補完/パス解決/検証。Configを値で返し、設定エラーはruntime_errorをmainへ伝える(C-42) | 起動時1回・使い捨て |
| `Config` | パース結果の最上位入れ物 | 起動時1回・終了まで不変 |
| `ServerConfig` | server ブロック1個分のデータ | 同上 |
| `LocationConfig` | location ブロック1個分のデータ | 同上 |

※ Config / ServerConfig / LocationConfigはprivateメンバーとconst getterを持つ読み取り専用データクラス。設定値の解析・検証はConfigParserが行う(C-42)。

構築方法は2026-10-07にConfigParserをfriendとする案を暫定採用し、コンストラクターの初期値とConfigParserによる既定値補完を分ける方針、[ConfigParserの内部構成](reports/05_config_agreement.md#config-parser-design)も採用した。具体化の正本は [05の構築方法](reports/05_config_agreement.md#config-construction) とする。2026-10-08に[検証済みPWDによる絶対パス取得](reports/05_config_agreement.md#config-pwd-resolution)を仮決定した。相対指定・引数なし起動には現在ディレクトリとの一致をstatで確認できるPWDを要求し、使えない場合は絶対パス指定を求めて起動エラーとする。絶対パス指定はPWDに依存しない。実環境で検証後に見直し得る。[その他の補助関数・ファイル分割](reports/05_config_agreement.md#config-parser-helpers)は提案段階。

### ネットワーク/イベントドメイン(tasugiya担当)

| クラス | 責務 | 生成タイミング |
|---|---|---|
| `ServerSocket` | socket→setsockopt→bind→listen→ノンブロッキング化(listen先のhost:port 1組分) | 起動時・listen先数分 |
| `EventLoop` | poll 本体。fd 登録簿、イベント配送、タイムアウト見回り、終了時の後始末 | 起動時1個 |
| `Client` | 接続1本の状態機械(バッファ・状態・各段階の開始時刻、HTTP-65〜70) | accept 時に生成 / close 時に破棄 |
| `FileReadTask` | Clientの値メンバー。fd・途中の本文・状態を保持し、各周最大1回・8 KiBのreadと後始末を担当する。FileReadTask/Clientはコピー禁止。詳細APIは未決定 | Clientと一緒に生成・破棄。未開始時はfd=-1・空本文 |
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
     │   └─ FileReadTask fileTask_(値メンバー。通常ファイルfd・途中の本文・状態)
     │   └─ HttpResponse pendingResponse_(値メンバー。error_page読み取り中も元応答を保持)
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
    void   releaseBody(); // COMPLETE後にClientが本文の保持領域を解放
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

- 外部の状態はHTTP-60の3つ、内部の要求行/ヘッダー/本文/chunked解析段階は非公開。生成時の本文上限を変更しない。COMPLETE/ERROR後のappendDataは結果を維持する(HTTP-61)。COMPLETE後のreleaseBodyは本文の保持領域を解放する
- エラー判定(400/413/417/501等)はパース時点でHttpRequest自身が行い、ソケットI/Oや応答生成は行わない
- ヘッダー完了時に受信バージョン別のHost・Transfer-Encoding・Expectを検証する。拒否する場合はボディを待たずERRORへ遷移し、Clientがエラー応答を送る
- ClientはappendDataのたびに状態を確認する。ERRORになった要求のボディ受信を続けてCOMPLETEを待たない
- ヘッダーキーは内部で小文字化して格納。getHeadersは全行を保持し、単一値getterのために重複を上書きして失わない(CGI-97)
- chunked デコードはこのクラスに閉じる。HTTP/1.1〜1.9要求で受理し、1.0では拒否する(HTTP-04・08)。CGIには復号済みボディを渡す

<a id="http-request-internals"></a>

#### HttpRequestの内部状態と保持データ（2026-10-10採用）

2026-10-08に提案した内部状態・保持データ・解析関数の分担を、2026-10-10にrysatoの了承により採用した。HTTP-59〜62の公開APIと既存のHTTP動作は維持する。実装・動作試験の完了は示さない。HttpRequestは独立した1クラスとし、別のヘッダーパーサー/チャンクデコーダークラス、Configへの参照、friend、継承を追加しない。

**外部のStateと、内部の解析段階Phaseを分ける。** 外部は引き続きINCOMPLETE / COMPLETE / ERRORの3つ。state_がINCOMPLETEの間だけphase_を使用し、完了・エラー用のPhaseは重ねて作らない。

| privateなPhase | 読むもの | 正常時の次の段階 |
|---|---|---|
| REQUEST_LINE | 要求行1行 | HEADERS |
| HEADERS | ヘッダー各行と最後の空行 | 検証後、本文なし/CL=0ならCOMPLETE、正のCLならFIXED_BODY、chunkedならCHUNK_SIZE |
| FIXED_BODY | Content-Lengthで指定された本文 | 残り0でCOMPLETE |
| CHUNK_SIZE | サイズとextensionの1行 | 正のサイズならCHUNK_DATA、0ならTRAILERS |
| CHUNK_DATA | 指定されたバイト数の本文 | 残り0でCHUNK_CRLF |
| CHUNK_CRLF | チャンク本文直後のCRLF | CHUNK_SIZE |
| TRAILERS | trailer各行と最後の空行 | 共通構文を検証して各行を捨て、空行でCOMPLETE |

0サイズの行の後は直接TRAILERSへ進む。通常チャンク用の本文/CRLFを余分に要求しない。エラーなら、どの段階からでも最初の確定コードを記録してERRORへ進む。EOF・期限・接続の破棄は既存どおりClient/EventLoopが担当し、Phaseに追加しない。

保持データは次の構成とする。既存のgetterは完成後の値を参照し、要求行の原文や符号化済みの本文全体を別途保存しない。

```cpp
// 採用したHttpRequestのprivateデータ構成。公開宣言は上記を維持する。
enum Phase {
    REQUEST_LINE, HEADERS, FIXED_BODY,
    CHUNK_SIZE, CHUNK_DATA, CHUNK_CRLF, TRAILERS
};

State state_;
Phase phase_;
int errorCode_;
std::size_t maxBodySize_;

std::string method_;
std::string path_;
std::string queryString_;
std::string version_;
std::vector<std::pair<std::string, std::string> > headers_;
std::string body_;

std::string line_;                 // 現在の1行。CRLFは格納しない
bool pendingCR_;                  // 行末のCRを受け取り、LF待ち
std::size_t lineBytes_;            // 現在の1行の受信量。CRLF込み
std::size_t headerBytes_;          // 初期ヘッダー合計。終端空行込み
std::size_t trailerBytes_;         // trailer合計。終端空行込み
std::size_t chunkExtensionBytes_;  // 完了済みサイズ行のextension合計
std::size_t extensionStart_;       // 現在のサイズ行のextension開始位置
std::size_t remainingBodyBytes_;   // CL本文、または現在のチャンクの残量
std::size_t chunkCrlfBytes_;       // 本文直後のCRLFの消費数(0または1)
```

コンストラクターはstate_=INCOMPLETE、phase_=REQUEST_LINE、errorCode_=0、maxBodySize_=引数とする。文字列・vectorは空、pendingCR_=false、カウンターは0、extensionStart_だけstd::string::nposとする。maxBodySize_は以後変更しない。Configの検証済みの正の本文上限を渡す既存契約を維持する。

remainingBodyBytes_はPhaseに応じて意味が決まるため、CL用とチャンク用に2つ持たない。復号済みの累積本文量はbody_.size()から取得し、別の累積カウンターを増やさない。headers_は到着順・名前小文字・前後SP/HTAB除去済みの全行を保持する既存のvectorだけを持ち、同じ値を保存するmapは追加しない。Host/CL/TE等の重複・意味検査に必要な件数や参照はヘッダー検証中のローカル変数で求める。

**appendDataは受け取った範囲をその呼び出し中に順に消費する。** dataへのポインターは保存せず、lenで範囲を管理する。len=0は何もしない。len>0の場合はClientが有効な受信範囲を渡す。COMPLETE/ERRORになった時点で処理を止め、同じ断片の残りも保存しない。受信断片全体を保持するrawBuffer_は追加しない。

| private関数の分担 | 処理 |
|---|---|
| consumeLineByte(char) | REQUEST_LINE/HEADERS/CHUNK_SIZE/TRAILERS用。1バイトごとに既存の上限とCRLFを検査し、line_へ格納する。行が完成したかをboolで返し、エラーはstate_で区別する |
| parseRequestLine() | 完成したline_の書式・メソッド・版・要求先を検査する。queryを分離し、pathを1回デコード/正規化して保持する |
| parseHeaderLine() | 完成したline_の共通構文を検査し、名前と値をheaders_へ格納する。trailer用の共通構文検査は補助関数として再利用し、trailerをheaders_へ入れない |
| finishHeaders() | 重複・Host・CL/TE・Connection・Expect等の既存規則を検査し、本文の読み方と残量を決める。既存の400/413等に優先する形で417を早決めしない |
| parseChunkSizeLine() | サイズとextensionを検査し、残量を決める。正常なサイズでもmaxBodySize_ - body_.size()を超えれば本文を待たず413 |
| consumeBody(const char*, size_t) | 入力長とremainingBodyBytes_の小さい方だけbody_へappendし、消費数を返す。NULを含む本文を長さ付きで扱い、残り0なら対応する次段階へ進む |
| consumeChunkCrlfByte(char) | 本文直後のCRとLFを順に検査する。断片境界ではchunkCrlfBytes_を保持する |
| fail(int) | INCOMPLETEのときだけerrorCode_とERRORを設定する。最初のエラーを後から上書きしない |

完成した行は対応する解析関数を呼んでからline_・lineBytes_・pendingCR_を次行用に戻す。headerBytes_・trailerBytes_・chunkExtensionBytes_は要求内の合計なので行ごとに戻さない。extensionStart_はサイズ行ごとに未検出へ戻す。

上限は完成行を待たずに検査する。要求行8 KiB/414、初期ヘッダー合計32 KiB/431、サイズ行8 KiB/400、extension合計32 KiB/400、trailer合計32 KiB/431は既存どおり。extensionStart_はサイズ行の先頭の16進数字列が終わった位置とし、CRLFを除く後続部分を数える。受信中にも「完了済み合計＋現在行のextension量」を検査してから蓄積する。上限判定では加算前の残量比較を使い、カウンターを桁あふれさせない。初期ヘッダー各行の追加の独自上限は設けない。

本文はまとめてコピーし、行部分だけを1バイトずつ処理する。断片を受け取るたびに行頭から再走査したり、巨大な受信バッファの先頭を何度もeraseしたりする構成にはしない。各反復は入力を消費するか、Phase/Stateを進める。メモリ確保失敗は既存の接続処理側の例外捕捉・資源解放に委ね、通常の不正HTTP入力はfailによる状態/コードで返す。

今回採用した範囲は状態・保持データ・関数の分担。単一ヘッダーgetterの細則とヘッダー検証順序は次節で採用済み。private関数の正式な宣言は後続の具体化案と区別する。getHeadersの全行保持、既存のエラー規則、CGI側の専用検証は変更しない。

<a id="http-request-header-design"></a>

#### HttpRequestのヘッダー取得と検証順序（2026-10-10採用）

2026-10-10、rysatoの了承により以下のgetterの細則とfinishHeaders内の順序を採用した。HTTP-06〜11・57・61・62、C-28〜35、CGI-76・79・80を具体的な呼び出し順へ落としたもの。公開シグネチャとheaders_による全行保持は変更しない。実装・動作試験の完了は示さない。

**getHeader(key)は、ASCIIの大小を無視して最初に一致した行の値をコピーして返し、欠落時は空文字列を返す。** keyにはヘッダー名だけを渡し、前後の空白や末尾のコロンを補正しない。値を結合・再検証せず、state_やerrorCode_も変更しない。重複拒否をgetterの呼び出し有無に依存させない。

| 保存されたヘッダー | 呼び出し | 戻り値 |
|---|---|---|
| `X-Test: one` | getHeader("x-TEST") | `"one"` |
| X-Testなし | getHeader("X-Test") | `""` |
| `X-Test:` | getHeader("X-Test") | `""` |
| `X-Test: one`と`X-Test: two` | getHeader("X-Test") | `"one"`。getHeadersには両行が残る |

欠落と空値、重複の有無、全行の値が必要な処理はgetHeadersを使う。特にCGIのCONTENT_TYPEは欠落なら省略、空値なら空の変数という既存の区別を維持するため、getHeaderの空判定では決めない。Connectionの全行処理とCGI転送対象の重複検査もgetHeadersを使う。Host等の単一行制約はHttpRequestの検証で強制され、getHeaderの「最初の行を返す」という便宜的な動作で重複を受理しない。公開hasHeader/getHeaderCountや検索用mapは追加しない。

**受信時の共通構文検査と、ヘッダー終端後の意味検査を分ける。** consumeLineByteでCRLF・受信量の上限を検査し、完成した非空行をparseHeaderLineへ渡す。parseHeaderLineは名前のtoken・コロン直前空白・obs-fold・値の制御文字等を検査し、正常なら名前をASCII小文字、値を前後SP/HTAB除去済みにしてheaders_へ追加する。ここで確定した400/431等は直ちにERRORとし、終端を待たない。空行はヘッダーフィールドとして保存せずfinishHeadersを呼ぶ。

finishHeadersはheaders_を走査し、対象行へのポインター・件数・Expectの存在をローカル変数で集める。ポインターは関数内だけで使い、走査中・検証中にheaders_を変更しない。空値でも行の存在として数え、getHeaderを存在/重複判定に使わない。判定は次の順とする。

| 順序 | 検証・処理 | 失敗時 |
|---|---|---|
| 1 | Host / Content-Length / Content-Type / Transfer-Encodingの各最大1行、HTTP/1.1〜1.9のHost必須を確認する | 400 |
| 2 | 存在するHostをC-30の範囲で検証する。HTTP/1.0も空値・不正値を許可しない | 400 |
| 3 | CLとTEの併記、HTTP/1.0のTE指定を拒否する。CLの十進構文・保持型の範囲、TEの一覧構文・chunkedの回数/パラメーター・最後がchunkedであることを検査する | 不正は400。正常な未対応の符号化の存在はローカルに記録し、順序6で判定 |
| 4 | Connectionの全行をHTTP-10のtoken一覧として検証する | 400 |
| 5 | 正常なCLの値をmaxBodySize_と比較する。桁あふれを413として扱わない | 上限超過413 |
| 6 | TEの構文は正常だが未対応の符号化を含む場合を拒否する(例: gzip, chunked) | 501 |
| 7 | HTTP/1.1〜1.9でExpectが1行以上あれば、値・空値・重複によらず拒否する。HTTP/1.0では無視する | 417 |
| 8 | 検証成功時だけ本文の段階を決める。TEならCHUNK_SIZE、正のCLならFIXED_BODY、CL=0または両方なしならCOMPLETE | — |

順序3で未対応のTEを記録してもERRORはまだ確定させず、Connectionの不正を検査してから501を確定する。これはこの検証関数内の順序であり、全処理に適用するHTTPコードの優先順位表は作らない(HTTP-57)。先に要求行や共通構文・受信上限で確定したエラーはそのまま保持する。finishHeadersの判定はヘッダー行の到着順やrecvの分割位置で変えない。

Expectを無視するHTTP/1.0でも共通構文検査は省略しない。HTTP/1.1〜1.9のExpectはヘッダー終端で417となるため、同じrecv断片にボディが続いていても解析へ進まない。後続のチャンクの不正や本文超過を調べるために417を保留せず、既に確定した結果も後続データで変更しない。100 Continueは生成しない。

Content-Typeのメディアタイプ/パラメーター、multipart、CGI転送対象の名前・重複等は既存の処理先で検証する。HttpRequestはlocationを知らず、通常要求をCGI専用条件で拒否しない。trailerは共通構文だけ検査して捨て、finishHeadersへ再投入しない。

以下は採用した動作を実装で確認する例。試験は未実施。

| 入力条件(特記以外の構文は正常) | 期待する結果 |
|---|---|
| HTTP/1.1、Hostなし、Expectあり | 400 |
| HTTP/1.1、正しいHost、CLが本文上限を超過、Expectあり | 413 |
| HTTP/1.1、正しいHost、CLとTE併記、Expectあり | 400 |
| HTTP/1.1、正しいHost、TEはgzip, chunked、不正なConnection値 | 400 |
| HTTP/1.1、正しいHost、TEはgzip, chunked、Expectあり | 501 |
| HTTP/1.1、正しいHost、空のExpect、CL=0 | 417 |
| HTTP/1.0、共通構文が正常なExpect、CL=0、Hostなし | COMPLETE。Expectを無視 |
| 通常配信の要求でX-Testが2行 | この重複だけでは拒否しない。CGIへ進む場合は別途既存の転送規則を適用 |

取得方法と検証順序は採用済み。以下ではprivate関数の正式な宣言・引数・戻り値と、HttpRequest.hpp/.cppの境界を具体化する。

<a id="http-request-private-api"></a>

#### HttpRequestのprivate宣言とファイル分割（2026-10-10暫定採用）

2026-10-10、rysatoの「一旦Ok」を受け、以下のprivate宣言・引数/戻り値の契約・ファイル分割・暗黙のコピー/破棄を暫定採用した。採用済みのメンバーと公開APIに組み合わせ、実装時に必要があれば見直す。**解析を進める関数は状態を更新し、単独の値を検査する関数は結果だけを返す。** 通常の不正入力に例外を使わず、状態を更新する側がfail(code)で拒否を確定する。

```cpp
// HttpRequestのprivate部分に追加する宣言案
bool consumeLineByte(char byte);
void parseRequestLine();
void parseHeaderLine();
void finishHeaders();
void parseChunkSizeLine();
void parseTrailerLine();
std::size_t consumeBody(const char* data, std::size_t len);
void consumeChunkCrlfByte(char byte);
void resetLine();
void fail(int code);

// 状態を変更しない検査・変換。すべてprivate static。
static bool parseHeaderFields(const std::string& line,
    std::string& name, std::string& value);
static bool parseRequestTarget(const std::string& target,
    std::string& path, std::string& query);
static bool parseContentLength(const std::string& value,
    std::size_t& length);
static bool isValidHost(const std::string& value);
static bool isValidConnection(const std::string& value);
static int checkTransferEncoding(const std::string& value);
static bool isValidChunkExtensions(const std::string& line,
    std::size_t start);
```

前半の関数はstate_/phase_や保持データを更新するためconstを付けない。後半はHttpRequestのインスタンスデータを使わないのでstaticとする。外部から呼べる共通Utilクラスは作らない。文字のtoken判定・ASCII大小変換・OWS除去等の小さな共通処理は、実装時に必要な範囲で同クラスのprivate補助関数として分ける。ConfigParserをHTTPの数値解析に使い回さない。

戻り値と呼び出し条件は次のように揃える。

| 関数 | 呼び出し側が守る条件・結果の意味 |
|---|---|
| consumeLineByte | 行を読むPhaseで、現在の1バイトを消費する。trueはCRLFまで揃い正常に1行完成、falseは続き待ちまたはERROR。呼び出し側はまずstate_を確認し、ERRORなら直ちに止める |
| parseRequestLine / parseHeaderLine / parseChunkSizeLine / parseTrailerLine | CRLFを除いた完成行がline_にある状態で呼ぶ。成功時の格納/遷移は各関数内で行う。異常ならfailして直ちに戻る。parseHeaderLineは非空行専用、ヘッダーの空行はappendDataがfinishHeadersへ振り分ける |
| parseTrailerLine | 空行ならCOMPLETE。非空ならparseHeaderFieldsで共通構文を検証し、結果はローカル変数のまま捨てる。Host等の意味検査やheaders_への追加をしない |
| finishHeaders | 初期ヘッダー終端で1回だけ呼ぶ。採用済みの順序で検査し、成功時だけ本文PhaseまたはCOMPLETEへ進む |
| consumeBody | FIXED_BODY/CHUNK_DATAで、len>0かつremainingBodyBytes_>0のとき呼ぶ。入力先頭から格納したバイト数を返し、appendDataがその分だけ入力位置を進める。通常復帰時は必ず1バイト以上消費する |
| consumeChunkCrlfByte | CHUNK_CRLFで1バイト消費し、CRの次はLFを要求する。不正なら400。LFまで揃ったらchunkCrlfBytes_を0に戻しCHUNK_SIZEへ進む |
| resetLine | 正常に処理した行の後、次の行に備えてline_をclear、lineBytes_=0、pendingCR_=false、extensionStart_=nposにする。要求全体のカウンター・結果・本文は消さない。公開reset/reuse APIではない |
| fail | INCOMPLETEの場合だけ最初のエラーコードを記録する。呼び出した関数はその直後に戻り、appendDataも止める。応答生成やソケット操作はしない |
| boolの検査関数 | trueは成功、falseは不正。出力引数を持つ関数の出力はtrueの場合だけ使う。falseを呼び出し側が400に変換する。メモリ確保失敗は通常の不正入力に変換せず、既存の接続側の例外処理へ伝播する |
| checkTransferEncoding | 0は受理できる単独chunked、400は構文/終端等が不正、501は構文は正常だが未対応の符号化を含む。状態は変更せず、finishHeadersが既決定の位置でfailする |

parseHeaderFieldsは名前をASCII小文字、値を前後SP/HTAB除去済みで返す。空行は呼び出し側で扱い、この関数には非空行を渡す。parseRequestTargetはorigin-formの検証からquery分離・パスの1回デコード/正規化まで行い、queryは未デコードで返す。parseRequestLineがメソッド/版の検証と呼び出し順を管理する。

parseContentLengthは空・非十進・size_tの桁あふれをfalseとし、0と先頭ゼロは受理する。本文上限との比較はfinishHeadersに残す。isValidHostはC-30、isValidConnectionはHTTP-10の1行分を検査する。isValidChunkExtensionsのstartはサイズ行の先頭16進数字列の直後を指し、extensionなしならline.size()とする。CRLFを含まない残り全部をHTTP-15の文法で検査する。サイズ数値の変換・本文上限・extension受信量の制限は既存の担当箇所に残す。

appendDataの実装では、行用Phaseは1バイトずつ、本文用PhaseはconsumeBodyが返した分だけ進める。行完成時に現在のPhaseで解析関数を選び、正常かつINCOMPLETEならresetLineする。終端状態ではその場で戻る。CHUNK_DATAからCHUNK_CRLFへ入るときはchunkCrlfBytes_=0とする。正常な0サイズ行は直接TRAILERSへ進む。

ファイルはHttpRequest.hppとHttpRequest.cppの2つに分ける案。hppにはinclude guard、公開宣言、privateのPhase・メンバー・関数宣言を置き、直接必要な標準ヘッダー`<cstddef>`・`<string>`・`<utility>`・`<vector>`をincludeする。Config/Router/HttpResponse/ClientやOSのヘッダーはincludeしない。cppは自分のhppをincludeし、コンストラクター・getterを含む関数定義、実装に必要な標準ヘッダー、既存上限定数を置く。小さなgetterもまずcppへ揃え、inline化は必須にしない。

専用のコピーコンストラクター・コピー代入・デストラクターは追加せず、標準の値型メンバーに対する暗黙生成を使う案。Clientは生成時に本文上限を渡して直接メンバーを構築し、通常の処理では要求をコピー/代入せずconst参照で渡す。コピー可能であることを要求再利用や途中の上限変更に使わず、受信中のdataやヘッダーへの一時ポインターをメンバーに保持しない。

この宣言は既存の公開APIと内部方針を具体化したもので、実装ファイルは未作成。暫定採用によりHttpRequestのクラス設計を一区切りとし、HttpResponseの保持データと組み立て責務へ進む。実装時の細かな関数分割は、合意済み動作を変えない範囲で調整する。

### HttpResponse

2026-11-15追記: 元応答をClientが保持し、prepareErrorPageで読み取り要否を判断し、finishErrorPageで結果から本文を確定する方針・APIを合意した。正本は[error_pageの準備とClientへの受け渡し](#error-page-prepare)。makeErrorは設定を受け取らない生成専用APIへ変更済み。旧applyErrorPage/readErrorPageは廃止し、HttpResponseからファイル読み取りを外した。Client側の共通送信準備への集約は合意済みで、共通の入口はClient::prepareResponse(const HttpResponse&)とする。[通常ファイル読み取りの設計修正](#file-read-design)に従い、下記をそのまま実装可能な最終APIとは扱わない。ステータス・ヘッダー・本文置換の既存動作は維持する。

```cpp
class HttpResponse {
public:
    HttpResponse();
    void reset();               // 保持領域を解放し200/OKへ戻す。送信バッファには触れない
    void setStatus(int code);   // reason phrase は内部表で自動解決
    void setStatus(int code, const std::string& reason); // CGI説明文を空も保持(CGI-98)
    void setHeader(const std::string& key, const std::string& value);
    void setBody(const std::string& body, const std::string& contentType);
                                // 通常の本文付き応答のContent-Length / Content-Typeを設定
    void setBodyBySwap(std::string& body, const std::string& contentType);
    std::string serializeHeaders() const;
    const std::string& getBody() const; // 送信対象の本文を参照
                                // HTTP-51〜58: HTTP/1.0・Connection: close。Serverを生成せずDate生成も暫定的に省く
                                // CGIの204/304は本文・Content-Lengthなし、205は本文なし・長さ0(CGI-73)
    bool prepareErrorPage(const ServerConfig& config, FileReadRequest& file);
                                // true: fileが有効で読み取りが必要。openしない
    void finishErrorPage(bool readSucceeded, const std::string& body);
                                // 成功ならbody、失敗なら元コードの内蔵HTMLで確定
    static HttpResponse makeError(int code);
    static HttpResponse makeUploadSuccess(const std::string& filename);
                                // コード・標準説明文のみ。本文は共通の準備/確定経路で決める
};
```

- `\r\n` の組み立ては serialize() に封じ込める
- Transfer-Encodingは出力しない。本文を送れないステータスはserialize時にも検証し、setStatus/setBodyの呼び出し順にかかわらず正しい形式にする
- 送信完了後はClientをCLOSINGへ遷移させる。次の要求を受けるための再初期化は行わない
- サーバー側エラーの元応答生成はmakeError(code)に一元化。HttpRequestはコードだけを返し、Router/Client/EventLoop側が応答を作る。makeErrorは設定参照/本文生成/ファイル読み取りを行わず、生成後に共通のprepareErrorPage経路へ渡す。正常なCGI応答は検証済みの元応答を同じ経路へ渡し、makeErrorで作り直さない。読み取りを行った場合はfinishErrorPageで確定し、元コード・CGI説明文・必要ヘッダーを保持する。prepareErrorPageはClient側の共通送信準備で1回呼び、finishErrorPage後は再準備せず送信する。共通の入口はClient::prepareResponse(const HttpResponse&)とする
- serialize()は送信開始前に1回だけ呼び、生成文字列をClientのsendBufferへswapで渡す。Clientは送信済みオフセットを保持し、元のHttpResponseの寿命に依存しない(HTTP-64)
- Routerの即時応答/CGI要求の戻り値と非同期結果の受け渡しはCGI-91〜98で確定済み。所有権・寿命はCGI-99〜106で確定済み。後始末はCGI-107〜118で確定済み。起動失敗検出・計時は03の9/10の現時点の代替案(CGI-120〜127)を採用し、動作確認で見直し得る。

<a id="http-response-internals"></a>

#### HttpResponseの保持データと関数の責務（2026-10-10採用）

2026-10-10、rysatoの了承により以下の保持データと責務分担を採用した。HTTP-51〜58・64、CGI-98・130〜133の公開APIと動作を維持する。実装・動作試験の完了は示さない。状態機械・Builder・friend・継承は追加せず、**ステータスコード・説明文・ヘッダー・本文の4つを値で持つ。**

```cpp
// 採用したHttpResponseのprivateデータ構成
int statusCode_;
std::string reasonPhrase_;
std::map<std::string, std::string> headers_;
std::string body_;
```

コンストラクターはstatusCode_=200、reasonPhrase_="OK"、headers_とbody_は空にする。処理結果を組み立てる側は必要なstatus/header/bodyを設定し、Router等が正常に応答を完成させてからserializeする。HTTPバージョンは既存どおり1.0固定なのでメンバーに持たない。Content-Length専用の数値メンバー、本文サイズの別カウンター、送信オフセット、Configへの参照は追加しない。送信オフセットはClientが持つ。

**headers_のキーはASCII小文字に統一し、同名は1個にする。** setHeaderは既存の同名値を置き換える。HttpRequestでは全行保持が必要なのでvectorを使うが、応答はCGIも含め同名最大1行という既存の制限(CGI-69)に合わせてmapを使える。setHeaderの上書きはサーバー内部の応答組み立て用であり、CGIの不正な重複出力を受理する手段にはしない。Router::parseCgiOutputは空値省略・転送除外・setHeaderより前に元の全行の重複を検査し、重複なら既存どおり502を返す。

応答のヘッダー名は小文字のまま出力し、元の大小文字や挿入順を別途保存しない。値は検証済みの内容を保持し、一律に小文字化・カンマ結合しない。複数Set-Cookieを受理するための特例も追加しない(CGI-69)。

| 公開関数 | この構成での責務 |
|---|---|
| reset() | 本文・ヘッダー等の保持領域を解放し、コード/説明文を200/OKへ戻す。Clientの送信バッファ・オフセット・状態・期限は変更しない |
| setStatus(int code) | コードとステータス表の標準説明文を設定する。前の説明文を引き継がない |
| setStatus(int code, const std::string& reason) | コードと渡された説明文をそのまま設定する。空文字列でも標準説明文で補完しない |
| setHeader(key, value) | 名前を小文字に揃えてmapへ格納/置換する。CGIの元出力の構文・重複検査は担当しない |
| setBody(body, contentType) | 本文を値で保持し、Content-Typeとその時点の本文サイズに対応するContent-Lengthを設定する。最終的な本文/CLの出力可否はserializeで確定する |
| prepareErrorPage(config, file) | 200〜399は不変でfalse、400〜599の未指定は内蔵HTMLで確定してfalse、指定ありは読み取り指示を設定してtrue。open/readは行わない |
| finishErrorPage(readSucceeded, body) | 成功時は空も含むbody、失敗時は元コードの内蔵HTMLを採用。コードとreasonPhrase_を変えず、本文関連ヘッダーを除去してtext/htmlと実長を設定する |
| makeError(code) | ローカルなHttpResponseへsetStatus(code)を適用して値で返す。設定参照/本文生成/読み取りは行わず、後続の共通error_page準備経路へ渡す |
| serialize() const | 最終コード・本文に従ってHTTP/1.0のバイト列を作り、値で返す。メンバーを書き換えず、ファイル読み取り・エラーページ適用・送信を行わない |

setStatusの2つのオーバーロードは呼び出し時にreasonPhrase_を確定するため、hasCustomReason等のフラグは不要。serializeは空のreasonPhrase_を標準文言に戻さない。例えばsetStatus(404, "")の後にprepareErrorPage/finishErrorPageで本文を確定しても説明文は空のままとし、ステータス行は`HTTP/1.0 404 \r\n`になる。setStatus(404)を呼び直した場合は標準説明文へ戻る。

**送信するContent-LengthとConnectionはserializeが最終決定する。** mapに保存されたcontent-length・connectionはそのまま出力せず、実際に出力する本文からContent-Lengthを生成し、Connection: closeを1行だけ生成する。transfer-encodingは出力しない。setBodyの後にsetStatus(204)が呼ばれたり、Content-Lengthが別値で設定されても、HTTP-51/52の送信形式を維持する。

| 最終コード | serializeが出力する本文 | Content-Length |
|---|---|---|
| 204 / 304 | なし | 省略 |
| 205 | なし | 0 |
| その他の本サーバーで扱う最終応答(200〜599) | body_の全バイト | body_.size() |

これはCGIの不正出力を補正して受理する意味ではない。CGIが204/205/304で本文を出した場合や、宣言CLと本文の実長が違う場合は、HttpResponseへ完成結果を渡す前に502とする。1xxだけのCGI出力も既存どおり502で、中間応答を生成する経路は作らない。

ステータス/長さ/接続用以外のヘッダーは既存の転送・本文置換規則に従う。serializeでServer/DateやContent-Typeを推測して追加せず、通常のContent-TypeはsetBody等で設定済みの値を使う。CGIから転送が認められたServer/Dateは保持する。エラーページ置換時はContent-*・ETag・Last-Modified・Digest・Repr-Digestを除去する一方、Allow・WWW-Authenticate・Set-Cookie等を保持する既存方針を維持する。

prepareErrorPageは呼び出し中だけconfigを参照し、ファイルを開かず指示を値として返す。Clientが元の応答とFileReadTaskを保持し、読み取り結果をfinishErrorPageへ渡す。空ファイルは成功した空本文、読み取り失敗は内蔵HTMLとする。HttpResponseは所有fdを持たず、ファイル読み取り処理を行わない。serializeは生成文字列をキャッシュせず、Clientが1回呼んで送信バッファへswapする。

今回採用した範囲はこのデータ構成と責務分担。setHeader等へ渡す値の前提、標準説明文の検索と未登録コードの扱い、private関数・ヘッダー/実装ファイルの分割は次の具体化対象として残す。公開APIや既存のCGI出力受付範囲は変更しない。

<a id="http-response-private-api"></a>

#### HttpResponseの入力契約・private宣言・ファイル分割（2026-10-10採用）

2026-10-10、rysatoの了承により以下の入力契約・private宣言・ファイル分割を採用した。rysato担当のHttpResponse内部を具体化したもので、tasugiya担当クラスの内部設計や、合意済みの共同APIは変更しない。前節の4メンバーと責務分担を維持する。実装・動作試験の完了は示さない。

**setterは、サーバーが生成した値または検証済みの値を受け取る内部APIとする。** CGIの生出力を直接渡さず、rysato側のRouter::parseCgiOutputで検証・除外・重複確認を完了してから設定する。setterに同じCGIパーサーを重ねたり、不正入力を無言で別のコードや文字列へ補正したりしない。通常の入力不正は既存の担当箇所でHTTPコードとして処理し、setter用の独自例外や公開エラー状態は追加しない。

| API | 呼び出し側の前提とsetterの扱い |
|---|---|
| setStatus(code) / setStatus(code, reason) | 最終応答の200〜599を渡す。CGIの1xx・範囲外は既存のCGI検証で502にする。説明文付きの場合は検証済みの値を渡し、CR/LF・HTAB以外の制御文字・DELを含めない。空の説明文は受理し、そのまま保持する |
| setHeader(key, value) | keyは空でないHTTP tokenで、コロンや周囲の空白を含まない。valueはCR/LF・HTAB以外の制御文字・DELを含まない値。名前のASCII小文字化だけを行い、値を再trim・エンコードしない。空値は空のヘッダー値として格納し、削除の意味にはしない |
| setBody(body, contentType) | bodyはNULを含む任意のバイト列。contentTypeはサーバーが選んだ値またはCGI検証済みの値。非空ならcontent-typeを設定し、空なら既存のcontent-typeを削除する。本文は空でも格納し、content-lengthをそのサイズに更新する |
| reset() | 分離送信では本文の送信完了または打ち切り後に保持領域を回収する。resetは保持領域を解放して初期状態へ戻す。失敗した接続の復旧目的で繰り返し呼ばない |
| makeError(code) | サーバーが生成する400〜599の元応答用。本文は未確定。正常応答やリダイレクト、正常なCGIのエラー応答の作り直しには使わない |
| prepareErrorPage(config, file) | 応答生成後の共通送信準備から1回呼ぶ。CGIは元出力の検証後。fileはtrueの場合だけ有効 |
| finishErrorPage(readSucceeded, body) | error_pageを読む400〜599の保持応答に、読み取り結果を1回適用する。falseではbodyを使わず、元コードの内蔵HTMLへ戻す |
| serialize() | 各処理が正常に応答を組み立て終えた後に呼ぶ。生CGIの検証や不正なsetter呼び出しの救済を担当させない |

setBodyの空contentTypeは「Content-Typeを付けない」という内部指定であり、通常のCGI文書応答の必須Content-Typeを省略可能にする変更ではない。CGIの空値省略はRouter側で行い、setHeader(key, "")で代用しない。setHeaderの前提を満たさない呼び出しは内部実装の不具合として扱い、実装時に呼び出し箇所を確認する。メモリ確保失敗等は既存の例外処理方針に従い伝播させる。

**標準説明文は、HttpResponse内のprivate static関数で検索する。表にないコードの説明文は空文字列とする。** コードを500へ変更したり、別コードの説明文を流用したりしない。例えばsetStatus(599)はコード599・空の説明文とし、setStatus(599, "Custom failure")は指定の説明文を保持する。未登録かどうかと、最終応答として使用可能なコード範囲は別の判定とする。

コード→標準文言はcpp内の固定の対応表またはswitchで保持し、動的なmapや新しいクラスを作らない。Webserv自身が生成するコードはすべて対応表へ登録する。CGIの明示した説明文はこの検索を通さず、空も含めて維持する。個々の標準文言の一覧は実装時に既存の生成コードと照合する。

採用したprivate宣言は次のとおり。

```cpp
// HttpResponseのprivate部分に追加する宣言
static const char* defaultReasonPhrase(int code);
static std::string buildDefaultErrorBody(int code);
static bool shouldSendBody(int code);
static bool shouldSendContentLength(int code);
void removeBodyHeaders();
void appendHeaders(std::string& output) const;
```

| 関数 | 責務・戻り値 |
|---|---|
| defaultReasonPhrase | 静的寿命の標準文言を返す。未登録は空文字列を返し、NULLは返さない。setStatus(code)がreasonPhrase_へコピーする |
| buildDefaultErrorBody | 400〜599のコードと標準文言から小さなHTMLを作る。未登録ならコードだけでも成立するページにする。CGI由来のreasonPhrase_や要求URLは埋め込まず、独自のHTMLエスケープ処理をこのために増やさない |
| shouldSendBody | 本サーバーの最終応答200〜599に対し、204/205/304はfalse、それ以外はtrue |
| shouldSendContentLength | 同じ範囲に対し、204/304はfalse、それ以外はtrue。205では送信本文なしなので出力値は0 |
| removeBodyHeaders | CGI-131のcontent-*、etag、last-modified、digest、repr-digestをheaders_から除去する。その他のヘッダー・コード・説明文は変更しない |
| appendHeaders | headers_を走査してoutputへ追記する。保存済みcontent-length/connection/transfer-encodingは出力対象から外し、最終本文長とconnection: closeを生成する。最後の空行までをここで追加する |

prepareErrorPageの未指定時とfinishErrorPageの読み取り失敗時は、buildDefaultErrorBodyで元コードの内蔵HTMLを用意する。取得本文または内蔵HTMLを用意してから、removeBodyHeaders、setBody(置換本文, "text/html")の順で適用する。ステータス行用のreasonPhrase_は変更せず、makeErrorやprepareErrorPageを再帰的に呼ばない。指定ファイルのread/closeとfd解放はFileReadTask側で行う。

serializeはローカルな出力文字列にステータス行→appendHeaders→送信可能な本文を順に追加する。通常の本文はstringの長さ付きで追加してNULを保持する。コードと長さはASCII十進表記で生成し、ロケールの桁区切り等を混ぜない。ヘッダーだけ/本文だけの送信キャッシュや新しい送信状態は持たない。serialize自身はconstであり、元のbody_を空にして本文禁止コードへ対応するのではなく、出力時に省略する。

ファイル分割は次のとおり。

| ファイル | 内容と依存 |
|---|---|
| HttpResponse.hpp | include guard、公開/非公開宣言と4メンバー。`<string>`・`<map>`をincludeし、ServerConfigとFileReadRequestは前方宣言だけ置く |
| HttpResponse.cpp | 自分のhppとServerConfig.hppをincludeし、コンストラクター・setter・serialize・エラーページ処理を定義する。FileReadRequestの定義もincludeし、固定の標準文言と内蔵HTML生成・本文関連ヘッダーの除去をここへ置く。ファイル検査/読み取り用のOSヘッダーは不要 |

HttpRequest/Router/Client/EventLoopのヘッダーはHttpResponseにincludeしない。コピーコンストラクター・コピー代入・デストラクターは暗黙生成とし、makeErrorの値返しと応答の値としての受け渡しに使う。所有するfdや生ポインターを追加せず、実装用の補助関数を公開APIへ出さない。

今回の採用範囲はrysato担当内の入力契約・内部関数・ファイル分割。次の対象はrysato担当のRouterであり、tasugiya側や共同部分の未確定事項は本作業では進めない。

<a id="router-internals"></a>

### Routerの保持データと処理分担（2026-10-10採用）

2026-10-10、rysatoの了承により以下の保持方針と処理分担を採用した。HTTP-57・59・63とCGI-91〜98の公開APIを維持し、ネットワーク/プロセス制御側の設計は進めない。routeとparseCgiOutputの正確な公開シグネチャは[03の共有API](reports/03_config_cgi_upload.md#cgi-api)を正本とする。実装・動作試験の完了は示さない。

**Routerはデータメンバーを持たず、要求ごとの値はローカル変数と引数だけで扱う。** 現在のrequest/config/location/connection/response、ファイルパス、CGI出力等をメンバーへ保存しない。既存の「1個を使い回す」構成に合わせ、次の要求やCGI完了時の別呼び出しへ前の処理の値を残さない。独自のコンストラクター・デストラクター・コピー処理は不要とする。

routeはCOMPLETEのHttpRequest、検証済みServerConfig、既存契約を満たすConnectionInfoを受け取る。HttpRequestの受信解析をやり直さず、getPathのデコード/正規化も繰り返さない。設定の継承・既定値補完も行わない。

#### routeの共通処理とメソッド別処理

route本体は次の共通処理を担当し、ファイル処理等はprivate関数に分ける。

| 順序 | 処理 | 結果 |
|---|---|---|
| 1 | 正規化済みURLパスに最長前方一致するlocationを選ぶ | なければ404のHttpResponseを作りRESPONSE_READY |
| 2 | 選択したlocationのallow_methodsを確認する | 禁止なら405＋Allowを作りRESPONSE_READY。ファイル確認・保存・削除・CGI準備へ進まない |
| 3 | locationがREDIRECTかを確認する | Configのコードと固定Location、空本文の応答を作りRESPONSE_READY |
| 4 | SERVEならGET/POST/DELETEのprivateハンドラーへ渡す | ハンドラーのRouteResultをそのまま返す |

locationのメソッド禁止による405のAllowは、許可したメソッドをGET, POST, DELETEの固定順で生成する。CGI非対象へのPOSTを405にする経路では、HTTP-53に従い対象で使えるメソッドだけを返す。エラー応答は既存のHttpResponse::makeErrorを使い、必要なAllowを追加する。通常の呼び出しでは未対応メソッドはHttpRequestで501となっており、Routerに新しいメソッド受付機能を追加しない。

| private関数の分担 | 責務 |
|---|---|
| findLocation | config.getLocations()を走査し、最長一致のconst LocationConfig*を返す。マッチなしはNULL。設定順に依存せず、独自の検索木/キャッシュは作らない |
| isMethodAllowed / buildAllow | locationの許可メソッドの確認と、その禁止による405のAllow値の生成。CGI非対象POSTの405は対象で使えるメソッドに絞る。補助関数の具体的な分割は未決定 |
| resolvePath | Config解決済みroot/aliasと正規化済みURLパスを既存規則で結合する。URLを再デコードせず、upload_storeをGET/DELETE先へ流用しない |
| handleGet | CGI候補の探索と通常の対象確認を既存規則に沿って進める。静的な通常ファイルはパス・Content-TypeをFileReadRequestへ設定しFILE_READ_REQUIREDを返す。ディレクトリは末尾/補完→index→未発見ならautoindex。選ばれたindexにもCGI判定を適用し、静的indexは同じ読み取り待ちへ進む |
| handlePost | upload有効ならmultipart全体を検証し、FileWriteRequestとFILE_WRITE_REQUIREDを返す。保存・成功応答はClient側の完了経路へ委ね、CGI有効ならスクリプト解決と実行要求の準備へ進む。CGI向け本文にアップロード用の形式制限を適用しない |
| handleDelete | root/aliasで解決した通常ファイルだけを削除し、成功後に204を作る。GET用の末尾/補完・index・autoindexへ進めない |
| prepareCgiRequest | 選択済みスクリプト・SCRIPT_NAME/PATH_INFO・要求/Config/接続情報から、既存CgiRequestのパス・cwd・環境変数・本文を完成させる。CGI起動は行わない |

handleGet/handlePost/handleDeleteは共通してRouteResultを返し、即時応答ならresponse、CGIが必要ならcgi、静的ファイル読み取り待ちならfileを完成させる。handleDeleteは常にRESPONSE_READYでresponseだけを返し、成功は204、失敗は既存分類のエラー元応答とする。makeErrorを使うエラー経路はコード/標準説明文の元応答を返し、後続の共通error_page準備経路で本文を確定する。すべての失敗を常に即時送信できるとは扱わない。prepareCgiRequest等の下位の検証関数は成功0/失敗HTTPコードを返す形を基本とし、細かな引数・補助型は次に具体化する。

GETでは`/cgi/test.py/extra`のようなPATH_INFO付き要求を、全体パスのstat失敗だけで拒否しない。設定拡張子に一致する最初の実在通常ファイルを左から探す既存のCGI-25/85を維持する。scriptが見つかった後のPATH_INFOはファイル確認しない。ディレクトリやindexの処理、CGIに該当しない対象との分岐は、この規則を保ってhandleGet以下で具体化する。

uploadとCGIの同一locationでの併用禁止、配信POSTの処理先必須、CGI有効locationのDELETE禁止はConfigで検証済み。Routerで設定値を補完して救済しない。メソッド別の失敗分類、保存完了前の成功禁止、途中ファイルの後始末、通常ファイルI/Oは既存のHTTP/CGI合意を維持する。

#### 出力引数と一時データ

RESPONSE_READYではresponseだけ、CGI_REQUIREDではcgiだけ、FILE_READ_REQUIREDではfileだけ、FILE_WRITE_REQUIREDではwriteだけ、AUTOINDEX_REQUIREDではautoindexだけを使う。即時応答・CGI・読み取り指示・保存指示・一覧生成指示の各経路はそれぞれ必要な値を完成させ、前回の処理の値を引き継がない。使用しない出力引数を整えるためだけの初期化は要求しない。FileReadRequestは必要な間だけ保持し、FileReadTask開始後にその参照を残さない。

処理中のLocationConfig*はConfig内の要素への非所有参照とし、削除・コピー保持しない。multipartの対象範囲、CGIスクリプト情報等は各関数内で保持し、Routerのメンバーにしない。静的ファイルのパスとContent-Typeはfileへ値として渡し、fdは渡さない。Router内で使うその他のファイル/ディレクトリのハンドルは担当関数内で解放する。FileReadTaskが開いたfdはClient側で保持・解放する。出力構築中のメモリ確保失敗は既存の例外処理方針へ伝播し、正常なRouteResultが返らなかった出力を有効と扱わない。

#### parseCgiOutputの独立性

parseCgiOutputは、引数で渡された完成済みstdoutだけから検証済みHttpResponseを構築する。過去のrouteで保存したメンバーを必要とせず、Config・Client・実行中CGIの情報を参照しない。行分割・全ヘッダーの重複/構文検証・応答形式の判定・本文長/本文禁止規則の確認を済ませてから、setterへ値を渡す。

正常時は0、不正なCGI出力は502を返す既存APIを維持する。0のときだけresponseが有効であり、不正な出力の途中結果を成功応答として使わない。ここではerror_pageを適用しない。正常なCGIの404等も元の応答として検証・構築し、本文置換は既存の適用箇所へ任せる。既存の呼び出し契約の説明に留め、共同APIを変更しない。

2026-11-15に読み取り対象の受け渡しを拡張した。Routerはデータメンバーなし・共通処理の順序・private関数の分担を維持する。以下でrysato側のパス/CGI対象解決に必要なローカル補助型と各ハンドラーの宣言を具体化し、その後にアップロード・CGI出力解析の内部構成へ進む。新しいHTTP動作が必要と分かった箇所は、内部実装の名目で確定せず、既存合意と分けて提示する。

<a id="router-target-api"></a>

#### RouterのCGI対象の補助型とハンドラーAPI（2026-11-16採用）

2026-11-16、CgiTargetをRouterのprivateな型として採用した。scriptPath/scriptName/pathInfoの3文字列を値で持ち、RouterのメンバーではなくGET/POST処理内のローカル変数として使う。通常のパス結合結果はstd::stringで十分とし、別のパス用クラスを増やさない。findCgiScriptとprepareCgiRequestの宣言/返却契約も2026-11-16に採用した。GET/POST/DELETEハンドラーの入力/出力、ServerConfig引数を渡さないこと、privateなconstメンバー関数としてRouteResultを返す宣言も2026-11-16に採用した。その他の共通補助関数の正式な宣言は引き続き提案段階。ConnectionInfo/CgiRequestは維持する。

```cpp
// Routerのprivate部分に置く型。Routerのデータメンバーにはしない。
struct CgiTarget {
    std::string scriptPath; // ディスク上のスクリプト絶対パス
    std::string scriptName; // SCRIPT_NAMEに使うURL上のパス
    std::string pathInfo;   // PATH_INFO。なければ空、/だけの場合は/を保持
};
```

CgiTargetはhandleGet/handlePost内のローカル変数として生成する。fd・Configへの参照・要求本文を持たせない。PythonのパスはLocationConfigから取得でき、cwdはscriptPathの親から生成できるので、この型に重複保持しない。CgiRequestはプロセス側へ渡す完成データ、CgiTargetはその前段階のRouter内部データとして区別する。

下記のfindCgiScript/prepareCgiRequestとhandleGet/handlePost/handleDeleteの宣言は採用済み。findLocation/isMethodAllowed/buildAllow/resolvePathの正式な宣言は提案段階。Routerの内部関数はconstメンバー関数として揃え、公開APIのconst有無は変更しない。

```cpp
const LocationConfig* findLocation(const std::string& urlPath,
    const ServerConfig& config) const;
bool isMethodAllowed(const std::string& method,
    const LocationConfig& location) const;
std::string buildAllow(const LocationConfig& location) const;
std::string resolvePath(const std::string& urlPath,
    const LocationConfig& location) const;

RouteResult handleGet(const HttpRequest& request,
    const LocationConfig& location,
    const ConnectionInfo& connection,
    HttpResponse& response, CgiRequest& cgi,
    FileReadRequest& file, AutoindexRequest& autoindex) const;
RouteResult handlePost(const HttpRequest& request,
    const LocationConfig& location,
    const ConnectionInfo& connection,
    HttpResponse& response, CgiRequest& cgi,
    FileWriteRequest& write) const;
RouteResult handleDelete(const HttpRequest& request,
    const LocationConfig& location,
    HttpResponse& response) const;

int findCgiScript(const std::string& urlPath,
    const LocationConfig& location, CgiTarget& target, bool& found) const;
int prepareCgiRequest(const HttpRequest& request,
    const LocationConfig& location, const ConnectionInfo& connection,
    const CgiTarget& target, CgiRequest& cgi) const;
```

3ハンドラーの返却契約も採用済み。

| privateなconstハンドラー | 入力 | RouteResultと有効な出力 |
|---|---|---|
| handleGet | request/location/connection | RESPONSE_READYならresponse、CGI_REQUIREDならcgi、FILE_READ_REQUIREDならfile、AUTOINDEX_REQUIREDならautoindex |
| handlePost | request/location/connection | RESPONSE_READYならresponse、CGI_REQUIREDならcgi、FILE_WRITE_REQUIREDならwrite。読み取り用のfileは受け取らない |
| handleDelete | request/location | 常にRESPONSE_READY。responseだけが有効 |

対応しない出力引数を増やさず、返却値に対応する出力だけを使う。handleDeleteは成功時の204だけでなく、既存の失敗分類によるエラー応答もRESPONSE_READYとして返す。これはエラー本文確定や即時送信を意味せず、後続の共通送信準備でerror_pageを処理する。

handleDeleteには使わないConnectionInfo/CgiRequest/FileReadRequestを渡さない。各ハンドラーに渡すlocationはrouteで選択・許可判定・REDIRECT判定を終えたSERVE用のものとし、同じlocationを選び直さない。2026-11-16合意により、GET/POST/DELETEの3ハンドラーからServerConfig引数を外す。makeError(code)は設定を必要とせず、error_pageの設定参照は後続のClient側で行う。配信先・index・CGI/アップロード設定は選択済みLocationConfigから取得する。Router本体にはlocation選択用のServerConfigを引き続き渡す。prepareCgiRequestにServerConfigを渡さないのは、本文上限の再判定やserver設定の再解釈を行わないため。

#### パス結合とCGI探索の契約

resolvePathは文字列の結合だけを担当し、stat/accessやHTTP応答生成を行わない。rootならbasePathにURL全体を追加し、結合境界で重なる/を1個にする。aliasなら選択済みprefixを取り除いた残りをbasePathに追加する。ディレクトリを示す末尾/は保持し、GETのリダイレクト判定を結合処理に入れない。URLに由来する%の再解釈や、設定パスに残る`.`/`..`の独自の正規化も行わない。

findCgiScriptはRouterのprivateなconstメンバー関数とし、上記のint戻り値・CgiTarget&とbool&出力引数の宣言を2026-11-16に採用した。結果は次の3通りで区別し、非該当とファイル確認失敗を同じboolだけで表さない。

| 戻り値 | found | 意味 |
|---|---|---|
| 0 | true | 読み取り可能な実在通常ファイルのCGI対象が確定し、targetの3項目が有効 |
| 0 | false | CGI無効、または探索に失敗せず終了したが対象がCGIスクリプトに該当しない。targetは使わない |
| 400〜599の既存エラーコード | false | stat/access等の失敗または禁止種別。ハンドラーがエラー応答へ変換し、通常配信へフォールバックしない |

関数の入口でfound=false、targetの文字列は空に戻す。探索完了前の部分的なtargetは使用させず、すべて確認できてからfound=trueにする。型・戻り値はCGI探索の内部契約であり、2026-11-15に追加したFILE_READ_REQUIREDとは別に、CGI探索専用の公開RouteResultは増やさない。

CGI有効なら、選択済みlocationの対応先に沿って、URLのパス要素の区切りを左から確認する。設定拡張子に一致する最初の実在通常ファイルを見つけたらaccess(R_OK)を確認し、scriptPath・scriptName・残りのpathInfoを確定して探索を終える。拡張子比較は大小を区別する。途中のディレクトリ名が`.py`等で終わっていても実行対象にしない。通常ファイル・ディレクトリ以外の種別やstat/accessの失敗は既存のCGI-86/90に従う。

この関数はindexを自動選択しない。ディレクトリまで辿ってCGIが見つからなければ0/falseで返し、呼び出し側がそのメソッドのディレクトリ処理へ進む。GETまたはPOSTで設定indexが選ばれた場合は、そのURLパスを対象として同じCGI判定を適用する。indexが見つかった後に別のlocationへ再ルーティングしない。POSTでは末尾/の補完リダイレクトを行わず、選択済みlocation内でindexを探す。POSTでCGI非該当の場合は下記の405規則へ進む。GETは既存の通常配信/ディレクトリ処理へ進む。実ディレクトリで設定indexが不在なら、下記の規則に従い403とする。

例: aliasが`/srv/cgi/`のlocation `/cgi/`で、要求が`/cgi/test.py/extra`なら、targetはscriptPath=`/srv/cgi/test.py`、scriptName=`/cgi/test.py`、pathInfo=`/extra`。`/srv/cgi/test.py/extra`の存在は確認しない。GETの`/cgi/`でindex.pyが選ばれた場合は、scriptName=`/cgi/index.py`、pathInfoは空とする。

prepareCgiRequestはRouterのprivateなconstメンバー関数とし、上記の宣言を2026-11-16に採用した。HttpRequest/LocationConfig/ConnectionInfoと確定済みCgiTargetをconst参照で受け取り、CgiRequestを出力する。有効なtargetを前提に、既存のCGI入力検証と環境変数生成を行う。成功0ならcgiの全項目が完成、失敗コードならcgiを使わずハンドラーがmakeErrorへ進む。cgi.bodyへはrequest.getBody()をコピーする既存契約を維持する。スクリプトの再探索、Pythonの要求ごとの事前検査、fork/pipe/execveは行わない。SCRIPT_NAME/PATH_INFOをディスク上のパスから逆算せず、targetのURL側の値を使う。

| 戻り値 | 出力と呼び出し側の処理 |
|---|---|
| 0 | Python絶対パス・スクリプト絶対パス・cwd・環境変数・復号済み本文を備えたcgiが有効。ハンドラーはCGI_REQUIREDとして返す |
| 既存のHTTPエラーコード | cgiは使用せず、ハンドラーがmakeError(code)で元応答を作る。共通送信準備への受け渡しは既存の規則に従う |

request/target等への参照をcgiに残さず、必要な値をCgiRequestへ保持する。成功時だけ出力を使用する契約とし、部分的な出力でCGIを起動しない。メモリ確保例外を通常の検証失敗コードへ変換せず、既存のbad_alloc時の資源回収/接続終了へ進む。

<a id="cgi-post-directory"></a>

#### CGI有効locationへのディレクトリPOST(2026-11-16合意)

POSTのメソッド許可・Configのreturn判定を終え、CGI処理へ進む選択済みlocationで、対象が実ディレクトリなら設定indexを探す。選ばれたindexがCGI対象なら通常のスクリプト検証・実行要求生成へ進み、元のPOST本文をstdinへ渡す。GET専用の末尾スラッシュ補完リダイレクトは行わない。末尾スラッシュの有無だけで別locationを探索せず、現在のlocationに一致した要求を対象とする。

例: location /cgi/、alias /srv/cgi/、index index.py、POST許可、CGI拡張子.pyの場合、/srv/cgi/tools/が実ディレクトリならPOST /cgi/tools/は/srv/cgi/tools/index.pyを探す。CGI対象の通常ファイルならSCRIPT_NAME=/cgi/tools/index.py、PATH_INFOは空、QUERY_STRINGとPOST本文は元の要求から渡す。同じlocationに一致するPOST /cgi/toolsも、toolsが実ディレクトリならリダイレクトせず同じindexを探す。

この合意はupload_storeによるPOST保存の規則を変えず、indexをCGIとして実行するかは既存のCGI条件で判断する。実在する通常ファイルまたは選択indexがCGI拡張子に該当しない場合は、下記の405規則へ進む。実ディレクトリで設定indexが不在なら403とし、autoindexのon/offによらず一覧へ進まない。対象ディレクトリ自体の不在や権限確認等の既存エラー分類は維持する。

<a id="cgi-post-non-script"></a>

#### CGI非対象へのPOSTとAllow(2026-11-16合意)

CGI有効locationでPOSTが許可されていても、実在する対象の通常ファイルまたはディレクトリで選択した通常ファイルのindexがCGI拡張子に該当しない場合は405 Method Not Allowedを返す。CGI実行、upload_storeへの保存、静的本文の成功応答への切り替えを行わない。パス不在・権限不足・非通常ファイル等は既存のエラー分類を維持し、すべてをこの405へ置き換えない。

| locationの許可メソッド | CGI非対象POSTの応答 |
|---|---|
| GETとPOST | 405、Allow: GET |
| POSTのみ | 405、空のAllowヘッダー(Allow:) |

この経路のAllowは対象の利用可能なメソッドを表すため、処理できないPOSTを含めない。GETはlocationで許可されている場合だけ載せる。CGI有効locationではDELETEを許可しない既存のConfig規則を維持し、HEAD等も追加しない。空のAllowはヘッダー省略と区別し、名前とコロンを出力する。後続のerror_page本文置換でもAllowを保持する。

location段階で禁止されたメソッドの405は、従来どおりlocationの許可一覧を使う。この対象確認後の405と発生段階を区別し、buildAllow(location)の結果をそのまま流用しない。補助関数の正式な宣言はRouter内部APIの具体化で決める。実ディレクトリの設定index不在は下記の403規則へ進む。

<a id="cgi-post-missing-index"></a>

#### CGIディレクトリPOSTのindex不在(2026-11-16合意)

CGI有効/POST許可のlocationで、要求対象のディレクトリは実在するが設定indexが不在なら403 Forbiddenを返す。ディレクトリはあるが、このPOSTを受け付けるCGIがないため拒否するチーム方針とする。autoindexがonでもPOSTへ一覧を返さず、GETのindex不在時の処理へ切り替えない。別スクリプトの自動補完、upload保存、リダイレクト、再ルーティングを行わない。

| 条件 | 応答・処理 |
|---|---|
| 実ディレクトリ内の設定indexがCGI対象 | スクリプト検証後、POST本文を渡してCGI実行 |
| 実ディレクトリ内の設定indexが不在 | 403。autoindexへ進まない |
| 実在する通常ファイルのindexがCGI非対象 | 405と対象別Allow |
| 要求対象のディレクトリ自体が不在 | 既存どおり404 |

設定index不在だけを上記の403として扱い、要求対象のパス不在まで403へ変更しない。権限不足・非通常ファイル・パス長等は既存の失敗分類を維持する。403の元応答はmakeError(403)で作り、共通送信準備へ渡す。error_page指定/内蔵HTMLのどちらでも元の403を維持する。

#### ハンドラーの動作判断と残る内部設計

ディレクトリPOSTで設定indexを探す方針、CGI非対象なら405・対象別Allow、実ディレクトリの設定index不在なら403・一覧へ切り替えない方針は2026-11-16に合意済み。これで当該POSTの動作判断は確定した。CgiTargetの3文字列とRouter内ローカル変数としての使用は同日に合意済み。findCgiScriptのprivate宣言と発見/非該当/確認失敗の返却契約、prepareCgiRequestのprivate宣言と実行要求生成/返却契約も同日に採用済み。3ハンドラーからServerConfigを外す方針も合意済み。3ハンドラーの残る入力/出力とRouteResultの返却契約も採用済み。その他のRouterの共通補助関数の正式な宣言は提案段階として区別する。

### CGIの共有型・受け渡しAPI

ConnectionInfo / CgiRequest / CgiResultと、Router::parseCgiOutput、CgiExecutor::start、CgiProcess::takeResultはCGI-91〜98を維持する。RouteResult/Router::routeは2026-11-15にFileReadRequestとFILE_READ_REQUIREDを追加した。宣言の正本と担当境界は [03の共有API](reports/03_config_cgi_upload.md#cgi-api) を参照する。

### Config 系

Config / ServerConfig / LocationConfigの型、getter、ConfigParser::parse、エラー処理、寿命はC-42で合意済み。公開APIは [05の型とAPI](reports/05_config_agreement.md#config-api) を参照する。

最長前方一致・マッチなし404は確定済み。findLocation等はRouter内部の実装詳細で、必要な公開APIを追加する未決定事項とは扱わない。

### EventLoop / Client / CgiProcess(tasugiya側内部)

2026-11-17更新: ClientのStateにREADING_FILEとWRITING_FILEを追加した。保存中の基本遷移は[保存状態の正本](#client-file-write-state)に従う。Clientの値メンバーFileWriteTaskには完全な型定義が必要なためClient.hppでFileWriteTask.hppをincludeする。[静的ファイル読み取りの基本遷移](#client-file-read-state)、FileReadTaskの値メンバー保持、FileReadTask/Clientのコピー禁止を合意済み。FileReadTaskはClientと一緒に生成・破棄し、未開始時はfd=-1・空本文とする。Client.hppでは値メンバーの完全な型定義が必要になるためFileReadTask.hppをincludeする。FileReadPurposeの2用途と読み取り成功/失敗の処理も合意済み。応答は値メンバーpendingResponse_で保持する方針も合意済み。読み取り期限・異常時の遷移/後始末・例外時の処理・EventLoopの進行APIは未決定であり、下記の宣言だけで新方式が完成したとは扱わない。

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

// FileReadTask/Clientの所有・コピー制約を示す抜粋。
// start/step/takeBody/cancelの4操作と役割は合意済み。
// 4操作の契約・privateなPhase・通常失敗の処理は合意済み。bad_allocは接続終了。
class FileReadTask {
public:
    enum StepResult { IN_PROGRESS, COMPLETED, FAILED };
    FileReadTask();  // 未開始、fd=-1、本文は空
    ~FileReadTask(); // cancelと同じ解放処理。例外を外へ出さない
    bool start(const FileReadRequest& request); // 未開始時のみ。再利用前はcancelが必要
    StepResult step(); // 最大8 KiBのreadを1回。EOFとclose成功でCOMPLETED
    bool takeBody(std::string& body); // 完了後の初回だけswapしてtrue。それ以外は変更せずfalse
    void cancel(); // fdを-1にして一度だけclose、本文を解放し未開始へ。再呼び出し可
private:
    FileReadTask(const FileReadTask&);            // 未定義
    FileReadTask& operator=(const FileReadTask&); // 未定義
    enum Phase { IDLE, READING, DONE, ERROR, TAKEN };
    Phase phase_; // 初期状態とcancel後はIDLE
    // 所有するファイルfd、読み取り済み本文
};

class Client {
public:
    void cleanupInChild(); // CGI起動失敗の子専用。Client破棄前に呼ぶ
    void prepareResponse(const HttpResponse& response); // コピーで受け取り、共通送信準備へ進む
    void prepareResponseBySwap(HttpResponse& response); // swapで受け取り、共通送信準備へ進む
    enum State { READING_REQUEST, PROCESSING, WAITING_CGI, READING_FILE,
                 WRITING_FILE, GENERATING_AUTOINDEX, WRITING_RESPONSE,
                 DRAINING, CLOSING };
private:
    void beginWritingResponse();     // 確定応答のヘッダー生成・分離送信準備・計時開始
    Client(const Client&);            // 未定義
    Client& operator=(const Client&); // 未定義
    enum FileReadPurpose { STATIC_FILE, ERROR_PAGE };
    FileReadPurpose fileReadPurpose_; // READING_FILE中だけ参照。読み取り開始時に設定
    bool fileTaskStartedThisRound_;   // 周頭にfalse、タスク開始時true。trueなら今周のstepを省く
    FileReadTask fileTask_;           // 値メンバー。タスク自体のnew/deleteは不要
    AutoindexTask autoindexTask_;     // 値メンバー。列挙/ソート/HTML生成の途中状態を所有
    HttpResponse pendingResponse_;   // 値メンバー。本文確定後にserializeして送信バッファへ渡す
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
