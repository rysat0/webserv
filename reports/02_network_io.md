# 02 ネットワーク・I/O — 理解すべきこと(主担当: tasugiya、一部共同)

一次情報: `man 2 socket/bind/listen/accept/recv/send/poll/fcntl/shutdown/waitpid/signal`, `man 7 socket/tcp/pipe/signal`, `man 3 getaddrinfo`。

課題の**0点条件がここに集中**している(クラッシュ、poll外のI/O、errno参照)。仕様を書く前に全員が理解しておくべき章。

---

- **確定(C-31)**: HTTP/1.0・HTTP/1.1要求のリクエスト行とHTTPヘッダーの改行はCRLFだけを受理し、単独LF・不正な単独CRは400とする。受信断片の末尾がCRだけなら、その時点では不正とせず次のバイトを待つ。次がLFなら正常なCRLFとして扱い、LF以外なら不正と判定する。切断やタイムアウトで未完了になった場合は既存の接続処理に従う。この改行制限はボディのデータには適用せず、ボディ中のCR/LFを変更・拒否しない。CGI出力ヘッダーのLF/CRLF両方受理と、HTTP応答ヘッダーのCRLF生成はCGI-15の既決定を維持する。
- HTTP要求ヘッダーの折り返し・空白処理はC-33に従う。rysatoのパーサーで行頭SP/HTABやコロン直前の空白を400にし、値の前後SP/HTABだけを除去する。tasugiyaのClientは受信断片をそのまま渡し、ボディも含めた一括trimや空白だけの行の読み飛ばしを行わない。
- ヘッダー名のtoken検証とASCII小文字での保持はrysatoのHttpRequestが行う(C-34)。tasugiyaのClientで受信データ全体を小文字化しない。値とボディの大小文字は保持する。
- 要求のContent-Lengthは同値でも重複・カンマ入りを400とする(C-35)。rysatoのパーサーは複数の受信断片にまたがる重複も検出する。tasugiyaのClientはERRORをボディ完了まで待たず既存の応答経路で扱う。
- CL/TEが両方ない要求はPOSTもヘッダー終端で空ボディとして受信完了する(C-36)。Clientは未到着のボディや切断を待たず通常の処理へ進め、1要求後の余剰データは処理しない。

HTTP/1.2〜1.9の要求はHTTP/1.1と同じ規則で扱う(HTTP-04)。受信した版の表記はHttpRequestとCGIのSERVER_PROTOCOLに保持する。

## N1 TCP/ソケットAPI 【tasugiya, R:M, D:M, ★★★】

**理解すべきこと**

- サーバー側の流れ: `socket` → `setsockopt(SO_REUSEADDR)` → `bind` → `listen` → `accept`。各ステップの失敗時の扱い(**起動時の失敗のみ例外/終了可**、という決定事項との整合)。
- **確定(C-39)**: listenを省略したserverは、起動権限に関係なく`0.0.0.0:8000`で待ち受ける。権限判定は実装しない。C-05のNGINX既定値採用方針のうちlistenだけをこの固定値に変更し、他の項目の既定値は維持する。明示したlistenはC-10・C-11に従ってそのIP・ポートを使用し、80番も指定可能とする。既定値補完後の重複・競合はC-12に従って起動エラーとし、複数serverでlistenを省略して同一アドレスになった場合も拒否する。bind失敗時は起動を中止し、別IP・別ポートへ自動変更しない。
- `SO_REUSEADDR` の意味(TIME_WAIT中の再bind。付けないと再起動時に `bind` が失敗する)。
- `listen` の backlog の意味と上限(`/proc/sys/net/core/somaxconn`)。
- `accept` はlistenソケットが**読み取り可能**になってから呼ぶ(poll対象)。1回のPOLLINで複数接続が溜まっている可能性 → 1回だけ受けるか、ループするか(read/write等のerrno禁止とacceptの失敗処理を区別し、具体的な手順はN5で決める)。
- 許可関数に `setsockopt`, `getsockname`, `getprotobyname`, `htons` 系がある。`getsockname` は「ポート0で bind したとき実ポート取得」「接続ごとの SERVER_PORT/ローカルアドレス取得」で使える。
- **確定(C-28)**: tasugiyaはacceptした接続に対するgetsocknameでサーバー側IP・ポートを取得し、rysatoのHTTP/Router側へ渡す。HTTP/1.0でHostがない場合の末尾/補完301に使う。listenの0.0.0.0やacceptで得たクライアント側アドレスを代用しない。有効なHostがある場合はその明示ポートごとURLに使い、Hostにポートがなければ接続のポートを自動追加しない。ConnectionInfoの受け渡しはCGI-93で合意済み。Client側の保持方法と取得失敗時の扱いは、残るHTTP/ネットワーク設計で決める。
- **確定(C-30)**: Hostの受付検証はrysatoがHTTP層で行う。Host値のためのDNS問い合わせはせず、接続先やServerConfigも切り替えない。Hostの名前/任意ポートの受付規則は01・05のC-30を参照する。
- **確定(CGI-37)**: HTTP/1.0でHostがないCGI要求のSERVER_NAMEには、C-28と同じ接続のサーバー側IPをgetsocknameで取得して使う。listenの0.0.0.0や接続元IPで代用しない。接続情報はCGI-93のConnectionInfoで渡す。Client側の保持方法と取得失敗時の扱いは、残るHTTP/ネットワーク設計で決める。
- **確定(CGI-36)**: REMOTE_ADDRにはacceptで取得した接続元のIPv4アドレスをドット区切りの10進数で必ず設定する。ポート番号は含めず、逆引きDNSやX-Forwarded-For等による上書きを行わない。4オクテットの十進表記とConnectionInfoによる受け渡しはCGI-93で合意済み。
- `accept` で得た接続のリモートアドレス(`REMOTE_ADDR` 用)の取得と文字列化(CGI-93。inet_ntop/inet_ntoaを追加しない)。

