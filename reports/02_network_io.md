# 02 ネットワークとI/O

tasugiyaは、接続の受付、各Clientの受信・送信、単一のpoll()で入出力を進めるEventLoopを担当します。Clientは受信したデータをHTTP側へ渡し、応答のヘッダーと本文を順に送ります。[EventLoopの処理順](../Requirements.md#event-loop-timeout-order)、[分離送信の設計](../Requirements.md#split-response-send)

実装前に、poll()の通知を受けてからソケットやCGIパイプを読み書きする条件と、read・write・recv・sendの後にerrnoで分岐しない規則を確認してください。通常ファイルの扱い、切断時の回収、各処理の期限も、この入出力の設計に含まれます。[I/Oの必須要件と設計](../Requirements.md#http-network-boundary)

調査にはsocket・accept・poll・fcntlなどのmanページを使います。

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
- **確定(C-28)**: tasugiyaはacceptした接続に対するgetsocknameでサーバー側IP・ポートを取得し、rysatoのHTTP/Router側へ渡す。HTTP/1.0でHostがない場合の末尾/補完301に使う。listenの0.0.0.0やacceptで得たクライアント側アドレスを代用しない。有効なHostがある場合はその明示ポートごとURLに使い、Hostにポートがなければ接続のポートを自動追加しない。ConnectionInfoの受け渡しはCGI-93で合意済み。[接続先情報の取得失敗時の契約](../Requirements.md#getsockname-failure)に従う。
- **取得失敗時**: accept直後のgetsocknameが失敗したら、EventLoopが新規接続のfdを一度だけ閉じる。Client生成・poll登録・HTTP応答生成は行わず、既存接続の処理を続ける。
- **確定(C-30)**: Hostの受付検証はrysatoがHTTP層で行う。Host値のためのDNS問い合わせはせず、接続先やServerConfigも切り替えない。Hostの名前/任意ポートの受付規則は01・05のC-30を参照する。
- **確定(CGI-37)**: HTTP/1.0でHostがないCGI要求のSERVER_NAMEには、C-28と同じ接続のサーバー側IPをgetsocknameで取得して使う。listenの0.0.0.0や接続元IPで代用しない。接続情報はCGI-93のConnectionInfoで渡す。[接続先情報の取得失敗時の契約](../Requirements.md#getsockname-failure)に従う。
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
  - 例外方針: 通常の失敗は戻り値方式、例外は接続処理側で捕捉する。std::bad_alloc時は当該接続のfd・保持本文・送信バッファ等を回収してcloseし、追加確保を伴う500の再生成や復旧は行わない。ほかの接続を継続し、関連CGIは既存の停止/回収へ進める。具体的な捕捉位置と回収経路は実装設計で確認する。
  - `std::string`/`vector` の無制限成長の防止(ヘッダー/ボディ/CGI出力に上限)。
  - ゼロ除算、範囲外アクセス、NULL参照、`std::string::substr` の例外(`out_of_range`)など、C++98特有のクラッシュ要因の洗い出し。

- fd管理: 上限(`ulimit -n`)、`EMFILE` 対策、CGIへのfd継承はCGI-116で確定済み。Linuxではlisten/acceptソケットと全CGIパイプにF_SETFD/FD_CLOEXECを設定する。
- **CGI同時実行数にアプリケーション独自の上限は設けない**。件数を理由に503を返す分岐や待ち行列も設けない。これはCGI 1件ごとの出力サイズ上限・10秒の時間制限を撤廃する意味ではない。一般の接続数・全体メモリに独自上限を設けない採用案はHTTP-72に従い、資源回収・負荷耐性は検証する。
- CGIの起動・実行失敗は原因を区別する。スクリプトなし404、読み取り不可403、サーバー側のpipe/fork失敗500、exec失敗・子の異常終了・不正出力502。OS資源不足でpipe/forkが失敗した場合も500とし、途中まで作ったfd等を回収する。受け渡し型・APIはCGI-91〜98で確定済み。後始末はCGI-107〜118に従い、エラー対応・適用境界はCGI-128〜134で確定済み。CgiResult失敗時は部分stdoutを使わずmakeError、成功時はparseCgiOutput後にClient側の共通送信準備へ進む。旧applyErrorPage/readErrorPageは廃止し、HttpResponseはファイル読み取りを行わない。正常なCGIの400〜599もConfig C-20の本文置換対象とし、最初の失敗コードを保持する。
- CGIの子の起動失敗時はCGI-123の子専用解放経路を使う。mainでcleanupInChildを呼び、複製された管理対象fd・所有メモリとEventLoop本体を解放してreturn 127する。子ではkill/waitpid/shutdownを実行しない。親の停止・回収は明示的なcleanupに置き、デストラクターはfd・メモリ解放だけとする。起動中のCgiProcessもfork前からEventLoopの所有管理へ登録し、一時fdはstartの子側失敗分岐で解放する。親は既存のwaitpidで502を検出・回収する。
- 長時間稼働でのメモリ/fdリーク検証方法(`valgrind`, `lsof`, `/proc/PID/fd`)→ 04。

## N10 通常ファイルI/O 【共, R:S, D:S, ★★】

2026-11-17合意: 静的ファイル/error_pageの読み取りとアップロード保存は固定期限方式を使い、read/writeが進行しても期限を延長しない。Clientが各タスクのstart直前に開始時刻を記録し、開始準備/openを計測区間に含める。2026-11-18合意でerror_pageは自身のstart直前から新しい固定期限を設ける。error_pageの期限超過ではcancel後、元のステータス等を保持してfinishErrorPage(false, body)で内蔵HTMLを確定し、beginWritingResponseへ進む。2026-11-18に静的読み取り/保存/error_pageを各60秒の採用案とし、並行負荷試験で検証する。静的読み取り/保存の期限超過はClientがcancelでfd/部分本文/今回作成した途中ファイルを回収し、makeError(500)をprepareResponseへ渡す。error_pageは新しい60秒を使う。step前に期限確認し、超過なら呼び出さず用途別の回収/応答へ進む。step復帰後はCOMPLETEDを優先して成功経路、FAILEDは失敗経路、IN_PROGRESSなら時刻を再確認し超過時は回収/応答へ進む。start成功直後にも期限を確認し、超過ならcancel後に用途別の超過処理へ進む。start失敗は静的読み取り500、保存の返却コード、error_pageの内蔵HTMLという既存の失敗経路を使う。2026-11-18合意で進行可能なファイルタスクがある間はpoll待ち時間0とし、各周でイベント/期限を確認し各タスクを最大1回進める。タスクがなくなったら通常のpoll待ち時間へ戻す。イベント0件でもファイルタスクを進める。poll復帰後はソケット/CGIイベントを先に処理し、その後、生存中で継続する接続のファイルタスクを期限確認付きで進める。新規タスクは開始した周にstart/直後の期限確認を行い、最初のstepは次の周からとする。失敗からerror_pageへ移る場合や同じFileReadTaskの再利用も同じ契約に従う。終了確定時はClientをCLOSINGにしてタスクcancel/fd回収/CGI関連付け解除を行い、本体のdeleteとコンテナ除去はその周の最後にEventLoopが行う。後段はCLOSINGを対象から外す。2026-11-18合意でファイル処理の時刻取得/解析失敗や逆行はタスクcancel後CLOSINGへ進め、fd回収/CGI関連付け解除から周末削除へつなぐ。追加の500/error_page/内蔵HTMLを生成せず、保存成功済みファイルを保護する。2026-11-19合意でClientにfileTaskStartedThisRound_を持たせ、周頭にfalse、新規タスク開始時にtrueとする。後段のstepはtrue/CLOSINGを対象から外し、次周から進める。同じタスクのerror_page用再開始もtrueとする。2026-11-19合意で周頭リセット/poll/全体期限確認/ソケットとCGIイベント/ファイル処理/周末削除の順とする。期限処理で無効になったfdの古い通知を除外し、新規タスクは次周からstepする。操作APIと例外経路の詳細は未決定。[期限方式の正本](../Requirements.md#file-task-timeout-policy)を参照する。

**2026-11-15合意による変更(HTTP-63・71)**: 静的ファイル/error_pageはClientが所有するFileReadTaskで読み、各周に各タスクのreadを最大1回・最大8 KiBだけ進め、未完了ならEventLoopへ戻す。全量の読み取り・close成功後に応答を生成する。Routerは対象の解決・権限確認までを担当し、静的ファイルはFILE_READ_REQUIREDとFileReadRequestを返す。FileReadTaskがopen/read/closeと途中の本文を所有する。詳細の正本は[Requirementsの設計修正](../Requirements.md#file-read-design)、共有宣言は[03](03_config_cgi_upload.md#cgi-api)。

ClientはFileReadTaskを値メンバーfileTask_として持ち、両クラスをコピー禁止とする。未開始時はfd=-1・空本文、タスク自体のnew/deleteは不要。EventLoopはClientへのポインターを管理する。Clientの読み取り中の状態はREADING_FILEとし、静的ファイルは未完了なら同状態を維持、完了なら200応答の送信準備後にWRITING_RESPONSE、失敗なら部分本文を破棄して500生成へ進むことを合意した。[基本遷移の正本](../Requirements.md#client-file-read-state)を参照する。FileReadTaskはstart/step/takeBody/cancelの4操作に分け、stepは各回最大8 KiBのreadを1回、EOFとclose成功で完了、takeBodyは本文をswapで渡し、cancelは保持fdと途中の本文を解放する。[4操作の正本](../Requirements.md#file-read-task-operations)を参照する。startはbool start(const FileReadRequest&)とし、trueはファイルを開いて読み取り開始の準備ができたこと、falseは開始失敗を表す。HTTPコードへの変換は呼び出し側が行う。stepは引数なしのStepResult step()とし、IN_PROGRESS/COMPLETED/FAILEDを返す。短い正のreadをEOF扱いせず、readの0とclose成功でCOMPLETEDとする。takeBodyはbool takeBody(std::string&)とし、完了後の初回だけswapで本文を渡してtrue、未開始/未完了/失敗/取得済みなら引数を変更せずfalseとする。正常な空本文もtrue。取得成功時に出力引数の以前の内容がタスク側へ移ったら解放する。cancelはvoid cancel()とし、保持fdを-1にしてからcloseを1回だけ呼び、失敗でも再試行しない。本文を解放して取得不可とし、どの状態でも再呼び出し可能で二重closeしない。デストラクターも同じ解放処理を使い例外を外へ出さない。HTTP応答は生成しない。startは未開始時だけ開始でき、読み取り中/完了/失敗/本文取得済みならfalseで現在の内容を変えない。再利用時はcancelで未開始へ戻してからstartする。stepは読み取り中だけI/Oを行い、完了かつ本文未取得ならCOMPLETED、失敗ならFAILED、未開始/本文取得済みなら状態不変でFAILEDを返す。終端後の再呼び出しでread/closeを繰り返さない。内部状態はprivateなPhase(IDLE/READING/DONE/ERROR/TAKEN)で持ち、完了/失敗/取得済みのboolを重ねて持たない。初期状態とcancel後はIDLE、start成功でREADING、EOFとclose成功でDONE、本文取得成功でTAKENとする。IDLEからの通常の開始失敗は、取得済みfdを一度だけ閉じ本文/保持情報を解放し、ERRORでfalseを返す。makeError(code)は設定を受け取らずコード/標準説明文の元応答だけを作り、本文生成/読み取りを行わない。生成したエラーと検証済みCGI応答は共通のerror_page準備経路へ渡す。error_pageはClientがHttpResponse pendingResponse_を値メンバーとして持ち元の応答を保持し、rysato側のprepareErrorPageが読み取り要否と指示を返す方針を合意済み。[準備APIの正本](../Requirements.md#error-page-prepare)を参照する。読み取り期限・異常時の詳細な回収経路、poll待ち時間、保持fdのCGI継承防止、error_pageの異常時の詳細遷移は未決定。共通の入口はClient::prepareResponse(const HttpResponse&)とし、pendingResponse_へコピー後にprepareErrorPageを1回呼ぶ。読み取り不要ならserializeしてWRITING_RESPONSE、必要ならERROR_PAGEで読み取りを開始してREADING_FILEへ進む。prepareErrorPageはClient側の共通送信準備で1回だけ呼ぶ。Router/CGIパーサーは個別に準備せず、finishErrorPage後は再準備せずserialize/送信へ進む。error_pageの本文確定はfinishErrorPage(bool readSucceeded, const std::string& body)とし、trueなら空も含む取得本文、falseなら元コードの内蔵HTMLを採用する。Clientは保持中の応答へ適用後、privateなbeginWritingResponseでserialize・送信バッファへのswap・送信オフセット0・送信期限開始を行い、WRITING_RESPONSEへ進む。prepareErrorPageがfalseの場合も同じ関数を使う。ここでsendやerror_page再準備は行わず、sendはpoll通知後とする。用途はClientのprivateなFileReadPurpose(STATIC_FILE/ERROR_PAGE)で区別し、READING_FILE中だけ参照する。STATIC_FILEの成功は200応答、失敗は500応答を共通送信準備へ渡す。ERROR_PAGEは成功/失敗をfinishErrorPageへ渡し再準備しない。FileReadTaskへ用途やHTTP応答の判断を持たせない。アップロード保存は2026-11-16合意でClient所有の値メンバーFileWriteTask（2026-11-17合意でコピー禁止。C++98のprivateなコピーコンストラクター/代入演算子を宣言し定義しない）へ分け、RouterがFILE_WRITE_REQUIREDとFileWriteRequestを返す。全体検証後に新規作成し、各周のwrite最大1回・最大8 KiB、全量保存とclose成功後に200とする。保存本文はFileWriteRequestのbodyからタスクへswapで移し、指示側bodyを空にする。タスクは移した本文を所有し、指示側本文への参照を残さない。元のHttpRequest本文は2026-11-17合意で保存タスクのstart成功後にClientが解放する。本文以外の要求情報は保持し、以後の保存はタスク所有の本文を使う。解放にはpublicなvoid HttpRequest::releaseBody()を採用する。COMPLETE後にClientが呼び、本文を空にして保持領域を解放し、本文以外の情報を維持する。COMPLETE後の繰り返し呼び出しも可能とする。INCOMPLETE/ERRORで呼んだ場合は全保持データと状態を維持して戻る。開始APIはint start(FileWriteRequest&)とし、パス/保存名の保持とopen成功後にだけ本文をswapする。0は開始準備完了、通常の開始失敗はHTTP-41のコードを返し、指示側本文を変更しない。進行APIはStepResult step()とし、IN_PROGRESS/COMPLETED/FAILEDを返す。各回write最大1回・最大8 KiB、全量保存とclose成功でCOMPLETED、writeの0/負値またはclose失敗でFAILEDとする。空本文はwriteせずclose成功で完了する。2026-11-17合意で、全量保存とclose成功をDONEとして確定し、本文/保存先パスを解放してからCOMPLETEDを返す。保存名だけをtakeFilenameまで保持する。結果取得はbool takeFilename(std::string&)とし、保存完了後の初回だけswapで保存名を渡してtrue、それ以外は出力不変でfalseとする。cancelはvoid cancel()を採用し、保存途中ならfdを閉じ今回作成した途中ファイルを削除、全量保存とclose成功後のファイルは残す。保持データを解放して未開始へ戻し、繰り返しても二重close/削除しない。デストラクターも同じ解放を行い例外を外へ出さない。内部状態はprivateなPhase(IDLE/WRITING/DONE/ERROR/TAKEN)で管理し、完了/失敗/取得済みのboolを重ねない。初期状態とcancel後はIDLE、start成功でWRITING、全量保存とclose成功でDONE、保存名取得でTAKEN、通常の開始/保存失敗でERRORとする。startはIDLEで開始し、それ以外は500でタスク/指示/ファイルを変更しない。再利用前にcancelでIDLEへ戻す。IDLEからの通常の開始失敗は、取得済みfdを一度だけ閉じ、タスクのパス/保存名/本文を解放し書き込み位置0・fd=-1でERRORへ進む。入力指示を維持し、後始末に必要なパスは削除試行/ログ記録後に解放する。stepはWRITINGで保存を進め、DONEはI/OせずCOMPLETED、ERROR/IDLE/TAKENはI/OせずFAILEDを返す。WRITING以外では状態/保持データを維持し、write/close/削除を繰り返さない。ClientのWRITING_FILEと基本遷移は2026-11-17に採用済み。開始成功で保存中へ、未完了は維持、完了は保存名取得/makeUploadSuccessからprepareResponseへ、開始失敗は返却コード、保存失敗ではstep内でfdのclose、今回作成した途中ファイルの削除試行/失敗ログ、本文/パス/保存名の解放と位置0への復帰を済ませ、ERRORでFAILEDを返す。Clientは500をprepareResponseへ渡す。再step/cancel/破棄で同じ資源をclose/削除しない。期限・異常時の詳細遷移・進行順序は未決定。poll待ち時間0の方針は合意済み。FileWriteTaskは保存結果と保存名だけを渡し、HTTP応答を生成しない。成功確認後にrysato側のHttpResponse::makeUploadSuccess(const std::string& filename)が200と保存名の結果HTMLを作り、ClientがprepareResponseへ渡す。成功応答の生成APIは採用済み。タスクの保存名取得APIも採用済み。[保存設計の正本](../Requirements.md#file-write-design)を参照する。autoindex生成は2026-11-19に列挙/ソート/HTML生成の段階分割を採用した。途中状態はClient所有の値メンバーAutoindexTaskへまとめる。ディレクトリ/項目一覧/ソート状態/生成途中のHTMLをタスクが所有する。各周の処理量とAPI/期限/回収は次に具体化する。serializeの停滞対策も残る。静的ファイル専用サイズ上限を追加する合意ではない。

- pipe/FIFO/ソケットは免除対象外で、必ず単一pollの準備通知とノンブロッキングI/Oを使う。
- [評価項目](https://www.42evalhub.com/common/webserv)では通常ファイル処理がイベントループを停滞させないことも確認する。通常ファイルをpollへ登録しても、同期のディスクI/Oを非同期にはできない。
- 各readやopen/stat/close自体のディスク待ちは残り、分割実行だけで完全なノンブロッキングとはならない。大きい/遅いファイルの処理と小さいGET・CGIを並行させる試験、メモリ量・回収・空ページGETのSiegeを04で確認する。基準を満たさない場合は構成を見直す。現時点で合格を実証したとは扱わない。

送信準備中のstd::bad_allocは対象接続の資源回収とcloseへ進め、ほかの接続の処理を継続する。

---

## この章の作業量サマリ

| 区分 | 量 | コメント |
|---|---|---|
| N1–N2 | M | man を読めば足りる。短時間で資料化可 |
| N3–N5 | XL | 仕様化で最も判断が多い。**errno制約の下での設計**はtasugiyaの最重要資料 |
| N6–N9 | M〜L | 小項目が多い。表にまとめやすい |
| 未解決/未検証事項 | S〜 | 接続情報取得失敗時のfd回収、計時/ファイル処理の実環境検証。accept/pollの失敗処理はN5 |

2026-11-18合意: FileReadTask/FileWriteTaskはopen直後にF_SETFD/FD_CLOEXECを設定し、設定成功後にstart成功を返す。保存本文のswapも設定成功後とする。設定失敗はfdを一度だけ閉じ、保存で今回作成した途中ファイルを削除し、指示側本文を維持して500、読み取りはfalseを返す。CGI子の起動失敗では複製fd/メモリを解放し、親が保存中のファイルを子から削除しない。2026-11-19にpublicなvoid FileWriteTask::cleanupInChild()を採用した。複製fdを一度だけ閉じてfd=-1、保持データを解放し位置0/IDLEへ戻す。途中ファイル削除責任を外し、以後のcancel/破棄でも親のファイルを削除しない。再呼び出し可能で追加確保/ログ/外へ出る例外を避ける。2026-11-19にpublicなvoid Client::cleanupInChild()を採用し、EventLoopの子専用経路がClient破棄前に呼ぶ。Client内で読み取りcancel/保存cleanupInChild/複製ソケットfdのcloseを行い、残る所有メモリを続く破棄までに回収する。再呼び出しと破棄で二重close/削除を防ぐ。[継承防止の正本](../Requirements.md#file-task-cloexec)を参照する。

2026-11-19合意: autoindexの受け渡しはAUTOINDEX_REQUIREDとAutoindexRequest(path/urlPath)を追加し、route/handleGetの末尾へ出力引数を設ける。ClientがAutoindexTaskを開始し、タスクが必要情報を値で保持する。[一覧生成待ちの正本](../Requirements.md#autoindex-route-api)を参照する。

2026-11-19合意: AutoindexTaskの開始はint start(const AutoindexRequest&)とし、path/urlPathを値で保持してopendirする。0は開始準備成功、失敗はHTTP-31の404/403/414/500。DIR*はタスクが所有し、列挙/ソート/HTML生成は後続のstepで進める。[開始APIの正本](../Requirements.md#autoindex-start-api)を参照する。

2026-11-19合意: AutoindexTaskはStepResult step()でIN_PROGRESS/COMPLETED/FAILEDを返す。列挙/ソート/HTML生成の段階を内部で管理し、一定量でEventLoopへ戻る。全工程とclosedir成功で完了し、HTTP-31/32の通常の生成失敗はFAILEDから500へ進む。処理量/状態/結果取得/回収/期限は次に具体化する。[進行APIの正本](../Requirements.md#autoindex-step-api)を参照する。

2026-11-19合意: AutoindexTaskの完成HTMLはbool takeBody(std::string&)で取得する。完了後の初回だけswapで渡してtrue、未開始/生成途中/失敗/取得済みでは出力不変でfalse。Client側は200/text/html応答を組み立てる。ClientはtakeBodyで取得した完成HTMLをHttpResponse::setBodyBySwap(body, "text/html")で応答へ移し、prepareResponseBySwapへ渡す。応答全体をpendingResponse_とswapし、共通送信準備へ進む。[応答の所有権移譲](../Requirements.md#response-ownership-handoff)を参照する。引き渡しに成功するとbodyは空になる。[本文の引き渡し](../Requirements.md#autoindex-response-body-swap)を参照する。[本文取得APIの正本](../Requirements.md#autoindex-take-body)を参照する。

2026-11-19合意: AutoindexTaskはreaddirの正常終端直後にDIR*を所有状態から外してclosedirを一度だけ呼び、成功後にソートへ進む。ソート/HTML生成中はディレクトリを開いたまま保持しない。closedir失敗は一覧を破棄してFAILEDから500へ進め、再closeを防ぐ。[解放時期の正本](../Requirements.md#autoindex-directory-close)を参照する。

2026-11-19採用案: AutoindexTaskの列挙は1回のstepでreaddir最大32回とする。.と../終端/失敗を含めて呼び出し回数を数え、終端/失敗/HTML上限超過で打ち切る。終端未確認なら途中状態を保持してIN_PROGRESSで戻る。並行負荷で回数を検証する。[列挙量の正本](../Requirements.md#autoindex-enumeration-budget)を参照する。

2026-11-19合意: AutoindexTaskのstepは一つの段階を進め、完了時は次段階の開始状態としてIN_PROGRESSで戻る。列挙終端/closedir成功後のソート、ソート完了後のHTML生成は次周から行う。空/1項目も同じ順序とし、HTML生成完了でCOMPLETEDとなる。[段階切り替えの正本](../Requirements.md#autoindex-stage-transition)を参照する。

2026-11-19合意: AutoindexTaskのソートはボトムアップのマージソートを使い、区間幅/範囲/左右と出力の位置を保持して比較と要素移動を一定量ずつ進める。残り要素の移動と作業領域の準備/初期化も分割設計へ含める。ソート対象は列挙順の名前一覧を参照する番号列とし、比較は元の名前、移動は番号、HTML生成は整列済み番号からの名前参照で行う。番号列の準備も分割し、ソート段階の各stepは比較/番号書き込み合計256操作の採用案とし、準備/初期化/残り番号の移動も含める。コンテナ/領域の準備方法は次に決める。[ソート方式の正本](../Requirements.md#autoindex-merge-sort)を参照する。

2026-11-19採用案: AutoindexTaskのHTML生成は各stepの実追加最大8 KiBとし、固定HTML/表示名/hrefの変換後バイト数を数える。項目や変換の途中位置を保持してIN_PROGRESSで戻り、全体完成でCOMPLETEDとする。完成HTML8 MiB上限も維持し、並行負荷で採用値を検証する。[HTML生成量の正本](../Requirements.md#autoindex-html-budget)を参照する。

2026-11-19合意: AutoindexTaskの回収はvoid cancel()とし、保持DIR*をNULLにしてclosedirを一度だけ行う。名前/番号列/作業領域/HTML/パスとURL/位置を解放または初期化して未開始へ戻し、再呼び出しと破棄で二重closedirを防ぐ。cancel後はtakeBodyが出力不変でfalseを返す。デストラクターも同じ回収を行い例外を外へ出さない。[回収APIの正本](../Requirements.md#autoindex-cancel-api)を参照する。

2026-11-19合意: AutoindexTaskはprivateなPhase(IDLE/ENUMERATING/SORTING/BUILDING/DONE/ERROR/TAKEN)で生成段階と終端状態を管理する。初期/cancel後はIDLE、start成功から列挙/ソート/HTML生成へ進み、全工程成功でDONE、本文取得でTAKEN、通常の失敗でERRORとする。別の状態列挙型や完了/失敗/取得済みboolを重ねない。[内部状態の正本](../Requirements.md#autoindex-phase)を参照する。

2026-11-19合意: AutoindexTask::startはIDLEで開始し、他のPhaseでは500を返してDIR*/保持データ/入力指示/状態を維持する。再呼び出し拒否でERRORへ遷移せず、再利用はcancelでIDLEへ戻してから行う。[開始の状態別契約](../Requirements.md#autoindex-start-state)を参照する。

2026-11-19合意: AutoindexTask::stepはENUMERATING/SORTING/BUILDINGで該当段階を進め、DONEは処理せずCOMPLETED、ERROR/IDLE/TAKENは処理せずFAILEDを返す。生成中以外では状態/本文/保持データを維持し、列挙/ソート/生成/closeを繰り返さない。[進行の状態別契約](../Requirements.md#autoindex-step-state)を参照する。

2026-11-19合意: AutoindexTaskの通常の生成失敗は、step内で保持DIR*をNULLにして一度だけclosedirし、名前/番号列/作業領域/HTML/パスとURL/継続位置を解放または初期化してERRORでFAILEDを返す。Clientは500を共通送信準備へ渡す。close失敗時も再試行せず、再step/cancel/破棄で二重回収しない。[失敗時回収の正本](../Requirements.md#autoindex-step-failure)を参照する。

2026-11-20合意: AutoindexTaskのIDLEからの通常の開始失敗は、取得済みDIR*をNULLにして一度だけcloseし、保持情報/名前/番号列/作業領域/本文を解放して継続位置を初期化し、ERRORで元の開始失敗コードを返す。入力指示を維持し、再利用はcancelでIDLEへ戻してから行う。[開始失敗時の正本](../Requirements.md#autoindex-start-failure)を参照する。

2026-11-20合意: AutoindexTaskはC++98のprivateな未定義コピーコンストラクター/代入演算子でコピー禁止とする。Clientの値メンバーとして所有し、DIR*の所有者を一つに保つ。Clientのコピー禁止とEventLoopのポインター管理に従う。[コピー禁止の正本](../Requirements.md#autoindex-noncopyable)を参照する。

2026-11-20合意: ClientのStateにGENERATING_AUTOINDEXを追加する。AutoindexTask開始成功で同状態へ、IN_PROGRESSは維持、COMPLETEDはtakeBodyで本文取得後200/text/html応答をprepareResponseへ渡す。開始失敗は返却コード、生成失敗は500を共通送信準備へ渡す。列挙/ソート/HTML生成の段階はタスク内で管理し、Routerを再実行しない。[Client状態の正本](../Requirements.md#client-autoindex-state)を参照する。

2026-11-20合意: AutoindexTaskにも共通の進行/期限規則を適用する。start直前から全段階通算60秒で延長せず、今周開始フラグで最初のstepを次周へ回し、各周最大1回/poll待ち0とする。start/step前後の確認とCOMPLETED優先を適用し、超過はcancel後500、計時失敗はcancel後CLOSINGへ進める。error_pageは新しい60秒とする。[期限と進行の正本](../Requirements.md#autoindex-loop-timeout)を参照する。

2026-11-20合意: AutoindexTaskはHTML完成時に完成本文を保持し、名前/番号列/マージ領域/パスとURL/不要な継続情報を解放または初期化してDONEへ進み、COMPLETEDを返す。本文はtakeBodyまで保持し、解放済み情報を使って再生成しない。完了時解放の時間とメモリも並行負荷で測定する。[完成時解放の正本](../Requirements.md#autoindex-success-release)を参照する。

2026-11-20合意: AutoindexTaskの名前一覧はstd::deque<std::string>とし、採用した名前を所有して列挙順に末尾追加する。番号列から対応位置の名前を参照して比較/HTML生成し、名前のソート用複製を行わない。領域確保/管理領域拡張/名前コピーと解放も負荷測定する。[名前一覧の正本](../Requirements.md#autoindex-name-container)を参照する。

2026-11-21合意で、番号列とマージ用作業領域はstd::vector<std::size_t>を1本ずつ使う。列挙とclosedirの成功後に空の2本へ項目数分をreserveし、マージ元の初期番号と最初のマージの出力番号をpush_backで追加する。追加は1個につき1操作とし、1回のstepあたり合計256操作に含める。各パスの末尾で2本をswapし、2パス目以降は出力位置をsize()と別に管理して既存要素へ上書きする。上書きも1件につき1操作と数える。[番号列と作業領域の設計](../Requirements.md#autoindex-index-containers)を参照する。

2026-11-21合意で、共通送信準備ではステータス行とヘッダーだけを生成し、保持済みの本文を別に送る。POLLOUT通知後に既存のsend上限を守り、ヘッダーを送り切ってから本文へ進む。送信期限は準備完了から通算60秒とする。2026-11-23合意で、送信中はClientのpendingResponse_に確定済み本文を保持し、そこから直接送る。送信完了または打ち切り後に本文と送信ヘッダーを回収する。HttpResponseのserializeHeaders()でステータス行・ヘッダー・終端の空行を生成し、getBody()で送信対象の本文をconst参照で取得する。本文の要否とContent-LengthはHttpResponseが決める。[送信用API](../Requirements.md#response-send-accessors)を参照する。
