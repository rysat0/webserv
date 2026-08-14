# webserv 要件・決定事項一覧

> 課題PDF (Version 24.1) の全要求事項と、チームの設計決定をまとめた内部ドキュメント。
> ※提出用 README.md(英語・別途作成)とは別物。

---

## 1. プロジェクト概要

- **C++98でHTTPサーバーを自作する**
- 実行形式: `./webserv [configuration file]`
- 設定ファイルは引数で渡されるか、デフォルトパスから読み込めること
- 実際のWebブラウザでテスト可能であること
- HTTP 1.0 が参考基準(強制ではない)。RFCを読むこと、telnet と NGINX で事前にテストすることが推奨されている

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
| 5 | HttpRequest | バイト断片 → 完成判定+構造化(chunked 対応) | B |
| 6 | Router | Request+Config → 静的/CGI/アップロード/エラー振り分け | B |
| 7 | HttpResponse | ステータス+ヘッダー+ボディ → 送信バイト列 | B |
| 8 | CgiHandler | fork/execve/pipe 管理、環境変数構築 | 共同 |
| 9 | Utils | ログ、変換系の小物 | 共同 |

## 実装順序

1. ConfigParser(B)/ 最小 Hello World サーバー(A)← 並行スタート
2. EventLoop + Client(poll 化・複数接続)
3. HttpRequest パーサー(GET+ヘッダーから)
4. Router + HttpResponse(静的配信)← 合流ポイント
5. エラーページ・autoindex・DELETE
6. POST(アップロード)+ chunked
7. CgiHandler(共同)
8. keep-alive、タイムアウト、body size 制限、ストレステスト

## 決定事項(2026-08-14 確定)

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
| keep-alive | 後回し。まず `Connection: close` 固定で骨格を作り、フェーズ8で余裕があれば対応 |
| HTTPバージョン | **HTTP/1.1** を名乗る(当面 `Connection: close` を併記) |
| ステータスコード | 主要コードを網羅的に実装(基本全部) |
| デフォルト設定パス | `./conf/default.conf`(引数なし起動時に読む) |
| シグナル処理 | `SIGPIPE` は `SIG_IGN` で無視(send失敗時はerrnoを見ず接続close)。`SIGINT` はフラグ方式でグレースフル終了(全fd close・全メモリ解放してから終了) |
| CGI環境変数 | RFC 3875 準拠: `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR, SERVER_SOFTWARE` + 全HTTPヘッダーを `HTTP_*` 形式(大文字化・`-`→`_`)で転送 |
| ConfigParser構成 | 1クラス(トークナイザとパーサーはprivate関数で分離) |
| CGI状態の持ち方 | 独立クラス `CgiProcess`(fd→オブジェクトの対応表で引ける形) |
| Router構成 | 1クラス+private関数分割(`handleGet/handlePost/handleDelete/handleCgi`...) |
| pollfd配列管理 | 毎周再構築(Client/CGI一覧から配列と監視フラグを毎回計算) |
| 例外方針 | 起動フェーズのみ例外可・ループ内は戻り値方式。保険としてループに `catch(std::exception&)` の防波堤(ログ+接続close)。bad_alloc もここで受ける |

## 未決定(次に決めること)

- [ ] ディレクティブ名の**具体的な一覧表**の作成(必須/任意の区別を含む)
- [ ] HttpRequest / HttpResponse の**具体的なシグネチャ**(メソッド名・戻り値の確定)

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
| `HttpRequest` | 逐次パース(断片投入→状態遷移→完成判定)。chunked デコード含む |
| `HttpResponse` | レスポンス組み立て+`serialize()`+エラーページ工場 |
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
    int    getErrorCode() const;      // ERROR 時: 400, 413, 501 など
    void   setMaxBodySize(size_t n);  // Config の値を注入(早期413判定)

    // Router(B側)が COMPLETE 後に呼ぶ
    const std::string& getMethod() const;
    const std::string& getPath() const;         // ? 以降除去済み
    const std::string& getQueryString() const;  // ? の後ろ、生のまま
    const std::string& getVersion() const;
    std::string        getHeader(const std::string& key) const;  // 大小無視
    const std::string& getBody() const;         // un-chunk 済み
};
```

- エラー判定(400/413/501)はパース時点で Request 自身が行う
- ヘッダーキーは内部で小文字化して格納
- chunked デコードはこのクラスに閉じる(CGI の un-chunk 要件を自動で満たす)

### HttpResponse

```cpp
class HttpResponse {
public:
    HttpResponse();
    void setStatus(int code);   // reason phrase は内部表で自動解決
    void setHeader(const std::string& key, const std::string& value);
    void setBody(const std::string& body, const std::string& contentType);
                                // Content-Length / Content-Type 自動設定
    std::string serialize() const;
                                // Server, Date, Connection: close 自動付与
    static HttpResponse makeError(int code, const ServerConfig& conf);
                                // error_page 指定 or 内蔵デフォルトHTML
};
```

- `\r\n` の組み立ては serialize() に封じ込める
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