> `inet_ntoa`/`inet_ntop`/`inet_addr`は許可関数一覧にない。CGIのREMOTE_ADDR等のIPv4文字列化は4オクテットを十進表記にする方針で合意済み(CGI-93)。

## N2 アドレス解決・複数listen 【tasugiya, R:M, D:S, ★★★】

- `getaddrinfo`/`freeaddrinfo`/`gai_strerror` の使い方(`AI_PASSIVE`、`host` 省略時 = 全インターフェース)。listen設定はIPv4:portのみで確定(C-10)。IPv4用の `AF_INET` を使い、ホスト名やIPv6の設定は受理しない。
- **確定(C-11)**: listenの明示は1serverに最大1回。複数ポートはserverを分け、省略時は既定値で1組を補完する。
- 異なるserver間でlisten先が重複・競合する場合:
  - **確定(C-12)**: 同一IPv4:portの重複は設定エラーで起動中止。`0.0.0.0:8080` と `127.0.0.1:8080` のような同じポートでの全IPv4/特定IPの競合も拒否する。listen省略時も既定値を補完してから検証する。
  - 異なる特定IPv4アドレス同士は同じポートを設定できる。実際にbindできるかは起動時に確認する。server_name・Hostによる仮想ホスト選択は実装せず、listenソケットの共有によるサイト選択も実装しない。HTTP/1.1のHost検証は維持する。

- **接続→どの ServerConfig か**の紐付け(accept元のlistenソケットから引く)。Clientは `const ServerConfig*` またはconst参照を保持し、所有しない(C-42)。mainのconst Configは全Client/EventLoopより長く生存し、起動後に設定を変更しない。

## N3 poll() の厳密な意味 【tasugiya(主)/共, R:L, D:L, ★★★】

**理解すべきこと**

- `struct pollfd { fd; events; revents; }`。`events` は監視したい条件(POLLIN/POLLOUT)、`revents` は結果。**POLLHUP/POLLERR/POLLNVAL は events に指定しなくても revents に出る**。
- 各イベントの意味:
  - `POLLIN`: 読める(データあり、またはEOF、またはlistenソケットでaccept可能)。EOF(recvが0)は POLLIN で通知される。
  - `POLLOUT`: 書ける(送信バッファに空きがある)。**書くデータがないときにPOLLOUTを監視し続けると busy loop になる** → 送信バッファが空でない時だけ監視する(決定事項「毎周再構築」の設計に直結)。
  - `POLLHUP`: 相手が閉じた(ソケットでは相手のclose/RST、パイプでは書き込み側がclose)。**パイプでは POLLHUP と POLLIN が同時に立ち、未読データがまだ残る**ことがある → CGI出力は、POLLHUPでも read して 0 が返るまで読み切る。
  - `POLLERR`, `POLLNVAL`(無効fd)の扱い。

- `poll` のタイムアウト引数(ms)と、**タイムアウト見回り**のための上限設定(最短期限までの時間 or 固定1秒など)。
- `poll` 自体のエラー(`-1`)と **EINTR**(シグナル割り込み)。**errno禁止制約**との関係(→ N5)。SIGINTでのグレースフル終了フラグ確認のためにも、`-1` が返ったらフラグを見てループを継続/終了する。
- 通常ファイルfdは poll で常に「準備完了」が返る(意味がない)→ N10。
- 同等関数(select/epoll)との比較。本チームは poll に決定済み。評価で**poll を選んだ理由とfd数上限の違い**(select は FD_SETSIZE=1024)を説明できる準備。

**資料化する内容**: 「fdの種類(listen/client/CGI stdin/CGI stdout)× 状態 × 監視するイベント」の対応表。毎周の再構築ロジックの仕様の根拠になる。

## N4 ノンブロッキングI/Oとバッファ 【tasugiya(主)/共, R:L, D:L, ★★★】

**理解すべきこと**

- `fcntl(fd, F_SETFL, O_NONBLOCK)`(Linux)。ソケットと、親サーバーが扱うCGI用パイプ端に設定する。CGIのstdin/stdout自体をノンブロッキングにする意味ではない。`accept` で得たソケットにも個別に設定する。
- **部分read/部分write**: `recv` は要求バイト数より少なく返りうる。`send` も**全部は送れない**ことがある → 送信バッファの先頭からのオフセット管理と、POLLOUT での継続送信。
- `recv` の戻り値: `>0` データ、`0` 相手のclose(EOF)、`-1` エラー or 「今は読めない」。
- **合意済み(HTTP-65)**: 通常ソケットは各周にrecv/sendをそれぞれ最大1回・最大8 KiBとする。準備通知・戻り値・部分送信の管理を守り、errnoでEAGAIN等へ分岐しない。
- HTTP-59〜64でClientはHttpRequestを値で保持し、本文上限は生成時に渡す。recv断片をappendDataへ渡し、getState/getErrorCodeで完成/拒否を確認する。要求全体を別バッファへ重複保存しない。HttpResponseはserializeで1回だけ生成しsendBufferへswap、Clientが送信済みオフセットを保持する。APIの正本は [Requirements](../Requirements.md#http-api)。通常ファイルI/O・全体メモリはHTTP-71・72の採用案に従う。
- **確定(C-32)**: HTTP/1.0・HTTP/1.1要求のリクエスト行は末尾CRLF込みで8 KiB(8,192バイト)、その直後のヘッダー部分全体は各行のCRLFと終端の空行込みで32 KiB(32,768バイト)を上限とする。ヘッダー合計にリクエスト行・ボディを含めない。URLデコード等の前の受信バイト数で数え、上限と同じサイズは受理、超過時はそれぞれ414 / 431で拒否する。受信中に検査し、超過が確定したら行末・ヘッダー終端・ボディの到着を待たずエラー応答へ進み、既存方針どおり応答送信完了後に切断する。同じrecvで届いたボディをヘッダーサイズへ加算しない。ヘッダー1行ごとの別上限と個数上限は追加しない。2つの上限はコード内定数とし、Config項目を増やさない。ボディ上限は既存client_max_body_sizeと413、CGI出力上限はCGI-20の規則を維持する。
- C-32のサイズ判定はrysatoの逐次パーサーで要求行・ヘッダー・ボディを区別して行う。tasugiyaのClientは終端まで未検査のデータを蓄積せず、受信断片をパーサーへ渡し、ERRORなら既存のエラー応答経路へ進む。単一recvの固定バッファサイズや全接続合計のメモリ上限とは別の規則である。

**CGIについて確定した処理**

- 全要求を受信・復号してから起動する。入力は加工せずstdinへ渡し、全量を書いたら親側のstdin用パイプを閉じてEOFを渡す。入力ボディ上限には既存の `client_max_body_size` を使う。
- stdinへの書き込みとstdoutの読み取りは同じpollで並行して進める。入力を書き終えるまで出力を読まない設計にはしない。
- stdoutは1件あたりヘッダー8 KiB(8,192バイト)、ヘッダーを含む出力全体8 MiB(8,388,608バイト)を上限として蓄積する。読み取り中に超過を検出し、CGIを打ち切ってfdの後始末・子の回収を行い502とする。
- EOFと子の終了の両方を確認してからHTTP応答を生成する。子が終了しただけで未読出力を捨てず、EOFだけで処理完了にもしない。ブラウザへの逐次転送はしない。
- CGI出力にContent-Lengthがあっても上記の完了条件を使う。本文の実長との照合を行い、不一致は502。指定がなければ実長からHTTP応答の長さを生成する。204等の本文・Content-Lengthを禁止するHTTP規則は維持する。

## N5 errno禁止制約下のエラー処理方針 【tasugiya(主)/共, R:M, D:M, ★★★】

課題要件: **read/write(recv/send)後に errno を見て挙動を変えてはならない**。違反は0点扱い。

**理解すべきこと**

- 従来のノンブロッキングI/Oでは `-1` かつ `EAGAIN/EWOULDBLOCK` を「待てばよい」、`EINTR` を「リトライ」、それ以外を「致命エラー」と区別する。**これを errno で区別できない**。
- **合意済み**: 通常ソケットはHTTP-65〜67、CGIパイプはCGI-112に従う。各周に各方向最大1回とし、recvの負値、送るデータがあるsendの0以下は接続を閉じる。recv=0は受信EOFであり、要求受信中か応答中かで扱いを分ける。poll後でもエラーは起こり得るため戻り値検査を省かない。

- 課題PDFのerrno禁止はread/write後の挙動変更について記されている。CGIのstat/access失敗は、失敗直後のerrnoをCGI-90の404/403/414/500の分類に使う。DELETEのstd::remove失敗はHTTP-48に従い失敗直後のerrnoで分類する。read/write(recv/send)のエラー処理にはこの分類を適用しない。保存失敗後にstd::removeを呼ぶ場合も、書き込み失敗のerrnoではなく削除呼出し自身の結果を使い、元の500を維持する。accept/poll等の具体的な分岐方法は別途確定する。
- `EMFILE`(fd枯渇)時の `accept` 失敗 → listen fd が常にPOLLINのままになるbusy loop化(対策: 接続数上限、失敗時のログ抑制)。

> **未実施の検証**: 「poll→各方向1回」の部分送受信・再通知・EOF/RST・ハングの有無は04で確認する。accept/poll自体の失敗処理はtasugiya側の残る内部設計であり、HTTP 10項目の完了と混同しない。

## N6 クライアント切断・SIGPIPE・shutdown 【tasugiya, R:M, D:S, ★★★】

**合意済み(HTTP-66・67)**: 正本は [Requirementsのネットワーク境界](../Requirements.md#http-network-boundary)。要求受信中の未完了EOFは破棄/close。完成後、または早期拒否の応答中のEOFはPOLLINを外して応答を続ける。通常ソケットのPOLLHUPだけで残りの要求データを捨てず、recv=0まで確認する。POLLERR/POLLNVALやI/O失敗はclose。

- 通常/早期エラーとも送信中の追加受信を読み捨て、全送信後にshutdown(SHUT_WR)。受信EOFまたは送信完了から1秒でcloseする採用案。1秒で応答到達が保証されるとはせず、早期413/417等と実負荷で検証する。
- **SIGPIPE**は親でSIG_IGNとする既決定を維持し、MSG_NOSIGNALの併用を前提にしない。
- CGI待機中の切断・半閉鎖・追加受信と停止/回収はCGI-107〜118を維持する。通常ソケットの規則でCGI-113の切断判定を上書きしない。

## N7 タイムアウト設計 【tasugiya, R:S, D:S, ★★】

**合意済み(HTTP-68〜70)**: 受信はacceptから60秒、送信は最終応答の送信バッファ準備から60秒で、途中のI/Oでは延長しない。未完了の受信期限は受信済みなら408、空接続なら応答なしclose。送信期限はcloseだけで別のHTTP応答を追加しない。全送信後の切断待ちは1秒。CGIは起動から10秒の504を維持する。

- CGIと同じLinux /proc/uptimeの取得方法を通常接続にも使う採用案。各周1回の現在値を共有し、poll待ちは通常接続だけでも最大100 ms。通常接続の計時失敗/逆行はclose、CGIの計時失敗は既存の500と停止/回収に従う。
- 計時元の利用可否・精度・追加I/O負荷とslowlorisへの期限適用は未検証。期限はEventLoopで検出するため、同期ファイル処理でループが止まる問題の解決にはならない。
- clock_gettimeは使用不可。C++98標準ライブラリと一覧外のOS関数を区別する(Requirements §4)。std::removeの採用によって経過時間の取得方法やDate生成の設計を変更したとは扱わず、DateはHTTP-56の暫定案、ログはHTTP-73の[LEVEL] messageとし、経過時間と暦日時を区別する。

## N8 シグナル 【tasugiya, R:M, D:S, ★★】

- `signal()` のみ許可(`sigaction` は許可外)。ハンドラ内で安全にできること(**`volatile sig_atomic_t` フラグ設定のみ**)。
- SIGINT: フラグ方式のグレースフル終了(決定済み)。pollがEINTRで返る → フラグ確認 → cleanup。
- SIGPIPE: 無視(決定済み)。
- SIGCHLD: ハンドラー・SIG_IGNによる自動回収を追加せず、EventLoopがwaitpid(pid, &st, WNOHANG)で回収する(CGI-109・110)。実行中/回収待ちはpoll待ち時間を最大100 msとする(CGI-111)。
- 子はexecve前にSIGPIPEをSIG_DFLへ戻す(CGI-117確定)。親のSIGPIPE無視は維持し、SIGCHLDをSIG_IGNにしない。

## N9 リソース管理・クラッシュ防止 【tasugiya(主)/共, R:M, D:M, ★★★】

- 「どんな状況でもクラッシュ禁止」(メモリ不足含む)への設計: 
  - 例外方針(決定済み): ループ内は戻り値方式 + `catch(std::exception&)` の防波堤、`bad_alloc` もここで受ける。
  - `std::string`/`vector` の無制限成長の防止(ヘッダー/ボディ/CGI出力に上限)。
  - ゼロ除算、範囲外アクセス、NULL参照、`std::string::substr` の例外(`out_of_range`)など、C++98特有のクラッシュ要因の洗い出し。

- fd管理: 上限(`ulimit -n`)、`EMFILE` 対策、CGIへのfd継承はCGI-116で確定済み。Linuxではlisten/acceptソケットと全CGIパイプにF_SETFD/FD_CLOEXECを設定する。
- **CGI同時実行数にアプリケーション独自の上限は設けない**。件数を理由に503を返す分岐や待ち行列も設けない。これはCGI 1件ごとの出力サイズ上限・10秒の時間制限を撤廃する意味ではない。一般の接続数・全体メモリに独自上限を設けない採用案はHTTP-72に従い、資源回収・負荷耐性は検証する。
- CGIの起動・実行失敗は原因を区別する。スクリプトなし404、読み取り不可403、サーバー側のpipe/fork失敗500、exec失敗・子の異常終了・不正出力502。OS資源不足でpipe/forkが失敗した場合も500とし、途中まで作ったfd等を回収する。受け渡し型・APIはCGI-91〜98で確定済み。後始末はCGI-107〜118に従い、エラー対応・適用境界はCGI-128〜134で確定済み。CgiResult失敗時は部分stdoutを使わずmakeError、成功時はparseCgiOutput後にapplyErrorPageへ進む。正常なCGIの400〜599もConfig C-20の本文置換対象とし、最初の失敗コードを保持する。
- CGIの子の起動失敗時はCGI-123の子専用解放経路を使う。mainでcleanupInChildを呼び、複製された管理対象fd・所有メモリとEventLoop本体を解放してreturn 127する。子ではkill/waitpid/shutdownを実行しない。親の停止・回収は明示的なcleanupに置き、デストラクターはfd・メモリ解放だけとする。起動中のCgiProcessもfork前からEventLoopの所有管理へ登録し、一時fdはstartの子側失敗分岐で解放する。親は既存のwaitpidで502を検出・回収する。
- 長時間稼働でのメモリ/fdリーク検証方法(`valgrind`, `lsof`, `/proc/PID/fd`)→ 04。

## N10 通常ファイルI/O 【共, R:S, D:S, ★★】

**合意済み(HTTP-71・72の採用案)**: 課題PDFは通常ディスクファイルのread/writeについてpollの準備通知を不要としている。静的ファイル/error_pageは全読み取り成功後に応答し、アップロードは全検証後に書く。8 KiBずつ処理するがRouterの1回の呼出しで完了まで進める同期処理であり、他接続へ毎回処理を譲る設計ではない。専用の静的ファイルサイズ上限を追加しない。

- pipe/FIFO/ソケットは免除対象外で、必ず単一pollの準備通知とノンブロッキングI/Oを使う。
- [評価項目](https://www.42evalhub.com/common/webserv)では通常ファイル処理がイベントループを停滞させないことも確認する。通常ファイルをpollへ登録しても、同期のディスクI/Oを非同期にはできない。
- 大きい/遅いファイルの処理と小さいGET・CGIを並行させる試験、メモリ量・回収・空ページGETのSiegeを04で確認する。基準を満たさない場合はこの採用案を見直し、必要ならHTTP-63の結果や内部状態も合わせて変更する。現時点で合格を実証したとは扱わない。

---

## この章の作業量サマリ

| 区分 | 量 | コメント |
|---|---|---|
| N1–N2 | M | man を読めば足りる。短時間で資料化可 |
| N3–N5 | XL | 仕様化で最も判断が多い。**errno制約の下での設計**はtasugiyaの最重要資料 |
| N6–N9 | M〜L | 小項目が多い。表にまとめやすい |
| 未解決/未検証事項 | S〜 | 接続情報取得失敗時の扱い、計時/一括ファイル処理の実環境検証。accept/pollの失敗処理はN5 |
