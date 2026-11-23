# 04 試験項目と進行計画

まず単体試験でHTTPの解析と応答を確認し、ClientとRouterをつないだ後に静的配信、アップロード、CGIを試します。結合後は、1件の要求が正しく処理されることに加え、別の接続が待たされないことや、失敗後にfdやメモリを回収できることを確かめます。

最初に試す入力は[実装の始め方](06_implementation_guide.md)、検証結果に応じて見直す項目は[設計の残件](#remaining-design)を参照してください。

---

## T1 テスト手法・ツール 【共, R:M, D:M, ★★】

課題は「telnetとNGINXで事前に試す」「複数のプログラム/言語でテストを書く」「ストレステストに耐える」ことを要求している。

| 目的 | 道具 | 理解しておくこと |
|---|---|---|
| 生のHTTPを手で送る | `telnet`, `nc`(netcat) | CRLFの送信(`printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\n' \| nc`)。断片送信(途中で止める)で逐次パースを検証 |
| 標準的クライアント | `curl -v`, `--http1.0/1.1`, `-X DELETE`, `-F file=@x`(multipart), `--data-binary`, `-H "Transfer-Encoding: chunked"`, `--max-time` | `Expect: 100-continue` の自動付与(H9)、`-i/-I`(HEAD)、`--resolve` |
| ブラウザ | Chrome/Firefox 開発者ツール(Network) | 並列接続、空の投機接続、favicon.ico要求、キャッシュ挙動 |
| 負荷 | `siege`, `ab`(ApacheBench), `wrk` など | 同時接続数、`ulimit -n` 影響、完走時の fd/メモリ確認 |
| 異常系 | 自作Pythonスクリプト(`socket`) | 不正リクエスト、巨大ヘッダー/ボディ、途中切断、1バイトずつ送信(slowloris)、大量接続して放置、RST送信 |
| リーク確認 | `valgrind`, `lsof -p`, `ls /proc/PID/fd` | 切断/タイムアウト/CGI失敗後のfdとメモリ |
| 比較基準 | `nginx`(ローカルに同等設定) | ステータス・ヘッダー・エラーページの挙動比較(T2) |
| 公式テスター | 課題提供の小型tester | 任意だがデバッグ補助。入手先と実行方法を確認 |

**資料化する内容**: 各トピック仕様に付ける受け入れテスト(コマンド+期待応答)を共通のフォーマット(`## テスト` セクション)で記述するルール。

[42EvalHubの評価項目](https://www.42evalhub.com/common/webserv)(2026-10-07確認)では、空ページへのGETでSiege -bのAvailabilityが99.5%を超えること、長時間の継続でメモリが増え続けないこと、ハングする接続がないこと、通常ファイルの同期I/Oがイベントループを停滞させないことを確認する。負荷条件は評価者と合わせ、実行条件・結果を記録する。設計判断の合意は試験合格を意味しない。

### 確定事項から作る受け入れテスト

以下は [Requirements.md](../Requirements.md) と [03のCGI合意事項](03_config_cgi_upload.md) に合わせた検証計画。テストを実行済みという意味ではない。

| 対象 | 確認する動作 |
|---|---|
| HTTPの接続とバージョン | HTTP/1.0・必要なHTTP/1.1要求にHTTP/1.0で応答し、1接続1要求でcloseする。HTTP/1.1 chunked入力は復号する。HTTP/1.0のTransfer-Encoding、およびTransfer-EncodingとContent-Lengthの併記は400 |
| HTTP要求行の書式 | HTTP-01: SP各1個の正常行を受理。複数SP・タブ・行頭/行末空白・要素欠落・先行空行は400。CRLF断片と8 KiB境界はC-31・32を維持 |
| HTTPメソッド | HTTP-02: GET/POST/DELETEを受理し、正常な要求行のHEAD/PUT/UNKNOWN/getは501。メソッド内の不正文字は400。対応メソッドのlocation禁止は405/Allow |
| HTTP要求先の形式 | HTTP-03: /・/a・/a?q=xを受理。GETで絶対URL・authority-form・*・相対パスを送った場合は400。連続/は既存の正規化へ進む |
| HTTP要求先の文字 | HTTP-05: パス/queryの許可文字と大文字/小文字%XXを受理。生の空白・制御文字・DEL・非ASCII・#・許可外記号・不正な%XXは400。queryの%00は未復号で保持、パスの%00は既存の400。%252eの二重デコードなし |
| HTTP版の検査 | HTTP-04: HTTP/1.0・1.1を受理。1.2/1.9では1.1のHost/TE/Expect規則を適用し、getVersionとCGI SERVER_PROTOCOLは受信表記を保持。http/1.1・HTTP/1・HTTP/1.10は400、HTTP/2.0等の未対応メジャーの表記は505。応答はHTTP/1.0 |
| HTTP Content-Lengthの数値 | HTTP-06: 0・0012を受理し、空・+1・-1・1.0・途中のSP/HTAB・非ASCII数字・桁あふれは400。正常な値の上限超過は既存の413 |
| HTTP Content-Typeの重複 | HTTP-07: 通常要求でも同値・大小文字違い・空値を含む重複は400。単一値は保持し、共通処理でMIME全体を検証しない。CGIの空値/欠落はCGI-80を維持 |
| HTTP Transfer-Encodingの値 | HTTP-08・09: 同値・空値を含む複数行は400。chunked/Chunkedと空要素除去後の単独chunkedを受理。gzip, chunkedは501、gzip・chunked, gzip・chunked, chunked・chunked;foo=bar・空/不正一覧は400。1.0のTEとCL併記は既存400 |
| HTTP Connectionの共通検証 | HTTP-10: 通常要求もtoken検証。不正な非空要素は400、空要素無視、複数行を処理。keep-aliveを要求しても1接続1要求。専用のHost/本文処理を変更せず、CGIの転送除外は維持 |
| HTTP本文の完成と断片 | HTTP-12〜14: CLちょうどN・0・CL/TEなしで完成。GET/DELETEも指定本文を受信。chunkedのサイズ行・本文・CRLF・trailerを別断片で受信し、Wikipedia例の復号後9バイトとNUL/CR/LF保持を確認。0/00は最後の空行まで未完成、不正サイズ・桁あふれ・区切り不正は400 |
| HTTP chunk extension/trailer | HTTP-15・16: サイズ行の4;foo=bar・quoted-string/escape・RFCの空白を受理して捨て、不正構文400。trailerの共通構文を検査し、Host/CLや重複も意味検査せず捨てる。初期Trailerなしでも受理し、初期ヘッダー・本文長・CGI環境変数は変更しない |
| HTTP chunkedの上限 | HTTP-17・18: サイズ行CRLF込み8,192バイト、extension合計32,768バイト、trailer終端空行込み32,768バイトは受理。各1バイト超過で400/400/431。終端チャンクのextensionも合計へ数える。正常な次のサイズで本文上限超過が判明すれば本文待ちせず413、付加情報は本文サイズへ加算しない |
| HTTP通常ソケットの部分I/O | HTTP-65: 準備通知後の各周/方向1回・各最大8 KiBと送信オフセットを確認。断片受信、短いsend、再通知で全文を受け取れること、recv負値/send0以下でcloseしerrnoによる分岐・追加応答がないことを確認する |
| HTTP EOF/半閉鎖/RST | HTTP-66: 未完了EOFは破棄、完成後/早期拒否応答中のEOFでも応答継続。通常ソケットのHUPと未読データを同時に発生させ最後まで解析できること、ERR/NVAL・RSTで資源を回収することを確認。CGI待機中はCGI-113を維持 |
| HTTP早期拒否と切断待ち | HTTP-67: 大きなPOSTへの413、Expectへの417等をブラウザ/自作socketで受信。追加受信を保存/再解析せず、全送信後にSHUT_WR、EOFまたは1秒でcloseすることを確認。相手が送り続けても期限を延長せず、通常応答・受信EOF済み・shutdown失敗も確認。応答欠落/RSTと負荷に問題があれば採用案を見直す |
| HTTP絶対期限 | HTTP-68・69: accept後に無送信なら60秒で無応答close、断片送信を続けても未完了なら60秒で408。未完了の保存/削除/CGI起動なし。読まない相手への送信はバッファ準備から60秒でclose、部分送信で延長/追加応答なし。CGI10秒と送信後1秒を混同しない |
| HTTP共通計時の代替案 | HTTP-70: /proc/uptimeの利用可否、無負荷/実負荷の精度・取得I/Oの影響、poll待ち最大100 msを確認。読取/解析失敗と逆行では通常接続close、CGIは既存の500と停止/回収。計時不能の無期限継続なし。Dateへ流用せず、ログに日時取得関数を追加しない |
| HTTP通常ファイルと並行処理 | HTTP-71・74(2026-11-15更新): 各FileReadTaskが各周にread最大1回・最大8 KiBだけ進み、次の読み取りまでにEventLoopへ戻ることを確認。空/大きいファイル、短い正のread、EOF、負値、close失敗を確認し、静的配信はEOFとclose成功前に200を送らず、失敗時に部分本文を送らない。GETとerror_pageの読み取り中にも小さいGET/CGIが進むことを確認。API改訂後にerror_page失敗で元コード・必要ヘッダーを保持し内蔵HTMLへ戻ることを検証する。単発のディスク待ちと全量保持は残る。アップロード書き込みは2026-11-16合意で分割するが実装・受入試験は未完了。一覧生成は2026-11-19に段階分割を採用し、所有/API/処理量を具体化する。serializeの停滞対策は未決定 |
| Routerのファイル読み取り待ち | HTTP-63・CGI-95(2026-11-15更新): RESPONSE_READY/CGI_REQUIRED/FILE_READ_REQUIRED/FILE_WRITE_REQUIRED/AUTOINDEX_REQUIREDに対応するresponse/cgi/file/write/autoindexだけを使う。静的ファイルと静的indexは解決・権限確認済みの絶対パスとContent-Typeをfileへ格納する。Router内で読み切らずFileReadTaskを開始し、指示の一時変数を破棄しても読み取りが継続できる。完了時にrouteを再実行しない。CGI起動待ち/リダイレクトなど不要な経路でファイル読み取りを開始せず、RESPONSE_READYのエラーは別途共通のerror_page準備経路へ渡す |
| Clientのファイル読み取り状態 | [READING_FILEの基本遷移](../Requirements.md#client-file-read-state): 完成要求からrouteを1回だけ呼び、RESPONSE_READYは共通error_page準備経路へ渡し、読み取り待ち不要かつ本文確定後にWRITING_RESPONSE、CGI_REQUIREDは起動成功後WAITING_CGI、FILE_READ_REQUIREDはREADING_FILEへ進むことを確認。読み取り未完了ではREADING_FILEを維持し、EOFとclose成功後に200応答を準備してWRITING_RESPONSEへ進む。open/read/close失敗は部分本文を破棄して500生成へ進み、route再実行や部分本文の200送信がないことを確認。error_page経由の状態遷移と異常時の後始末は設計決定後に試験を具体化する |
| FileReadTaskの保持とコピー制約 | ClientがFileReadTaskを値メンバーとして保持し、未開始時はfd=-1・空本文で本文用の大きなバッファを先行確保しないことを確認。タスク自体のnew/deleteを使わず、Clientの生成・破棄に伴って生成・破棄する。FileReadTask/Clientのコピーコンストラクター・コピー代入がprivate宣言かつ未定義で、外部からのコピー/代入をコンパイル時に拒否することを確認。EventLoopはClientへのポインターを管理する。fd解放・中止・CGI子専用解放の具体的な試験はAPI決定後に追加する |
| FileReadTaskの4操作 | [4操作の役割](../Requirements.md#file-read-task-operations): startで必要な指示を保持して対象を開き、stepでreadを最大1回・8 KiBだけ行うことを確認。EOF後のclose成功時だけ完了となり、失敗時に完了として本文を渡さない。takeBodyは完了本文をswapで呼び出し側へ渡し、cancelは保持fdと途中の本文を解放する。start(const FileReadRequest&)のbool戻り値について、trueが読み取り準備成功であり本文完成を意味しないこと、falseが開始失敗であることを確認。HTTPコードの生成や内蔵HTMLへの切り替えを呼び出し側が判断し、一時的なrequestの破棄後も必要情報が有効であることを確認。step()がStepResultのIN_PROGRESS/COMPLETED/FAILEDを返すこと、短い正のreadでは未完了、readの0とclose成功で完了、read負値またはclose失敗で失敗となることを確認。takeBody(std::string&)は完了後の初回だけtrueで本文を渡し、空ファイルでもtrueとなることを確認。未開始/未完了/失敗/取得済みならfalseで引数不変とする。出力引数の既存内容は成功時に置き換え、swapでタスクへ移った以前の内容を解放することを確認。void cancel()を未開始/読み取り中/完了/失敗/取得済みで呼び、保持fdを-1にして一度だけcloseし、close失敗でも再試行しないことを確認。本文の解放とtakeBody不可、再cancelと続くデストラクターで二重closeしないこと、取得済みの呼び出し側の本文は変わらないこと、HTTP応答生成と外へ出る例外がないことを確認。未開始時だけstartでき、読み取り中/完了/失敗/本文取得済みでのstartがfalseで現在のfd・本文・状態・保持情報を変更せず別ファイルを開かないことを確認。cancel後は未開始として新しい読み取りを開始でき、失敗後にerror_pageを読む場合もcancelを挟む。完了かつ本文未取得でのstepはI/OなしでCOMPLETED、失敗ならI/OなしでFAILED、未開始/本文取得済みなら状態不変・I/OなしでFAILEDとなることを確認。完了/失敗後の再stepでread/closeを繰り返さず、cancel後は未開始の規則に従うことを確認。privateなPhaseがIDLE/READING/DONE/ERROR/TAKENの5種類で、初期状態とcancel後はIDLE、start成功後はREADING、EOFとclose成功後はDONE、本文取得成功後はTAKENになることを確認。完了/失敗/取得済みのboolを重ねて持たず、IDLE/TAKENでのstepのFAILEDをERRORへの遷移と混同しない。IDLEからの通常の開始失敗では取得済みfdを一度だけ閉じ、本文/保持情報を解放しERROR・falseとなることを確認。IDLE以外からのstart拒否とは区別する。本文蓄積/start中のstd::bad_allocを通常失敗のfalse/FAILEDへ変換せず、接続の資源回収とcloseへ進むことを確認。具体的な捕捉位置/回収経路に合わせた追加試験は実装時に具体化する |
| error_pageの準備と応答保持 | [prepareErrorPageの契約](../Requirements.md#error-page-prepare): 200〜399は応答不変でfalse、400〜599かつ未指定なら内蔵HTMLで確定してfalse、指定ありなら絶対パスとtext/htmlをfileへ設定してtrueとなることを確認。fileはtrue時だけ使用し、準備関数がopenしないことを確認。Clientが元のHttpResponseを保持し、本文確定前にserialize/送信しないことを確認。元コード・空も含むCGI説明文・Allow等を保持し、読取成功の空ファイルと読取失敗を区別する。finishErrorPage(true, body)が空も含めた取得本文を採用し、falseでは引数の本文を使わず元コードの内蔵HTMLを生成することを確認。両経路で元コード/説明文/必要ヘッダーを保持し、元本文関連ヘッダーを除去してtext/htmlと最終本文長を設定する。ファイル読み取り/再選択をせず、確定後にserialize/送信することを確認。各応答経路からClient側の共通送信準備へ渡し、prepareErrorPageを応答ごとに1回だけ呼ぶことを確認。falseなら直接送信準備、trueなら読み取りへ進み、finishErrorPage後にprepareErrorPageへ戻らないことを確認。Router/CGIパーサーから個別に準備しない。Client::prepareResponse(const HttpResponse&)を入口とし、引数をpendingResponse_へコピー後に準備することを確認 |
| makeErrorと共通準備経路 | makeError(int code)が設定を受け取らず、400〜599のコード/標準説明文だけを設定した応答を返し、本文生成/ファイル読み取り/prepareErrorPage呼び出しを行わないことを確認。Routerのエラー・要求解析失敗・CGI開始/実行/解析失敗の応答を共通準備経路へ渡す。正常なCGIの404等は元の検証済み応答を渡し、説明文/必要ヘッダーを失わせない。RESPONSE_READYでもエラー本文確定前にserialize/送信しない。共通送信準備はClient側に置き、要求解析/Router/CGIの経路から集約することを確認。error_pageの開始/読取失敗で別のmakeErrorや再準備を呼ばず、元コードを保ってfinishErrorPageへ進む。Client::prepareResponse(const HttpResponse&)を使って各応答経路が共通化されていることを確認 |
| HttpResponseの旧I/O API廃止 | applyErrorPage/readErrorPageの宣言・実装・呼び出しが残らず、HttpResponseがファイル読み取りや所有fdを持たないことを確認。選択はprepareErrorPage、読み取りはFileReadTask、確定はfinishErrorPageへ分担する。buildDefaultErrorBody/removeBodyHeadersは共通補助として残し、空本文・失敗時の内蔵HTML・必要ヘッダー保持の既存試験を確認する |
| Clientの読み取り用途 | privateなFileReadPurposeがSTATIC_FILE/ERROR_PAGEの2種類で、読み取り開始時に設定しREADING_FILE中だけ参照することを確認。STATIC_FILE成功は200、失敗は500の元応答を共通送信準備へ渡す。ERROR_PAGEは元応答を保持し、成功/失敗をfinishErrorPageへ渡して直接送信準備へ進む。ERROR_PAGEの開始失敗も元コードを保持して内蔵HTMLへ戻し、500の再生成や再準備を行わない。FileReadTask/FileReadRequestが用途を持たず、「用途なし」の状態を追加しないことを確認する |
| Clientの応答保持 | HttpResponse pendingResponse_をClientの値メンバーとして保持し、応答自体のnew/deleteを行わないことを確認。共通送信準備へ渡した一時的な応答を破棄しても、error_page読み取り中に元コード・説明文・必要ヘッダーが有効なことを確認。finishErrorPageを保持応答へ適用し、確定後にserializeして送信バッファへ渡す。Client.hppが値メンバーの完全な型定義としてHttpResponse.hppをincludeすることを確認。分離送信で本文を送信完了または打ち切りまで保持し、その後に回収することを確認 |
| Client::prepareResponse | const HttpResponse&の引数をpendingResponse_へコピーし、元の一時応答を破棄しても有効なことを確認。prepareErrorPageを1回呼び、falseなら分離送信の準備後にWRITING_RESPONSE、trueなら用途ERROR_PAGEでタスクを開始してREADING_FILEへ進む。開始失敗は元コードの内蔵HTMLへ戻す。finishErrorPage後にprepareResponseへ再投入せず、直接serialize/送信準備へ進むことを確認。std::bad_alloc時は500再生成なしで資源回収とcloseへ進み、分離送信で必要な本文を送信中も保持することを確認 |
| Client::beginWritingResponse | 確定済み応答からヘッダー部を準備し、保持済みの本文を使う分離送信へ進む。sendはPOLLOUTの後に行い、送信準備完了から60秒の期限を適用する。prepareErrorPageがfalseの経路とfinishErrorPage後の経路から共用する |
| HttpResponse::resetと保持領域解放 | 公開void reset()が本文/ヘッダー等の保持領域を解放して200/OKへ戻し、Clientの送信文字列・オフセット・状態・期限に触れないことを確認。大きい本文を空に見せるだけで保持領域を残さないことを確認。[分離送信中の保持と回収](../Requirements.md#response-body-send-lifetime)に従い、部分送信中はpendingResponse_をresetせず、本文を直接参照することを確認する。正常完了・空本文・本文なし・切断・期限超過で、送信データを使い終えてから保持領域を回収することを確認する。切断待ちに入る場合もその前に回収し、メモリ確保失敗では接続終了と破棄による回収を確認する |
| CGIディレクトリへのPOST | [2026-11-16の合意](../Requirements.md#cgi-post-directory): CGI有効/POST許可の同じlocationに一致するディレクトリ要求を末尾/あり・なしで送り、設定indexがCGI対象ならリダイレクトなしで実行されることを確認。SCRIPT_NAMEに選択indexを含め、PATH_INFOは空、元query・復号済みPOST本文を渡す。GETの末尾/補完・既存のメソッド禁止・return優先・upload保存は維持する。別locationの再探索とindex.pyの自動補完を行わない。CGI非対象時は405・対象別Allow、実ディレクトリの設定index不在は403・一覧なしの試験へ進む |
| CGI非対象へのPOST | [2026-11-16の405規則](../Requirements.md#cgi-post-non-script): CGI有効/POST許可で実在する通常ファイルまたは選択indexがCGI拡張子に該当しなければ405。GET/POST許可ではAllow: GET、POSTのみ許可では空のAllowをヘッダーとして出力し、いずれもPOST/DELETE/HEADを追加しない。ファイル名の大小文字とCGI拡張子の既存比較規則を維持する。CGI起動/保存/静的本文の成功応答への切り替えを行わず、error_page置換後も405/Allowを保持する。locationでの禁止による405の一覧、対象なし404や権限不足403等の既存分類は維持する |
| CGIディレクトリPOSTのindex不在 | [2026-11-16の403規則](../Requirements.md#cgi-post-missing-index): CGI有効/POST許可で対象ディレクトリが実在し設定indexだけが不在なら403。autoindex on/offと末尾/の有無で結果を変えず、一覧/保存/別スクリプトの自動補完/リダイレクトへ進まない。ディレクトリ自体の不在は404、実在する通常ファイルの非CGI indexは405と対象別Allowに区別する。403の元応答を共通送信準備へ渡し、カスタム/内蔵error_pageでも403を保持する。GETの既存index/autoindex動作は維持する |
| RouterのCgiTarget | [2026-11-16採用の補助型](../Requirements.md#router-target-api)がRouterのprivateに置かれ、scriptPath(ディスク上の絶対パス)/scriptName(URL上のスクリプトパス)/pathInfo(追加パス)の3文字列だけを持つことを確認。GET/POST内のローカル変数として使い、Routerのメンバーに残さない。fd・Config参照・要求本文・Pythonパス・cwdを重複保持せず、公開CgiRequestとの役割を混同しない。値の生成/検証は採用済みのfindCgiScript返却契約の試験へ進む |
| Router::findCgiScript | [2026-11-16採用のprivate API](../Requirements.md#router-target-api)がurlPath/LocationConfigとtarget/found出力を受けるconst関数であることを確認。0/trueではCGI対象を確定しtargetを使用、0/falseではtargetを使わずメソッド別の後続処理へ、HTTPエラーコード/falseでは通常配信へ切り替えずエラー応答へ進む。indexを自動選択せず、ハンドラーが選択後に同じCGI判定を適用する。PATH_INFO付きスクリプト、CGI非該当の通常ファイル、ディレクトリ、stat/access失敗、禁止種別で結果を区別する。メモリ不足は通常の確認失敗へ変換せず既存の接続終了方針へ進む |
| Router::prepareCgiRequest | [2026-11-16採用のprivate API](../Requirements.md#router-target-api)が要求/location/接続情報/有効なCgiTargetから実行要求を生成することを確認。成功0のときPython/スクリプト絶対パス・script親ディレクトリのcwd・既存規則の環境変数・復号済み本文が揃い、CGI_REQUIREDへ進む。既存の入力検証失敗ではHTTPエラーコードを返し、部分cgiを起動に使わない。SCRIPT_NAME/PATH_INFOはtargetのURL側、本文はNULも保持する。対象の再探索・Python再検査・fork/execを行わず、一時的な入力を破棄しても出力値が有効なことを確認。bad_allocは通常の検証エラーへ変換せず資源回収/接続終了へ進む |
| メソッド別ハンドラーの設定引数 | GET/POST/DELETEの3ハンドラーがServerConfigを受け取らず、選択済みLocationConfigから配信先/index/CGI/アップロード設定を使うことを確認。location選択とメソッド許可はRouter本体で完了し、禁止メソッドでハンドラーを実行しない。エラー生成はmakeError(code)、error_pageの設定参照は後続のClient側で行い、ハンドラーで設定を再解釈しない。公開routeのServerConfig引数とConfig自体の設計は維持する |
| Routerの3ハンドラーAPI | [2026-11-16採用の宣言](../Requirements.md#router-target-api)でGET/POST/DELETEがprivateなconst関数としてRouteResultを返すことを確認。GETはrequest/location/connectionを受けresponse/cgi/file/autoindexを、POSTは同入力からresponse/cgi/writeを、DELETEはrequest/locationからresponseだけを出力する。GET/POSTは戻り値に対応する出力だけを使い、DELETEは成功204/各種エラーのどちらもRESPONSE_READYで返す。エラー本文をハンドラー内で確定せず、後続の共通送信準備へ渡す。不要なConnectionInfo/CGI/ファイル指示をDELETEへ渡さない |
| アップロード分割保存と受け渡し | [2026-11-16合意](../Requirements.md#file-write-design): multipart全体の検証前にファイルを作らず、RouterがFILE_WRITE_REQUIREDとFileWriteRequest(path/filename/body)を返すことを確認。Client所有の値メンバーFileWriteTaskが各周にwrite最大1回・最大8 KiBだけ進め、小さいGET/CGIと期限確認が進むことを確認する。短い正のwrite・0・負値・空ファイル・close失敗・既存対象409を確認し、保存とclose成功前に200を送らない。失敗時は今回作成した途中ファイルだけをfdの後始末後に削除し、削除失敗でも500とログを維持する。通常ファイルはpoll登録せず、write後のerrno分岐を行わない。個々の同期I/O待ち・全体検証・全量保持の負荷は実測する。所有権移譲API・状態・期限・切断時の詳細は設計確定後に試験を具体化する |
| 保存本文のswapによる引き渡し | FileWriteRequestのbodyからFileWriteTaskへ本文をswapで移し、指示側bodyが空になること、タスク開始用に本文のコピーを追加しないことを確認する。空本文・NULを含むバイナリ本文・大きい本文のバイト列を保持し、引き渡し後に一時的な指示を破棄しても分割保存を続けられることを確認する。タスクに指示側本文への参照・ポインターを残さない。元のHttpRequest本文はstart成功後に解放する。multipart抽出から開始までの重複保持は測定する。startはパス/保存名の保持とopen成功後にだけ本文を移すこと、通常の開始失敗では指示側本文を変更しないことを確認する |
| 保存タスクの開始API | [startの契約](../Requirements.md#file-write-start): int start(FileWriteRequest&)の0は開始準備完了であり、空本文でも保存成功の200を直ちに生成しない。パス/保存名をタスクの値として保持してopenし、成功後だけbodyをswapする。通常の開始失敗は同名409、権限/読み取り専用403、保存名のパス長400、保存先消失/その他500に分類し、指示側本文を変更しない。bad_allocはHTTPコードに変換せず既存の接続終了へ進める。状態別の呼び出し契約・再利用・通常の開始失敗時の解放は下記で検証し、例外時の詳細な回収は設計確定後に具体化する |
| 保存タスクの進行API | [stepの契約](../Requirements.md#file-write-step): StepResult step()が各回write最大1回、残量と8 KiBの小さい方までを要求することを確認。短い正のwriteで位置を戻り値分だけ進め、残量があればIN_PROGRESSでEventLoopへ戻る。最後のデータを書いた後はclose成功でCOMPLETEDとし、空本文ではwriteせずclose成功で完了する。writeの0/負値・close失敗はFAILEDで、後始末と500へ進む。完了前に成功応答を生成・送信せず、write後のerrno分岐を行わない。保存中以外の呼び出しは下記の状態別契約で検証し、回収用メンバーの具体形は設計確定後に検証する |
| 保存成功応答の生成分担 | [2026-11-16合意](../Requirements.md#file-write-success-response): FileWriteTaskが保存結果と保存名だけを渡し、HttpResponse/HTMLを生成しないことを確認する。全量保存とclose成功を確認した後、rysato側のHttpResponse::makeUploadSuccess(filename)が保存名をHTMLエスケープした200/text/html応答を作り、ClientがprepareResponseへ渡す。保存名にHTML特殊文字を含む場合も確認し、公開URL/Locationを自動生成せず、Routerの再実行や保存の再実行を行わない。開始失敗と保存途中の失敗では成功応答を生成しない。保存名はtakeFilenameで完了後の初回だけ取得し、異常時の回収は設計確定後に検証を具体化する |
| アップロード成功応答の生成API | [makeUploadSuccessの契約](../Requirements.md#upload-success-api): public static HttpResponse makeUploadSuccess(const std::string& filename)が200 OKと保存名をHTMLエスケープした結果HTMLを返すことを確認。Content-Typeはtext/html、Content-Lengthはserialize後の最終本文の実長と一致する。特殊文字・空白・非ASCIIの保存名を確認し、実際の保存名やパスをHTMLエスケープで変更しない。入力を破棄しても応答が有効であり、ファイル操作・Config参照・公開URL/Location自動生成・error_page準備を行わない。Clientは保存成功後だけ生成しprepareResponseへ渡す |
| 保存タスクの保存名取得API | [takeFilenameの契約](../Requirements.md#file-write-take-filename): bool takeFilename(std::string&)が保存/close成功後の初回だけswapで検証済み保存名を渡してtrueを返すことを確認。未開始/保存中/開始失敗/保存失敗/取得済みではfalseで出力不変とする。出力に既存内容があっても正しい保存名を渡し、swapでタスク側へ移った旧内容を保持し続けない。タスクへの参照を出力へ残さず、取得後にタスクを破棄してもmakeUploadSuccessへ渡せること、保存済みファイルを削除/再保存しないことを確認。cancel後は出力不変でfalseとし、内部状態の契約は状態設計の確定後に検証する |
| 保存タスクのキャンセルと破棄 | [cancelの契約](../Requirements.md#file-write-cancel): 保存途中ではfdの後始末後に今回作成した途中ファイルだけを削除し、全量保存とclose成功後は保存名取得前/取得後・応答生成前/送信前/切断時とも保存済みファイルを残す。close失敗は成功扱いにせず後始末する。未開始・開始失敗・保存中・保存失敗・完了・保存名取得済みでcancelを繰り返し、二重close/二重削除がなく保持データを解放して未開始へ戻ることを確認。削除失敗のログ、既存対象の保護、取得済み保存名の維持、cancel後のtakeFilenameがfalseで出力不変となることも確認する。デストラクターも同じ後始末を行い例外を外へ出さない。例外対策と異常経路への具体的な組み込みは設計確定後に検証を具体化する |
| 保存タスクの内部状態 | [Phaseの契約](../Requirements.md#file-write-phase): privateなPhase(IDLE/WRITING/DONE/ERROR/TAKEN)で保持し、完了/失敗/取得済みのboolを重ねない。初期/cancel後はIDLE、start成功でWRITING、通常の開始/保存失敗でERROR、全量保存とclose成功でDONE、保存名の初回取得でTAKENとなることを確認。各状態からcancelでIDLEへ戻り、DONE/TAKENの保存済みファイルは残す。公開StepResultと内部Phaseを混同せず、資源の二重close/削除を防ぐ。startの状態別契約は下記で検証し、stepの状態別契約も下記で検証する |
| 保存タスクのstart再呼び出し | [状態別契約](../Requirements.md#file-write-start-state)に従い、IDLEで開始できることを確認。WRITING/DONE/ERROR/TAKENでは500を返し、Phase/本文/パス/保存名/書き込み位置/fd/ファイルと入力指示を維持し、openやswapを行わないことを確認する。返却500でタスクをERRORへ変更しない。再利用時と通常の開始失敗後はcancelでIDLEへ戻してから開始し、途中ファイルの回収と保存済みファイルの維持を確認する |
| 保存タスクのstep再呼び出し | [状態別契約](../Requirements.md#file-write-step-state)に従い、WRITINGでだけ保存を進めることを確認。DONEでCOMPLETED、ERROR/IDLE/TAKENでFAILEDを返し、各状態で繰り返し呼んでもPhase/保持データ/ファイルを維持してwrite/close/削除を行わないことを確認する。IDLE/TAKENでのFAILEDで内部状態をERRORへ変更せず、DONEでのCOMPLETEDの再返却によって保存名を二重取得できないことを確認する |
| 保存タスクの通常の開始失敗時の解放 | [解放契約](../Requirements.md#file-write-start-failure)に従い、IDLEからの通常の開始失敗で保持パス/保存名/本文を解放し、書き込み位置0・fd=-1・ERRORとなることを確認。取得済みfdは一度だけcloseし、close失敗時も再試行しない。入力指示の全フィールドを維持し、再cancel/破棄で二重close/削除を起こさない。途中ファイルがあれば後始末/ログを終えてからパスを解放し、既存対象は保護する。ERRORからcancelでIDLEへ戻して再利用できること、IDLE以外のstart拒否では状態維持契約が適用されることを確認する |
| 保存タスクのコピー禁止 | [2026-11-17合意](../Requirements.md#file-write-noncopyable)に従い、FileWriteTaskがprivateな未定義のコピーコンストラクター/コピー代入演算子を持ち、コピーできないことを確認。Clientが値メンバーとして所有し、既存のClientのコピー禁止とEventLoopのポインター管理に従ってfd/途中ファイルの所有者を一つに保つ。タスクはClientと一緒に生成/破棄する |
| Clientの保存中の基本遷移 | [2026-11-17合意](../Requirements.md#client-file-write-state)に従い、FILE_WRITE_REQUIREDからstart成功でWRITING_FILE、IN_PROGRESSで同状態を維持することを確認。COMPLETEDで保存名を取得してmakeUploadSuccessからprepareResponseへ渡し、共通送信準備後にWRITING_RESPONSEへ進む。開始失敗は返却コード、保存途中の失敗は後始末後500を共通準備へ渡し、error_page設定があれば既存のREADING_FILE経路を使う。Router/保存を再実行しない。期限/切断/例外時と契約違反時の詳細は設計確定後に検証する |
| 保存タスクの保存失敗時の回収 | [2026-11-17合意](../Requirements.md#file-write-step-failure): WRITING中のwriteの0/負値・最後のclose失敗で、stepがfdの一度だけのclose、今回作成した途中ファイルの一度だけの削除試行と失敗ログ、本文/パス/保存名の解放、位置0への復帰を済ませてERRORでFAILEDを返すことを確認。最後のcloseが失敗しても同じfdを再closeしない。削除失敗でも500を維持し、後続のstep/cancel/破棄で二重close/削除を行わない。Clientは結果を受け500を共通準備へ渡す。ログや回収経路の例外時の詳細は後続の設計で具体化する |
| 保存成功時の本文/パス解放 | [2026-11-17合意](../Requirements.md#file-write-success-release): 全量保存とclose成功でDONEを確定し、タスクの保存本文/保存先パスの保持領域を解放してからCOMPLETEDを返すことを確認。空/大きいファイルで保存名がtakeFilenameまで有効であり、完了後のstep再呼び出しでI/Oや再保存が発生せず、cancel/破棄で成功済みファイルを保護することを確認する。解放経路の例外でも成功ファイルを維持する。元のHttpRequest本文の解放時期は別途設計する |
| アップロード元の要求本文の解放 | [2026-11-17合意](../Requirements.md#upload-request-body-release): FileWriteTask::startが0で本文を取得した後、Clientが元のHttpRequest本文の保持領域を解放してから分割保存を進めることを確認。メソッド/URL/ヘッダー/解析完了状態を保持し、保存本文のバイト列と保存結果が正しいこと、後続処理が解放済み本文への参照を使わないことを確認。空/大きい/バイナリ本文を対象とし、開始失敗時の指示保持を確認する。本文解放はCOMPLETE後のreleaseBodyで行い、誤呼び出し時は状態/保持データを維持し、例外経路は設計確定後に具体化する |
| HttpRequestの本文解放API | [releaseBodyの契約](../Requirements.md#request-release-body-api): publicなvoid releaseBody()をCOMPLETE後にClientが呼び、body_を空にして保持領域を解放することを確認。繰り返し呼んでも本文以外のメソッド/URL/ヘッダー/解析状態/エラーコードを維持し、getBody()が空となることを確認。アップロードではstart成功後に呼び、タスクの保存データを維持する。COMPLETE後のappendDataで本文を再蓄積しない。解放前の本文への参照/ポインターを後続処理で利用しない。INCOMPLETE/ERRORでreleaseBodyを繰り返し呼んでも本文/保持領域/解析段階/状態/エラーコードが不変となることを確認。INCOMPLETEでは続くappendDataで正しく解析を完了でき、ERRORでは最初の確定エラーを保持する。例外経路は後続の設計で具体化する |
| ファイル読み書きの固定期限方式 | [2026-11-17合意](../Requirements.md#file-task-timeout-policy): 静的ファイル/error_page読み取りとアップロード保存で、read/writeが正のバイト数を返しても期限を延長しないことを確認する。進行中の期限超過と、単発の同期I/Oから戻った後の期限確認を対象とする。Clientがstart直前に開始時刻を記録し、開始準備/openの経過時間を含めることを確認。Routerの検証/multipart解析は開始時刻より前に行い、起点を進行ごとに更新しない。error_pageは下記の独立期限を使う。期限は各60秒の採用案とする。静的読み取りと保存の超過時は下記の回収後500の契約に従う。計時失敗は下記の回収/接続終了の試験に従う |
| error_pageの独立した固定期限 | [2026-11-18合意](../Requirements.md#error-page-timeout-budget): 元の処理が期限切れでもerror_pageは自身のstart直前に開始時刻を記録し、進行による延長がないことを確認。error_pageの期限超過ではタスクをcancelし、fd/部分本文を回収してfinishErrorPage(false, body)で元のステータス/説明文/保持対象ヘッダーを維持した内蔵HTMLを確定し、beginWritingResponseへ進む。再選択/読み取り再開がなく、元処理とerror_pageと送信の期限を順に適用した接続全体の所要時間も確認する。error_pageも新しい60秒を計り、計時失敗は下記の回収/接続終了に従い、全体の期限確認は下記のpoll復帰後/イベント前の契約に従う |
| ファイル処理の各60秒の採用案 | 2026-11-18に静的読み取り/アップロード保存/error_pageを各60秒とした採用案を検証する。各start直前から計り、進行で延長せず、error_page開始時は新しい60秒とする。期限付近の処理と長時間の並行負荷を対象に、小さいGET/CGIの進行、期限検出、資源回収、接続全体の所要時間を測定する。同期I/Oの復帰待ちと期限確認の遅れを記録し、秒数の妥当性を見直す。超過時の応答と完了判定の優先順序は後続の設計に従う |
| 静的読み取り/保存の期限超過応答 | [2026-11-18合意](../Requirements.md#file-task-timeout-response): 期限超過でClientが対象タスクをcancelし、静的読み取りのfd/部分本文、保存中のfd/今回作成した途中ファイル/保持データを回収してからmakeError(500)をprepareResponseへ渡すことを確認。削除失敗はログを記録して500を維持し、既存対象と保存成功済みファイルを保護する。error_pageを設定した場合は新しい60秒を計り、その超過では500の内蔵HTMLへ進む。送信準備後の送信60秒も確認する。同期I/O復帰時は下記のstep前後の確認順序と完了優先に従う |
| step前後の期限確認と完了優先 | [2026-11-18合意](../Requirements.md#file-step-timeout-order): 静的読み取り/error_page/保存でstep前に期限確認し、超過ならstepを呼ばず用途別の回収/応答へ進む。期限内に呼んだstepがCOMPLETEDなら復帰時に期限を過ぎていても成功経路へ、FAILEDなら既存の失敗経路へ進む。IN_PROGRESSなら復帰後の時刻で再確認し、超過なら回収/応答、期限内なら次の周へ戻す。期限直前/到達/直後、短い正のI/O、最後のI/Oとcloseが長引くケースを確認し、保存成功済みファイルの保護とerror_page本文の適用も確認する。start復帰時は下記の契約に従い、計時失敗は下記の回収/接続終了に従う |
| start成功直後の期限確認 | [2026-11-18合意](../Requirements.md#file-start-timeout-order): 静的読み取り/error_page/保存でstart成功後すぐに再計時し、期限に達していればstepを呼ばずcancelから用途別の超過処理へ進むことを確認。open/開始準備が長引く場合と空ファイルを含める。start失敗時は期限超過より既存の失敗結果を優先し、静的読み取り500、保存の返却コード、error_pageの内蔵HTMLへ進む。期限内のアップロードは元のHttpRequest本文を解放してから分割保存する。計時失敗は下記の回収/接続終了に従い、例外経路は後続の設計で具体化する |
| ファイルタスク中のpoll待ち時間0 | [2026-11-18合意](../Requirements.md#file-task-poll-timeout): 進行可能な静的読み取り/error_page/保存タスクがある間、単一pollの待ち時間を0とし、ソケット/CGIイベントと期限を確認して各タスクを各周最大1回進めることを確認。イベント0件でもファイル処理が進み、read/writeは各タスク最大1回/8 KiBとする。完了/失敗/取得済みタスクで待ち時間0が継続せず、進行可能タスクがなくなれば通常の最大100 msの待機へ戻る。通常ファイルをpoll登録せず、ソケットPOLLOUTで駆動しない。並行GET/CGIの応答、CPU使用率、終了後の待機と資源回収を測定する |
| poll復帰後のイベント優先 | [2026-11-18合意](../Requirements.md#file-task-event-order): poll通知のソケット/CGIイベントを先に処理し、生存中で継続する接続のファイルタスクを後段で期限確認付きで進めることを確認。切断/終了が確定した接続を回収して後段の対象から外し、受信EOFでも応答を継続する既存の判定を維持する。破棄したClientを参照せず、イベント0件でもファイル処理を進める。大量のreadyイベントでも既存の処理量制限に従って後段へ進むことを並行GET/CGI/保存で確認する。新規タスクは下記の次周stepの契約に従う。走査/破棄の具体形は後続設計で検証を具体化する |
| 新規ファイルタスクの最初のstep | [2026-11-18合意](../Requirements.md#file-task-first-step): イベント処理/別タスクの結果処理で開始した静的読み取り/error_page/保存タスクは、その周にstartと直後の期限確認を行い、最初のstepを次の周へ回すことを確認。開始失敗/直後の期限超過はその場で既存の経路へ進む。静的読み取り/保存失敗からerror_pageへ移る場合と同じFileReadTaskの再利用でも同じ周にstepを連鎖しない。次周のpollは待ち時間0とし、イベント優先/期限確認/各タスク最大1回を維持する。対象管理の具体形は後続の内部設計で定める |
| Client本体の周末での削除 | [2026-11-18合意](../Requirements.md#client-deferred-destruction): 終了確定時にCLOSINGとし、ファイルタスクcancel/fd監視解除とclose/CGI関連付け解除を行い、本体deleteとコンテナ除去はEventLoop周末まで遅らせることを確認。後段のイベント/期限/ファイル走査でCLOSINGを処理せず、古いpoll通知を既存の有効性確認で除外する。終了処理の再呼び出しとデストラクターで二重close/削除を起こさず、成功済みファイルを維持する。CGI停止/子回収を継続し、切断後の結果をClientへ返さない。走査中と周末削除後の参照の有効性を確認する |
| ファイル処理の計時失敗 | [2026-11-18合意](../Requirements.md#file-task-clock-failure): 静的読み取り/error_page/保存の開始時刻取得・開始直後/step前後の必要な再計時で取得/解析失敗や逆行を注入し、該当ClientがタスクcancelからCLOSING、fd監視解除/close、CGI関連付け解除と既存回収、周末削除へ進むことを確認。開始時刻取得失敗ではタスクを開始せず、追加の500/error_page/内蔵HTMLを生成しない。保存途中の今回作成したファイルを回収し、保存とclose成功後のファイルは保護する。再終了処理/破棄で二重close/削除を防ぎ、共通時刻の失敗時は影響する各ファイル処理中の接続を終了する |
| ファイルタスクのFD_CLOEXEC | [2026-11-18合意](../Requirements.md#file-task-cloexec): open直後にF_SETFD/FD_CLOEXECを設定し、成功後だけstart成功と保存本文のswapを行うことを確認。設定失敗を注入し、fdの一度だけのclose、今回作成した途中ファイルの削除、指示側本文の維持、保存500/読み取りfalseの返却を確認。設定処理も開始期限に含める。ファイル読み書き中にCGIを起動し、exec成功後の不要fd非継承を確認する。子の起動失敗では複製fd/メモリを回収して親の保存途中ファイルを保護する。専用解放は下記のcleanupInChildを採用し、入口は下記のClient::cleanupInChildを使う |
| FileWriteTaskの子専用解放API | [2026-11-19合意](../Requirements.md#file-write-child-cleanup): publicなvoid cleanupInChild()をCGI起動失敗の子で各Phaseから呼び、複製fdを一度だけcloseしてfd=-1、保持本文/パス/保存名を解放し位置0/IDLEへ戻すことを確認。親の保存途中/成功済みファイルを削除せず、再呼び出しと続くcancel/デストラクターで二重close/削除を起こさない。Client破棄前に実行し、子の追加確保/ログ/HTTP応答/外へ出る例外がなく、親の保存処理を継続できることを子対象のメモリ検査と並行保存で確認する。入口は下記のClient::cleanupInChildを使う |
| Clientの子専用解放API | [2026-11-19合意](../Requirements.md#client-child-cleanup): EventLoop::cleanupInChildが各Client破棄前にpublicなvoid Client::cleanupInChild()を呼び、読み取りタスクcancel/保存タスクcleanupInChild/複製クライアントfdの一度だけのcloseを行うことを確認。CLOSINGとして続く破棄までに所有メモリを回収し、繰り返し呼び出し/デストラクターで二重close/削除を防ぐ。子のkill/waitpid/shutdown/応答/ログ/追加確保/外へ出る例外を避け、親の接続/CGI/保存途中と成功済みファイルを保護する。CGI起動失敗を注入し、子対象のメモリ検査で回収を確認する |
| Clientの今周開始フラグ | [2026-11-19合意](../Requirements.md#client-file-started-this-round): ClientのfileTaskStartedThisRound_を初期false、周頭でfalse、新規読み取り/保存開始時にtrueとする。周頭リセットがイベント/期限/タスク開始より前となり、後段でtrue/CLOSINGのClientをstep対象から外すことを確認。start直後の期限確認と開始失敗処理はその場で行い、次周にリセットしてから進行する。静的読み取り/保存失敗から同じFileReadTaskでerror_pageへ移る場合もtrueとし、同じ周のstep連鎖を防ぐ。trueでも進行可能タスクがあれば次のpollは待ち時間0となることを確認する |
| EventLoop全体の期限確認順序 | [2026-11-19合意](../Requirements.md#event-loop-timeout-order): 周頭の今周開始フラグリセット、poll、全体の期限確認、ソケット/CGIイベント、ファイルタスク、周末Client削除の順を確認。期限に達した接続をイベント処理前に既存の対象別経路へ進め、閉じたfdやCLOSINGの古い通知を有効性確認で除外する。期限処理で開始したerror_pageは今周開始フラグをtrueとしてstepを次周へ回す。イベント処理が長引いてもファイルタスクのstart/step前後で既存の期限確認を行い、COMPLETED優先を維持する。複数接続/CGI/期限超過/切断を同じ周で発生させ、解放後参照と二重回収を防ぐ |
| autoindexの段階分割 | [2026-11-19合意](../Requirements.md#autoindex-incremental-design): 列挙/ソート/HTML生成で途中状態を保持し、一定量でEventLoopへ戻ることを確認。HTTP-27〜32の表示対象/符号なしバイト順/HTMLエスケープ/URLエンコード/完成HTML8 MiB上限/失敗分類を検証する。全工程とclosedir成功前に200を返さず、列挙中の上限超過で打ち切り回収する。大量項目と並行GET/CGIで、各段階の処理量/応答/期限確認/メモリを測定する。Client所有のAutoindexTaskで進め、API/分割ソート/各周の処理量/期限/回収は設計確定後に検証を具体化する |
| AutoindexTaskの所有 | [2026-11-19合意](../Requirements.md#autoindex-task-ownership): ClientがAutoindexTaskを値メンバーとして保持し、Clientと一緒に生成/破棄することを確認。ディレクトリ/項目一覧/ソート途中状態/HTML生成位置/生成中本文/段階をタスクが所有し、各段階の進行と後始末を担当する。EventLoopは既存のClientポインター管理を使い、Routerは対象解決と一覧生成への分岐を担当する。コピー禁止/Phase/4操作は各採用契約で検証し、初期資源値/期限/異常時とCGI子専用回収は後続の設計確定後に具体化する |
| Routerの一覧生成待ちAPI | [2026-11-19合意](../Requirements.md#autoindex-route-api): AUTOINDEX_REQUIREDのときだけAutoindexRequest(path/urlPath)を使い、route/handleGet末尾の出力引数から対象ディレクトリ絶対パスと正規化済みURLパスを渡すことを確認。既存の末尾/補完/index優先/autoindex設定で一覧生成を選び、Clientが所有するAutoindexTaskを開始する。指示を破棄しても必要情報が有効で、fd/DIR*と指示への参照を渡さず、完了時にRouterを再実行しない。4操作とClientの基本遷移は各採用契約で検証し、期限/進行順序は共通規則を適用し、異常経路の具体的な組み込みは後続設計で定める |
| AutoindexTaskの開始API | [2026-11-19合意](../Requirements.md#autoindex-start-api): int start(const AutoindexRequest&)がpath/urlPathを値で保持してopendirし、成功0とDIR*の所有を確定することを確認。指示を破棄しても有効で、ENOENT/ENOTDIR=404、EACCES/EPERM=403、ENAMETOOLONG=414、その他500を返す。0で列挙完了/200生成とせず、空ディレクトリも後続のstepで確認する。開始時に列挙/ソート/HTML生成を一括実行しない。bad_allocは既存の接続終了へ進める。状態別契約/開始失敗時の回収/期限/CGI子専用解放は後続設計で具体化する |
| AutoindexTaskの進行API | [2026-11-19合意](../Requirements.md#autoindex-step-api): 共通のStepResult step()が内部で列挙/ソート/HTML生成の段階を管理し、一定量でIN_PROGRESSを返すことを確認。全列挙/バイト順ソート/HTML生成/closedir成功後だけCOMPLETEDとし、空ディレクトリも正常な一覧HTMLを生成する。列挙失敗/closedir失敗/HTML上限超過はFAILEDから500へ進み、部分一覧を成功応答へ渡さない。bad_allocは接続終了とする。段階切り替えと結果取得は下記の採用契約で確認し、ソート合計256操作/HTML生成最大8 KiBの採用案で検証し、状態別契約/回収/期限は後続の設計で具体化する |
| AutoindexTaskの完成本文取得 | [2026-11-19合意](../Requirements.md#autoindex-take-body): bool takeBody(std::string&)が全工程/closedir成功後の初回だけ完成HTMLをswapで渡してtrueを返すことを確認。未開始/生成途中/開始失敗/生成失敗/取得済みではfalseで出力不変とし、既存の出力内容がタスク側へ移ったら解放する。空ディレクトリの正常HTMLと8 MiB境界を含め、タスク破棄後も取得本文が有効であること、Client側が200/text/htmlと本文実長のContent-Lengthを生成しprepareResponseへ渡すことを確認。Client状態とキャンセル後は各採用契約で検証し、本文格納は下記のswapによる引き渡しを確認する |
| AutoindexTaskのディレクトリ解放時期 | [2026-11-19合意](../Requirements.md#autoindex-directory-close): readdirの正常終端直後/ソート前にDIR*を所有状態から外してNULLとし、closedirを一度だけ呼ぶことを確認。成功後にソート/HTML生成へ進み、その間ディレクトリを保持しない。空ディレクトリも同じ順序とする。closedir失敗を注入して取得済み一覧等を破棄し、FAILEDから500へ進み、ソート/HTML生成や部分一覧の成功応答を行わないことを確認。再試行と後続の回収/キャンセル/破棄での二重closedirを防ぐ。列挙途中の回収は後続APIの確定後に具体化する |
| AutoindexTaskの列挙量 | [2026-11-19採用案](../Requirements.md#autoindex-enumeration-budget): 列挙段階のstepがreaddir最大32回で戻り、.と../NULL終端/失敗も回数に含めることを確認。終端未確認ならDIR*/項目一覧を保持してIN_PROGRESSとし、次のstepで継続する。32回目付近の正常終端/列挙失敗/HTML上限超過、空/大量項目を試し、不要な追加readdirを行わない。正常終端後は既存のclosedir契約に従う。小さいGET/CGIの進行、期限確認、各周の処理時間/CPU/メモリと一覧生成の所要時間を測定し、回数を検証する |
| AutoindexTaskの段階切り替え | [2026-11-19合意](../Requirements.md#autoindex-stage-transition): stepが一つの段階だけを進め、列挙終端/closedir成功後はソート開始状態、ソート完了後はHTML生成開始状態としてIN_PROGRESSで戻ることを確認。次段階の実処理は次周からとし、空/1項目でも同じ切り替えを行う。HTML生成完了でCOMPLETED、通常の失敗はその段階でFAILEDとする。段階ごとにRouter再実行やstartを行わず、同一タスクとして継続し、並行GET/CGIと期限確認の機会を確保する |
| AutoindexTaskの分割マージソート | [2026-11-19合意](../Requirements.md#autoindex-merge-sort): ボトムアップのマージソートで区間幅/対象範囲/左右の読み取り位置/出力位置を保持し、比較/移動を一定量で中断・再開できることを確認。片側区間の残りの移動と作業領域の準備/初期化も処理量に含める。HTTP-28の符号なしバイト順を逆順/同じ接頭辞/非ASCII/空/1項目/大量項目で検証し、整列完了後は次周からHTML生成する。並行GET/CGIの進行とCPU/ピークメモリを測定する。番号をソート対象とし、比較と書き込みは合計256操作の採用案とし、コンテナ/作業領域は次に具体化する |
| AutoindexTaskの番号によるソート | [2026-11-19合意](../Requirements.md#autoindex-sort-indices): 名前一覧を列挙順で保持し、対応する番号列を名前の符号なしバイト順でマージソートすることを確認。名前一覧の並び/内容を維持し、名前をソート用に複製せず番号を作業領域へ移動する。整列済み番号から名前を参照して生成したHTMLが既存の順序/エスケープ/リンク規則に一致することを確認。番号列の準備も分割し、空/1項目/逆順/共通接頭辞/非ASCII/大量項目と、追加番号列/作業領域を含むピークメモリを検証する。比較/書き込みを合計256操作へ制限し、名前一覧はdeque<string>とし、番号列/作業領域はstd::vector<std::size_t>を1本ずつ使い、列挙後のreserveと段階的なpush_backは下記の契約で検証する |
| AutoindexTaskのソート処理量 | [2026-11-19採用案](../Requirements.md#autoindex-sort-budget): ソート段階のstepが比較1回/番号書き込み1回を各1操作と数え、合計256操作以内で戻ることを確認。番号列準備/作業領域初期化/片側区間の残り移動も数え、途中で回数をリセットしない。上限付近、比較後/書き込み前の中断と再開を確認し、番号の欠落/重複と順序の誤りを防ぐ。長い共通接頭辞/逆順/非ASCII/大量項目で処理時間/並行GETとCGIの応答/CPU/メモリを測定し、採用値を検証する。領域確保/コンテナ拡張の負荷も測定対象とする |
| AutoindexTaskのHTML生成量 | [2026-11-19採用案](../Requirements.md#autoindex-html-budget): HTML生成のstepで固定HTML/表示名/hrefを含む実追加バイト数が8,192以内となり、変換後のバイト数を数えることを確認。固定HTML/項目/HTMLエスケープ/percent-encodingの途中で中断・再開し、重複/欠落のない完成HTMLを得られることを確認。8 KiB目で完成ならCOMPLETED、未完成ならIN_PROGRESSとする。空/長い名前とURL/非ASCII/大量項目/8 MiB上限付近を試し、先に全体を一括変換せず段階的に生成する。並行GET/CGIの応答と本文拡張/CPU/保持メモリを測定する |
| AutoindexTaskのキャンセルと破棄 | [2026-11-19合意](../Requirements.md#autoindex-cancel-api): void cancel()を未開始/列挙/ソート/HTML生成/完了/失敗/取得済みで呼び、保持DIR*をNULLにしてclosedirを一度だけ行うことを確認。失敗でも再試行せず、列挙後に閉じたDIR*も再closeしない。名前/番号列/作業領域/HTML/パスとURL/継続位置を解放または初期化し、未開始へ戻す。再cancel/デストラクターで二重closedirと外へ出る例外を防ぎ、取得済み本文とディレクトリ内のファイルを維持する。cancel後のtakeBodyはfalseで出力不変とする。closedir失敗ログと異常経路への組み込みは後続設計で具体化する |
| AutoindexTaskの内部状態 | [2026-11-19合意](../Requirements.md#autoindex-phase): privateなPhaseのIDLE/ENUMERATING/SORTING/BUILDING/DONE/ERROR/TAKENで生成段階と終端状態を管理することを確認。初期/cancel後IDLE、start成功でENUMERATING、列挙終端とclose成功でSORTING、ソート完了でBUILDING、全工程成功でDONE、本文取得でTAKEN、通常の開始/生成失敗でERRORへ進む。生成段階の切り替えでIN_PROGRESS、完成でCOMPLETEDを返し、cancelは全状態からIDLEへ戻る。別のタスク状態や完了/失敗/取得済みboolを重ねず、公開StepResultと区別し、DIR*の二重closeを防ぐ |
| AutoindexTaskのstart再呼び出し | [2026-11-19合意](../Requirements.md#autoindex-start-state): IDLEで開始でき、ENUMERATING/SORTING/BUILDING/DONE/ERROR/TAKENでは500を返してPhase/DIR*/名前/番号列/作業領域/HTML/継続位置/入力指示を維持することを確認。opendir/closedir/本文破棄を行わず、拒否の500でERRORへ変更しない。通常の開始失敗と生成失敗後も含め、cancelでIDLEへ戻してから再利用できることを確認する |
| AutoindexTaskのstep状態別契約 | [2026-11-19合意](../Requirements.md#autoindex-step-state): ENUMERATING/SORTING/BUILDINGでだけ該当段階を進める。DONEはCOMPLETED、ERROR/IDLE/TAKENはFAILEDを返し、再呼び出しでもPhase/完成本文/保持データが不変で、readdir/ソート/HTML生成/closedirがないことを確認。IDLE/TAKENでERRORへ変更せず、DONEのCOMPLETED再返却でもtakeBodyは一度だけ成功する。cancel後はIDLEとして扱い、失敗後の再stepで同じ資源を重ねて回収しない |
| AutoindexTaskの通常の生成失敗時の回収 | [2026-11-19合意](../Requirements.md#autoindex-step-failure): 列挙失敗/closedir失敗/HTML上限超過などで、stepが保持DIR*をNULLにして一度だけcloseし、名前/番号列/作業領域/HTML/パスとURL/継続位置を解放または初期化してERRORでFAILEDを返すことを確認。ソート/HTML生成中は列挙後にclose済みで再closeしない。Clientは500を共通準備へ渡し、close失敗でも500を維持する。再stepはERRORを維持し、cancel/破棄で二重回収せず、cancel後はIDLEへ戻る。bad_allocは通常FAILEDへ変換せず既存の接続終了を使う。ログ方針と例外経路は後続設計で具体化する |
| AutoindexTaskの通常の開始失敗時の回収 | [2026-11-20合意](../Requirements.md#autoindex-start-failure): IDLEからの通常の開始失敗で、取得済みDIR*があればNULLにして一度だけclosedirし、パス/URL/名前/番号列/作業領域/HTMLを解放して継続位置を初期化し、ERRORで元の404/403/414/500を返すことを確認。回収中のclose失敗でも元コードを維持し、入力指示を変更しない。再cancel/破棄で二重closeを防ぎ、cancel後に再利用できることを確認。IDLE以外のstart拒否では状態/資源維持の既存契約を適用する。bad_allocとログ/例外経路の詳細は既存方針と後続設計に従う |
| AutoindexTaskのコピー禁止 | [2026-11-20合意](../Requirements.md#autoindex-noncopyable): privateなコピーコンストラクター/コピー代入演算子を宣言して定義せず、外部からのコピー/代入を拒否することを確認。Clientが値メンバーとして所有し、既存のClientのコピー禁止とEventLoopのポインター管理でDIR*の所有者を一つに保つ。Clientと一緒に生成/破棄し、指示のAutoindexRequestはDIR*を持たないことを確認する |
| Clientの一覧生成中の基本遷移 | [2026-11-20合意](../Requirements.md#client-autoindex-state): AUTOINDEX_REQUIREDからstart成功でGENERATING_AUTOINDEX、IN_PROGRESSで同状態維持、COMPLETEDでtakeBodyから200/text/html応答を生成してprepareResponseへ渡すことを確認。開始失敗は返却コード、生成失敗はタスク内回収後500を共通準備へ渡す。成功時はWRITING_RESPONSEへ、設定error_pageを読むエラーはREADING_FILEへ進み、Router再実行/一覧再生成を行わない。生成段階はAutoindexTaskのPhaseに集約する。期限/進行順序は共通規則を適用し、切断/例外と契約違反時の具体的な経路は後続設計で定める |
| AutoindexTaskの共通期限/進行規則 | [2026-11-20合意](../Requirements.md#autoindex-loop-timeout): start直前から列挙/ソート/HTML生成の全段階を通算60秒で計り、opendir/準備を含めて段階切り替えで延長しない。開始周のstep省略、各周最大1回/poll待ち0/イベント0件でも進行、全体期限確認後のイベント優先を確認。start成功直後とstep前/IN_PROGRESS後の期限確認、COMPLETED優先、通常の失敗500、超過時cancel後500、計時失敗cancel後CLOSINGを検証する。error_pageの新しい60秒と送信60秒、各段階の境界付近での超過/失敗/並行GETとCGI/資源回収も確認する |
| AutoindexTaskの完成時の保持データ解放 | [2026-11-20合意](../Requirements.md#autoindex-success-release): HTML完成/先行段階/closedir成功後に完成本文だけを保持し、名前/番号列/作業領域/パスとURL/不要な継続情報を解放または初期化してDONEでCOMPLETEDを返すことを確認。空/1項目/大量項目でtakeBodyが初回だけ正しい本文を渡し、DONEの再stepで再生成や解放済みデータ参照がなく、cancel/破棄で二重回収しないことを確認。完了時の解放時間/並行GETとCGIの応答/ピークメモリ/回収量も測定する。例外経路は既存方針と後続の具体化に従う |
| AutoindexTaskの名前一覧コンテナ | [2026-11-20合意](../Requirements.md#autoindex-name-container): std::deque<std::string>で名前を所有し列挙順に末尾追加し、ソート中の並び/内容を維持することを確認。番号で対応名を参照して比較し、整列済み番号からHTML生成して、ソート用の名前複製を行わない。大量項目/長い名前/共通接頭辞/非ASCIIで順序と完成HTMLを確認し、deque領域確保/管理領域拡張/名前コピーと解放の時間/CPU/ピークメモリを測定する。成功/失敗/cancel時の解放と本文維持も各既存契約に従う |
| AutoindexTaskの番号列とマージ用作業領域 | [2026-11-21合意](../Requirements.md#autoindex-index-containers)に従い、列挙とclosedirの成功後に空の2本へ項目数分をreserveし、両方のsizeが0のままであることを確認する。元の番号列と最初のマージ出力をpush_backで追加し、追加中の再確保がないことと、番号追加を比較と合わせて1 stepあたり256操作に含めることを確認する。複数周に分けても番号に欠落や重複がなく、正しい順序のHTMLを生成することを確認する。領域確保時の所要時間とピークメモリを並行負荷で測定する |
| AutoindexTaskのマージ出力先の再利用 | [再利用の契約](../Requirements.md#autoindex-merge-buffer-reuse)に従い、全N件を書いたパス末尾でswapし、2パス目以降はsizeをNに保って既存要素へ上書きすることを確認する。奇数件・2の累乗以外の件数、パス途中の中断と再開を試し、末尾まで欠落や重複がなく、最終パス後のマージ元から正しい順序のHTMLを生成することを確認する。上書きも256操作の上限に含める |
| 完成HTMLの応答への引き渡し | [本文の引き渡し](../Requirements.md#autoindex-response-body-swap)に従い、takeBodyで取得したHTMLがsetBodyBySwapでコピーなしに応答へ移り、成功後に入力文字列が空になることを確認する。既存本文がある応答でも旧本文を解放し、200/text/htmlの本文と実バイト数に対応するContent-Lengthを確認する。最大8 MiBの本文も対象とし、pendingResponse_への移譲は下記の契約で確認し、ヘッダーと本文の連結コピーは分離送信の試験で確認する |
| autoindex応答の所有権移譲 | [応答の所有権移譲](../Requirements.md#response-ownership-handoff)に従い、prepareResponseBySwapがステータス・説明文・ヘッダー・本文をpendingResponse_へswapで移すことを確認する。元のローカル応答を破棄しても送信する内容が有効で、最大8 MiBの一覧本文をこの受け渡しでコピーしないことを確認する。既存の送信準備に入り、prepareErrorPageを1回だけ呼ぶことと、200/text/htmlおよび本文の実バイト数に対応するContent-Lengthを確認する |
| ヘッダーと本文の分離送信 | [ヘッダーと本文の分離送信](../Requirements.md#split-response-send)に従い、ヘッダーを先に送り、短いsendでも欠落・重複なく本文へ移ることを確認する。空本文・本文を送らない応答、ヘッダーと本文それぞれの部分送信、切断と期限超過を検証する。各周のsend上限と送信準備からの通算60秒を守り、大きい本文の連結コピーを省けることを確認する |
| HttpResponseの送信用API | [送信用API](../Requirements.md#response-send-accessors)に従い、serializeHeaders()がステータス行・ヘッダー・終端の空行を生成し、getBody()が送信対象本文をコピーせず参照として返すことを確認する。通常の本文と空本文の実バイト数、204/304のContent-Length省略、205のContent-Length: 0と本文省略を検証する。本文を送らない場合にgetBody()が空を返し、Clientがヘッダー送信後に完了できることを確認する |
| HTTPメモリ/fdと負荷 | HTTP-72・74: 同時接続と大きい応答でピーク資源量を測定し、接続完了/失敗/期限後のfd・所有メモリ回収を確認。応答コピー/本文蓄積/error_page生成/serialize/reset等にbad_allocを注入し、fd・保持本文・送信バッファを回収してcloseすること、500/内蔵HTMLの再生成や追加確保で復旧を試みないこと、ほかの接続が継続し未捕捉例外で終了しないことを確認。Siege -bの空ページGETでAvailability >99.5%、長時間の継続でメモリの際限ない増加/接続ハングがないことを確認。未達なら独自上限なしを含む採用案を見直す |
| HTTP未完了要求 | HTTP-19: CL不足・チャンク本文途中・区切り途中・終端trailer途中の受信EOFは追加応答せず要求破棄/close。保存・DELETE・CGI起動を行わない。要求完成後のCGI半閉鎖はCGI-113を維持。タイムアウトの詳細試験はHTTP-68〜70に従う |
| HTTP静的ファイルの種類・失敗 | HTTP-20・21: 通常/空ファイルは200、FIFO等は開かず403。stat/accessのENOENT/ENOTDIR=404、EACCES/EPERM=403、ENAMETOOLONG=414、その他=500。index不在なら既存autoindex規則、存在して読み取り不可/非通常ファイルなら403。index.pyの既存CGI判定を維持 |
| HTTP静的ファイルの読み取り | HTTP-22: 事前確認後のopen/読み取り失敗は500。成功が確認されるまで200を送らず、失敗時の部分本文を送らない。NUL/改行を含む実際の本文とContent-Lengthのバイト数を照合。空ファイルは本文0/CL0 |
| HTTP静的ファイルのMIMEと名前 | HTTP-23・24: 固定表全拡張子、大文字JPG、拡張子なし/未知、index選択後の型を確認。内容推測・一律のUTF-8宣言なし。隠しファイル/.始まりのディレクトリも通常扱い。MIMEの比較がCGI拡張子判定に影響しないことを確認 |
| HTTP静的ファイルの限定仕様 | HTTP-25・26: Rangeの正常/不正/範囲外、列挙された各条件ヘッダーを付けても通常のGETと同じ処理。成功なら200/全体本文、存在/権限エラーは維持。206/416・条件判定による304/412・ETag/Last-Modified生成なし。CGI転送規則は維持 |
| HTTP autoindexの表示 | HTTP-27・28: 名前/リンクのみ、サイズ/日時/種類表示や項目ごとのstatなし。隠し項目を表示し.と..を除外、親リンクなし。空の一覧200。A.txt/a.txtや非ASCII名で元の名前のバイト順を確認。ディレクトリを優先せず、クリック後の通常ルーティング/301/権限判定を確認 |
| HTTP autoindexの変換 | HTTP-29・30: 空白・&・<・>・引用符・%・#・?・日本語を含む名前でHTMLとして解釈されず、リンクから元の対象へ到達。/docs/a%20b%26c.txtの例と%252Fを確認。正規化済みURLパスを再エンコードし、物理パス/queryを使わない。パス内のエンコードされた/でも正しいリンクを生成 |
| HTTP autoindexの失敗と上限 | HTTP-31・32: opendirの404/403/414/500、readdir途中/closedir失敗500、正常NULLとの区別、全経路のディレクトリ解放を確認。途中の一覧を200で返さない。HTML本文8,388,608バイトは受理、1バイト超過は列挙中に打ち切り500。エスケープ後の名前/リンク/固定HTMLを含め、HTTPヘッダーは含めない |
| HTTPアップロードの受付とパート | HTTP-33〜37: ブラウザ/curl -Fの1ファイルを受付。ファイル0/2個以上400、通常項目/空filenameは捨て、名前あり0バイトを保存。他形式/CT欠落/空415。boundary引用符/大小文字/1〜70文字、前後付加部分/行末空白を確認。不正/終端欠落/パートヘッダー8,192バイト超過400。NUL/改行/境界に似た文字列を保存。非空の要求CEや非binaryのパートCTEは415 |
| HTTPアップロードの名前と競合 | HTTP-38〜40: ../photo.jpgと逆斜線のパスは最後の名前だけ使う。空/./../制御文字400、空白/非ASCII/percentを保持。保存先はupload_store、URIはlocation選択だけ。同名ファイル/ディレクトリは409で変更なし。新規mode指定0600。本文上限は区切り/パートヘッダーも含め、超過413 |
| HTTPアップロードの完了と公開 | HTTP-41〜44: open失敗409/403/400/500、短いwriteの継続、write 0以下/close失敗500を確認。全体構文検査前に作成せず、保存/close成功後に200/保存名HTML。HTMLエスケープ/実長CLを確認。Location/GET公開先を自動設定せず、root/aliasで公開。CGIで実行されない。途中ファイル削除はHTTP-49の試験項目へ分離し、既存対象は削除しない |
| HTTPのその他のヘッダー | HTTP-11: 未実装のヘッダーも共通構文を通れば保持。通常の静的配信のX-Test重複はそれだけでは400にせず、CGI転送対象ならCGI-48の400。除外対象の重複を一律拒否せず、全行の順序・値を保持 |
| HTTP APIと寿命 | HTTP-59〜64: Clientから本文上限を生成時に渡す。NULを含むrecv断片をバイト数付きappendDataへ投入し、INCOMPLETE/COMPLETE/ERRORとgetErrorCode(正常0)を確認。ERRORからCOMPLETEを待たず応答、完成時routeは1回、終端後の追加入力で結果不変。getPathのデコード/正規化1回・生query・復号本文・重複保持を確認。serialize1回/swap後に元応答を破棄しても短いsendの続きが正しく送れ、Config/CGI APIと寿命を維持 |
| HTTP応答の形式 | HTTP-51〜56: HTTP/1.0/CRLF/Connection: close、最終本文のCL(文字数でなくバイト数、通常の空本文0)、既決定CT、204の本文/CL省略を確認。Webservの405 Allowを設定順に依存しないGET, POST, DELETE順で生成。301/302はLocation/空本文/CL0。Serverを生成せず、Date生成も暫定的に省く。CGI転送の既決定は維持 |
| HTTPエラー処理順と本文置換 | HTTP-57・58: 要求検証の早期拒否、locationのメソッド禁止405がreturn/ファイル存在404より先、確定エラーの保持を確認。400〜599にだけerror_pageを適用し、未指定/読み取り失敗では元コードの内蔵HTMLへ戻す。Allow等とCGI説明文を保持し置換後CLを生成。200〜399は置換せず再帰処理なし |
| HTTP DELETE | HTTP-45〜50: root/aliasで通常ファイルを選びstd::removeで削除、成功204/本文とCLなし。ディスク上の対象名の消失・削除後GET404・再DELETE404を確認。空/非空ディレクトリとその他種別は403で対象を残し、index/再帰削除なし。削除失敗ENOENT/ENOTDIRは404、EACCES/EPERM/EROFSは403、ENAMETOOLONGは414、その他500を必要に応じ失敗注入で確認。読み取り不可でも親ディレクトリ等の削除権限があれば削除でき、ファイルのW_OKを不要な条件にしない。権限試験は非rootで実施。location禁止405/AllowとCGI禁止は維持 |
| 保存失敗後の削除 | HTTP-43・49・50: write/close失敗を注入し、保存用fdの後始末後に今回新規作成した途中ファイルだけをstd::removeで削除することとfdの回収を確認。後始末の成功/失敗とも元の500を維持し、削除失敗ではパスをログへ記録。読み書き失敗のerrnoによる分岐なし、削除の結果との混同なし。無期限の再試行をせず、409等の既存対象へ触れない |
| Expect | HTTP/1.1のExpectはヘッダー受信後に417(先に確定した400/413を優先)とし、100応答を送らない。HTTP/1.0のExpectは値によらず無視する(CGI-76) |
| Configのパス | rootはURI全体を追加し、aliasはlocationのprefixを置換する。同じlocationで両方を明示した場合は記述順によらず起動エラー(C-08)。aliasはディレクトリ対応のみとし、prefixと値の末尾 `/` が片方または両方欠けていれば自動補完せず起動エラー(C-09)。相対パスは設定ファイルのディレクトリ基準。error_pageは400〜599だけで指定ファイルを読み、未指定・読み取り失敗時は元のステータスの内蔵ページ。100〜399には適用しない(C-20) |
| Configの省略 | index省略でindex.html、autoindex省略でoff、ボディ上限省略で1 MiB、配信locationのroot/alias両方省略で設定ファイル基準のhtml。alias明示時はrootを補完しない。明示値を優先し、不正な明示値は既定値で救済しない。listen省略時は起動権限に関係なく0.0.0.0:8000(C-39)。明示値を優先し、複数serverの省略による同一listen先はC-12で拒否。bind失敗時に別ポートへ変更しない。upload_store省略時はWebservによるアップロード保存を行わず、保存先を自動補完しない。cgi_extension省略時はCGIを実行せず、拡張子とPythonのパスを自動補完しない(C-07)。CGI用locationでupload_storeを省略してもPOSTボディをCGIへ渡せる |
| Configのlisten書式 | IPv4:portを受理する(C-10)。IPv4の各数値0〜255、ポート1〜65535の境界を確認し、範囲外・構成要素の欠落・ポートのみ・ホスト名・IPv6・追加オプションは起動エラーにする。書式の受理と実際のbind成功は区別する |
| Configのlisten指定回数 | 1server内でlistenが0回なら既定値、1回なら指定値を使う(C-11)。2回以上は同じ値でも異なる値でも起動エラー。serverを分けた8080/8081で異なるコンテンツを配信できることを確認する |
| Configのlisten競合・仮想ホスト | 同一IPv4:port、同じポートでの0.0.0.0/特定IP、listen省略で生じる競合は起動エラー(C-12)。異なるポート、異なる特定IP同士は設定として受理する。server_name指定は起動エラー。Hostでサイトを切り替えず、HTTP/1.1のHost欠落・重複・不正は引き続き400 |
| Configのボディ上限 | 正の十進整数のバイト数を受理し、省略時は1 MiB(C-13)。0・符号・小数・単位接尾辞・整数型の範囲超過は起動エラー。復号後ボディが上限と同じなら受理、1バイト超過なら413。multipartはファイル以外の区切り等もボディサイズに含める |
| Configの指定回数 | 通常の設定項目を同じブロックに2回書いたら、値が同じでも起動エラー(C-14)。別ブロックでの指定は可。同じserver内でerror_pageの異なるコード・locationの異なるprefixは受理し、同一コード・同一prefixは拒否する。allow_methodsの複数行を結合しない |
| Configのindex | ファイル名1個を受理し、省略時index.html(C-15)。複数候補・`/`を含むパス・`.` / `..` は起動エラー。index不在でも起動可。ディレクトリへのGETでindexがあれば配信/既存CGI判定、なければautoindexがonなら一覧・offなら403。ディレクトリ不在は404。読み取り不可のindexから一覧へ切り替えない |
| Configのautoindex | 小文字on/offの1引数だけ受理し、省略時off(C-16)。空・複数引数・ON/true/1は起動エラー。autoindex_format等の未対応設定も拒否する。onでもindexを優先し、index不在時はHTMLの一覧を返す。offでも読み取り可能な個別ファイルへのGETは配信する |
| Configのallow_methods | 大文字GET/POST/DELETEの空白区切り・重複なし・1個以上を受理する(C-17)。POST単独ならGETを自動追加しない。空・重複・小文字・カンマ区切り・HEAD/PUT等は起動エラー。記述順を変えても許可集合は同じ |
| Configのupload_store | 省略/offで保存無効、既存ディレクトリのパス1個で有効(C-18)。絶対・設定ファイル基準の相対パスを確認。空・複数引数・不在パス・通常ファイルは起動エラーとし、自動作成しない。GETの公開先を自動設定しない。保存失敗や命名規則の試験はHTTP-33〜44に従う |
| Configのcgi_extension | 小文字offまたは拡張子＋Python絶対パス1組を受理(C-19)。.py/.cgi/.py3を受理し、.pyと.PYを区別。py/./*.py/.tar.py、不足/余分な引数、複数組、相対パスは起動エラー。省略/offでCGI無効。別の拡張子でもPythonで実行する |
| Configのerror_page | server内の100〜599の3桁コード1個＋パス1個を受理(C-20)。100/599を受理し、099/600・引数不足/余分・複数コード・=200等は起動エラー。同一コード重複は拒否。400〜599の応答で指定ファイルの不在/読み取り失敗は元コードの内蔵ページ。100〜399の設定は応答に適用せず、200の通常/CGI成功本文や301/302のリダイレクトを変更しない。204等の本文禁止を維持 |
| Configのreturn | location内の301/302＋固定HTTP(S)絶対URLを受理(C-21)。ホストなしURL・相対URL・他コード・不足/余分な引数・変数は起動エラー。設定URLをLocationで返し、要求パスやqueryを自動追加しない。移動先の取得や本文のConfig指定は実装しない |
| リダイレクト専用location | returnのみなら配信パスなしでConfigを受理し、GETは指定した301/302、POST・DELETEは405とAllow(C-22)。allow_methodsでPOSTを明示すればPOSTもリダイレクトする。root / alias / index / autoindex / upload_store / cgi_extensionとの併記はoffでも起動エラー。記述順によらず拒否し、配信項目の既定値を補完・検証しない |
| locationの最長前方一致 | /aは/a・/a/file.txt・/abcに一致する(C-23)。/a/は/a/file.txtに一致し、/a・/abcには一致しない。/・/a・/a/b/を定義したとき/a/b/file.txtは/a/b/、/abcは/aを選び、設定順を変えても結果が同じ。URIのデコード・正規化は別途検証する |
| URIの分離とデコード | C-24: /files/hello%20world.txt?name=a+bではパスの%20だけを空白にし、queryはname=a+bのまま。/%61は/a、/a%2Fbと/a%2fbは/a/bとして照合。%252eを%2eまで戻し、二重デコードしない。パスの+を保持し、%3Fから生じた?で再分割しない。パスの%・%2・%GG・%00は400。正規化はC-25の確認例に従う |
| URLパスの正規化 | C-25: /a//b.txt・/a/./b.txt・/a/x/../b.txtは/a/b.txt、/a/../b.txtは/b.txt。/a/.・/a/b/..は/a/、/a/..は/。/../secret.txt・/a/../../secret.txtは400。file..txt・.hiddenは維持。/a/%2e%2e/b.txtを/b.txtとしてlocation選択し、末尾/も維持する。symlinkはC-26の運用前提に従い、検出・拒否を合格条件にしない |
| symlinkの運用前提 | C-26: デモ用の公開・CGI・アップロード領域の配下にsymlinkを置かないことを二人が確認する。Webservに検出・拒否機能は要求せず、起動時の自動検証項目にも追加しない。リンクが置かれた場合の領域外アクセスを完全に防ぐ仕様・試験とはしない |
| ディレクトリの末尾スラッシュ補完 | C-27: GET /docsが選択済みの配信locationで実ディレクトリに解決されるとき、/docs/へ301。?lang=jaは移動先でも保持する。/docs/でindexまたはautoindexへ進む。禁止GETは405が優先し、POST・DELETEにこの補完は適用しない。returnは設定URLに従う。/docs/のlocationを/docsへ特別に一致させない。ホスト・ポートはC-28、再エンコードとquery保持はC-29の確認例に従う |
| 自動補完URLのHostとポート | C-28: Host localhost:8080ならhttp://localhost:8080/docs/、Host localhostならhttp://localhost/docs/。1.0でHostなしなら接続のサーバー側IP/portを使用し、0.0.0.0やクライアントのアドレスを返さない。Host重複・不正は両版400、1.1のHost欠落も400。Forwarded / X-Forwarded-*を変えてもURLは変わらず、Configのreturnは固定URLを使う |
| Hostの受付範囲 | C-30: localhost・localhost:8080・127.0.0.1:8080・example.com・my-site.example.com・example.com.・EXAMPLE.COMを受理。前後SP/HTABは除去。空値・内部空白・同値でも重複・空要素・要素先頭/末尾のハイフン・http://localhost・localhost/path・user@localhost・localhost:・port0/65536・[::1]:8080・%表記は400。port1/65535を受理。DNSで名前の実在を調べない。HTTP/1.0のHostなしは引き続き受理 |
| HTTP要求の改行 | C-31: 1.0/1.1の要求行・各ヘッダー・空行をCRLFで受理し、単独LF・CRの次がLF以外なら400。受信断片末尾のCRではERRORにせず、次のLFで継続する。ボディ内の単独LF・CR・CRLFは保持。CGI出力はLF/CRLFを従来どおり受理し、HTTP応答ヘッダーはCRLF |
| HTTP要求のサイズ上限 | C-32: 1.0/1.1とも要求行は末尾CRLF込み8,192バイトまで、ヘッダーは各CRLF・終端空行込み合計32,768バイトまで受理。各1バイト超過で414/431。終端未受信のまま上限を超えてもエラーへ進む。要求行とボディをヘッダー合計へ数えず、ヘッダー＋ボディの一括受信/断片受信でも同じ結果。行別・個数の別上限は設けず、Configも増やさない |
| HTTP要求ヘッダーの折り返し・空白 | C-33: 1.0/1.1とも継続行・最初のヘッダー行の字下げ・SP/HTABだけの行・Hostのコロン直前SP/HTABは400。Host:localhostとHost: localhostは同じ値。値の前後SP/HTABを除去し、X-Noteの値の内部の空白/タブとボディは保持。空のCRLF行だけでヘッダーを終了する |
| HTTP要求ヘッダー名 | C-34: 1.0/1.1とも大小文字の異なる名前を個別に送って同じ名前として読む。ASCII英数字とRFC tokenの全記号を受理。空名・名前内の空白/非ASCII/スラッシュ・コロンなしは400。名前をASCII小文字で保持し、値のAbC123や値内のコロンは保持。重複はHostのC-28・C-30とContent-LengthのC-35に従い、CGI転送の重複はCGI-48・75〜83に従い、通常HTTP処理の重複はHTTP-07〜11に従う |
| 要求のContent-Lengthの重複 | C-35: 1.0/1.1とも正常な1行は受理。同値/異値の2行・名前の大小文字違いの2行・10, 10や10, 20などカンマ入りは400。別受信に分かれた重複も検出し、ボディ完了を待たずエラー応答へ進む。結合や先勝ち/後勝ちで通さない |
| 長さ指定のないPOST | C-36: 1.0/1.1ともCL/TEが両方なければ空ボディでヘッダー終端時に受信完了し411にしない。Content-Length: 0も空。CGIへ進む場合はstdinがEOFになる。正のCL/chunkedは従来どおり受信し、不正値を省略扱いしない |
| 自動補完URLの再エンコード | C-29: 内部パス/hello world・/a+b・/name?part・/literal%20を、それぞれ/hello%20world/・/a%2Bb/・/name%3Fpart/・/literal%2520/にする。ASCII英数字・-._~・区切り/は維持し、その他はバイト単位の大文字%XX。?name=a+bや?x=%20は元のまま付ける。元要求/literal%2520は1回デコード→出力時1回エンコードで/literal%2520/となる。Configのreturnは変換しない |
| Configの基本文法 | server/location・`;`・`{}`・空白/タブ/改行区切り・`#`コメントを受理する。改行位置を変えても同じ設定として読む。引用符・エスケープ・変数・include・正規表現・入れ子location・http/events等の未対応構文、`;`の欠落・未閉じブロックは理由を表示して起動を中止する。reload機能は設けない |
| Configの絶対パス取得（2026-10-08仮決定・未検証） | [05のPWD方式](05_config_agreement.md#config-pwd-resolution): 相対指定・引数なし起動ではPWDと現在ディレクトリのstatによる同一性を確認する。PWD欠落/空/非絶対パス/不一致・stat失敗で、絶対パス指定を案内して起動エラーになること。絶対パス指定ならPWD欠落/不正でもこの理由で拒否しないこと。設定内の相対パスが設定ファイルのディレクトリ基準になり、親プロセスのcwdを変更しないことをLinux実環境で確認する |
| Config文法の細則 | C-37: 空/コメントだけの設定・locationのないserver・ブロック後の`;`・ROOT/Root・特殊location・if/rewrite/try_filesは起動エラー。server/location各最低1個を満たす設定を受理。同じ階層内の設定順やlistenとlocationの前後を変えても同じ意味。空locationは既定値を補完する。重複拒否とパス検証規則は別途適用する |
| Configのlocation prefix | C-38: /・/images/・/cgi-bin/・/v1.0/・/.hidden/・/file..txtを受理。先頭/なし・連続//・単独./..要素・query・%エンコード・非ASCIIを起動時に拒否し補正しない。/Images/と/images/は別prefix。location /なしも受理しマッチなし404。要求の/images/hello%20world.pngは既存規則で処理。alias末尾/制限は維持 |
| Configの起動時ファイル検証 | C-40: root/alias/upload先が不在・通常ファイル・stat失敗なら起動エラー。既定rootにも適用し、conf/default.confの空locationはconf/htmlを検査。Pythonが不在・非通常ファイル・X_OK失敗なら起動エラー。index/error_page不在だけでは起動を止めない。ディレクトリへの追加access検査・自動作成・配下全件検査は行わない。redirect専用locationの配信パスは検証せず、起動後の消失/操作失敗も処理する |
| Configの設定組み合わせ | C-41: 配信POSTでupload/CGI両方無効、同じlocationで両方有効、CGI有効でDELETE許可は起動エラー。省略/offを無効として判定し、記述順を変えても同じ結果。POST許可のreturnは受理(C-22の併記禁止は維持)。GETのみのCGI、POST不許可のupload有効設定は受理し、POSTを自動追加しない。受理済みCGI用locationへのDELETE要求は405 |
| Configの型・API | C-42: parseが既定値/パス/検証済みConfigを返す。不正設定はruntime_errorをmainで扱いlistenしない。size_t範囲超過を拒否し、upload無効は空、CGI無効は2文字列とも空、有効は1組。getterが可変参照を返さず、Configより先に参照元が破棄されることを確認 |
| 許可メソッドの省略 | allow_methods省略時はGETのみ許可し、POST・DELETEは405。明示した場合はその許可リストを使う。CGIを有効にしてもPOSTは自動許可せず、GET/POSTを明示したCGI用locationでもDELETEは405。CGI・アップロード保存の有効化は別設定として確認する |
| CGIの実行対象 | CGI有効locationと設定拡張子で実在する通常ファイルを選び、設定した絶対パスのPythonをexecveで起動する。GET/POSTと、indexに選ばれたindex.pyを実行する。CGI用locationのDELETEは405で、CGIファイルを削除しない |
| CGIのスクリプト特定 | CGI-25: test.pyが通常ファイルなら、/cgi/test.py/extra/a.pyでもtest.pyを実行し、PATH_INFOは/extra/a.pyとする。tools.pyがディレクトリなら実行対象にせず、/cgi/tools.py/test.pyでは配下の通常ファイルtest.pyを選ぶ。test.py.bakは設定拡張子.pyに一致するスクリプトとして選ばない |
| CGIの実行パス・cwd | CGI-26: /srv/cgi/tools/test.pyを絶対パスで指定して実行し、cwdが/srv/cgi/toolsになることを確認する。open("data.txt")で同じディレクトリのファイルを読めること、PATH_INFOを変えてもcwdが変わらないこと、親サーバーのcwdが変わらないことを確認する |
| CGIのREQUEST_METHOD | CGI-27: GET要求ではREQUEST_METHOD=GET、POST要求ではREQUEST_METHOD=POSTが設定されることを確認する。変数の省略や空文字列への置き換えがないことを確認する |
| CGIのQUERY_STRING | CGI-28: /cgi/test.py?name=taro&x=a+bで値がname=taro&x=a+bとなり、?を含めず未デコードで保持されることを確認する。%20も保持する。クエリなしと末尾が?だけの場合は変数が存在し、値が空文字列となることを確認する |
| CGIのPATH_INFO | CGI-29: /cgi/test.py/extraでは/extra、/cgi/test.py/では/を渡す。/cgi/test.pyと/cgi/test.py?name=taroでは変数が存在し、値が空文字列となることを確認する。既存のデコード・正規化後の後続部分が渡されることも確認する |
| CGIのSCRIPT_NAME | CGI-30: /cgi/test.py/extra?name=taroでは/cgi/test.pyを渡し、PATH_INFO・クエリ・ディスク上のパスを含めないことを確認する。デコード・正規化後のURLパスを使うこと、/cgi/で設定されたindex.pyを実行する場合は/cgi/index.pyを渡すこと、変数が省略されないことを確認する |
| CGIのCONTENT_LENGTH | CGI-31: 本文name=taroと同じ内容のchunked本文で値が9となることを確認する。長さ指定なし・Content-Length: 0・空のchunked本文では変数が存在し、値が空文字列となることを確認する。文字数ではなくCGIへ渡す本文の実際のバイト数を使う |
| CGIのCONTENT_TYPE | CGI-32: 要求Content-Typeの値が前後SP/HTAB除去後に渡され、multipartのboundary等のパラメーターとその大小文字が保持されることを確認する。本文0バイトでもヘッダーがあれば設定し、ヘッダーなしでは変数が省略され、推測値や既定値が設定されないことを確認する |
| CGIのSERVER_PROTOCOL | CGI-33: HTTP/1.0要求ではHTTP/1.0、HTTP/1.1要求ではHTTP/1.1を設定することを確認する。HTTP/1.0で応答しても要求側の値が保持され、変数の空文字列・省略がないことを確認する |
| CGIのGATEWAY_INTERFACE | CGI-34: HTTP/1.0・HTTP/1.1、GET・POSTのCGI要求で固定値CGI/1.1が設定され、変数の空文字列・省略がないことを確認する |
| CGIのSERVER_PORT | CGI-35: 複数のlistenで、それぞれの接続を受け付けたポート番号が10進数の文字列として渡されることを確認する。標準ポートも省略しない規則を確認し、変数の空文字列・省略がないことを確認する |
| CGIのREMOTE_ADDR | CGI-36: 接続元のIPv4アドレスがドット区切りの10進数で渡され、ポート番号を含まないことを確認する。X-Forwarded-For等に異なる値があってもREMOTE_ADDRが変わらず、変数の空文字列・省略がないことを確認する |
| accept直後の接続先情報取得失敗 | [接続先情報の取得失敗時の契約](../Requirements.md#getsockname-failure)に従い、新規接続だけgetsocknameの失敗を注入する。該当fdを一度だけ閉じ、Client生成・poll登録・Router呼び出し・応答送信が行われず、既存の接続と以後の正常な新規接続が処理されることを確認する。成功時は実際のローカルIPv4とポートがConnectionInfoに入ることを確認する |
| CGIのSERVER_NAME | CGI-37: Hostのポート部分が除かれ、名前の大小文字が保持されることを確認する。HTTP/1.0でHostなしならgetsocknameで取得した接続のサーバー側IPを渡し、listenの0.0.0.0や接続元IPを代用せず、変数の空文字列・省略がないことを確認する |
| CGIのSERVER_SOFTWARE省略 | CGI-38: CGIへ渡す環境変数にSERVER_SOFTWAREが含まれず、固定値や空文字列で設定されていないことを確認する |
| CGIのHTTP_*変換 | CGI-39: 転送対象についてAccept→HTTP_ACCEPT、User-Agent→HTTP_USER_AGENT、X-Test→HTTP_X_TEST等の名前の変換を確認する。ヘッダー名の大小文字によらず同じ変換となり、値は前後SP/HTABを除去して大小文字を保持することを確認する。転送対象・重複・名前の検証はCGI-40〜49・75〜83に従う |
| CGIの専用変数と重なるヘッダーの除外 | CGI-40: Content-Type・Content-Lengthが専用変数の合意済み規則で渡され、HTTP_CONTENT_TYPE・HTTP_CONTENT_LENGTHが設定されないことを確認する。chunked入力と空本文についてもCGI-31・32の値・省略規則を維持する |
| CGIのTransfer-Encoding除外 | CGI-41: chunked要求のCGI入力が復号後の本文となり、HTTP_TRANSFER_ENCODINGが設定されないことを確認する。CONTENT_LENGTHとstdinのEOF通知は合意済みの規則を維持する |
| CGIのConnection除外 | CGI-42: 要求にConnection: closeまたはkeep-aliveがあってもHTTP_CONNECTIONが設定されないことを確認する。既存の1接続1要求・応答後切断の方針も維持する |
| CGIのその他の通信制御用ヘッダー除外 | CGI-43: Keep-Alive・TE・Upgrade・Proxy-Connectionの名前の大小文字によらず、対応するHTTP_*が設定されないことを確認する。要求自体の受付・検証はHTTP層の合意済み規則に従う |
| CGIのConnection列挙ヘッダー除外 | CGI-44: Connection: X-Testがある場合にHTTP_X_TESTを設定しないことを確認する。複数名のカンマ区切り・各名前の前後SP/HTAB・大小文字の違いも確認する。本文の受信処理と専用環境変数の規則は維持する |
| CGIのProxy除外 | CGI-45: 要求のProxyヘッダーの名前の大小文字によらずHTTP_PROXYが設定されないことを確認する。その他の検証を通る要求では、Proxyの存在だけを理由に拒否されないことを確認する |
| CGIの認証用ヘッダー除外 | CGI-46: Authorization・Proxy-Authorizationの名前の大小文字によらず、HTTP_AUTHORIZATION・HTTP_PROXY_AUTHORIZATIONが設定されないことを確認する。その他の検証を通る要求では、これらの存在だけを理由に拒否されないことを確認する |
| CGIの転送するヘッダー名の文字範囲 | CGI-47: ASCII英数字と-だけの名前に合意済みの除外規則を適用し、_やその他の記号を含む名前はHTTP_*へ転送しないことを確認する。X-TestとX_Testを同時に指定してもX_Testは転送対象にならず、名前の範囲だけを理由にHTTP要求が拒否されないことを確認する |
| CGIの転送対象ヘッダー重複拒否 | CGI-48: 実行可能なCGIへの要求で、転送対象のX-Testを異なる値・同じ値・名前の大小文字違いで2行指定し、400でCGIが起動されないことを確認する。結合・先勝ち・後勝ちが行われないことを確認する |
| CGIのHTTP_*空値省略 | CGI-49: X-Testの値が空・SP/HTABだけ・ヘッダーなしの場合はいずれもHTTP_X_TESTが設定されないことを確認する。非空値は前後SP/HTAB除去後に保持する。空値を含む複数行はCGI-48で400となることも確認する |
| CGI用環境の構築 | CGI-50: 起動元の環境に専用の確認用変数・PATH・HOME・PYTHONPATH・HTTP_PROXY・SERVER_SOFTWARE等を設定しても、execveへ渡すenvpにコピーされないことを確認する。親のREQUEST_METHOD等で要求から生成した値が上書きされず、合意済みの設定・省略規則が維持されることも確認する。Python自身が起動後に設定する変数とは区別する |
| CGIのstderr | CGI-51: 正常なCGI応答をstdoutへ、確認用メッセージをstderrへ出して正常終了するスクリプトで、メッセージがWebservのstderr出力先に届きHTTP応答へ混ざらず、stderr出力だけで502にならないことを確認する。異常終了や不正なstdoutは既存の規則で502となることも確認する |
| CGI専用応答ヘッダーの重複拒否 | CGI-52: Content-Type・Status・Locationをそれぞれ2行出力した場合に502となることを確認する。異なる値・同じ値・名前の大小文字違いでも拒否し、結合・先勝ち・後勝ちが行われないことを確認する。各ヘッダーが最大1行の正常な文書応答とLocationのみの応答は既存の規則で処理する |
| CGI出力ヘッダーの折り返し・終端 | CGI-53: LF/CRLFの両方で、SP/HTABで始まる継続行・最初のヘッダー行の字下げ・SP/HTABだけの行が502となることを確認する。完全に空の行で終端し、その後の本文の行頭空白・タブ・改行は保持することを確認する |
| CGI出力ヘッダーのコロン直前の空白 | CGI-54: ヘッダー名とコロンの間にSPまたはHTABがある出力は502となり、除去して受理されないことを確認する。コロン直前に空白がない正常な出力は既存の規則で処理され、本文の空白には影響しないことを確認する |
| CGI出力ヘッダー名の文字範囲 | CGI-55: 空でないASCIIのtoken名が検査を通り、空の名前・名前内の空白・制御文字・非ASCII文字・スラッシュ等の不正な名前が502となることを確認する。X-Test・X_Test等の有効な名前を補正・拒否しない。出力ヘッダーの転送可否は別の規則で判定する |
| CGI出力の基本構文・改行 | CGI-56・57: コロン欠落・終端空行欠落・単独CRで502。LF/CRLFと混在は受理し、本文のコロンや改行は保持する |
| CGI出力の値・空値 | CGI-58・59: 前後SP/HTAB除去、内部の空白・大小文字・非ASCIIバイトの保持、HTAB以外の制御文字・DELの502を確認する。空値は省略として処理するが、CGI-52の重複は空値を含んでも502。空Content-Typeの通常文書応答は502となる |
| CGI出力のContent-Type検証 | CGI-60: type/subtypeとtoken・quoted-stringを含む正常なパラメーターを受理し、不正な構文は502。未知の正常なメディアタイプも受理し、本文からの推測・変換は行わない |
| CGI出力のStatus検証 | CGI-61: ASCII数字3桁の100〜599がコードの範囲検査を通り、099・600・4桁・非数字は502。コード直後のSP必須、空の説明文の受理、説明文の使用、通常文書応答のStatus省略・空値時の200を確認する。1xxだけで終了するCGIは502になることも確認する(CGI-74) |
| CGIのLocation値の検証 | CGI-62: 任意のフラグメント付き絶対URIの一般構文とHTTP/HTTPS以外のスキームも検証し、空白・不正な%表記等は502。値の大小文字・%表記・パス・クエリ・フラグメントを保持し、デコード・正規化や宛先の存在確認はしない |
| CGIのリダイレクト形式 | CGI-63〜65: Location単独かつ空本文は302。Location・Status・Content-Typeの形式では301・302・303・307・308を反映し、空本文も受理する。欠けた必須ヘッダー・対応外のコード・不完全な組み合わせは502となり補完されないことを確認する |
| CGIの内部・相対Location | CGI-66: /other・other・//example.com/等のスキームのない非空Locationが502となり、内部再処理・絶対URIへの補完・外部302への変換を行わないことを確認する |
| CGI出力のContent-Length | CGI-67・68: 0・先頭ゼロを受理し、符号・カンマ・内部空白・桁あふれ・本文との不一致は502。HTTP応答の長さは実際に送る本文のバイト数から生成する |
| CGI出力全ヘッダーの重複 | CGI-69: 専用3種類・Content-Length・その他・除外対象・Set-Cookieを含め、大小文字違い・同値・空値の重複が502となり結合されないことを確認する |
| CGI出力ヘッダーの転送・除外 | CGI-70〜72: 接続用ヘッダーとConnection指定名を除外し、専用4種類を指定した場合は502。非空Transfer-Encodingは502。その他の正常なヘッダーは値を保持して最大1行転送し、Webserv側で重複生成しない |
| CGIの本文禁止応答・1xx | CGI-73・74: 204・205・304の非空本文は502。空本文なら204・304はHTTPのContent-Lengthなし、205は0。CGI側の長さ指定は先に実長と照合する。コードの数値検査を通る1xxも、唯一のCGI応答として終了したら502とする |
| CGI入力ヘッダーの最終転送規則 | CGI-75〜77: 規則を満たす任意の名前が転送され、HTTP_TRAILER・HTTP_EXPECTと本文後のtrailerが追加されないことを確認する。HTTP/1.0のExpectは任意の値で無視し、HTTP/1.1では既存の417でCGIが起動されない |
| CGI要求Connectionの検証・複数行 | CGI-78: 全行のカンマ区切りの名前を大小文字を区別せず除外し、空要素は無視、不正な非空tokenは400。本文受信と専用変数を変更しないことを確認する |
| 要求ヘッダー値・CGI環境のNUL | CGI-79・83: 除外対象を含む要求ヘッダーでHTAB以外の制御文字・DELは400。非ASCIIバイトは保持し、本文に同じバイトがあってもこの検査で拒否しない。環境に渡す実NULは起動前400、クエリの文字列%00はそのまま渡す |
| CGI要求のContent-Type・除外対象の重複 | CGI-80・81: 単一の空Content-Typeは空のCONTENT_TYPE、ヘッダーなしなら省略。非空の値は保持して渡し、同値・空値の重複は400。転送除外対象の重複を一律に拒否せず個別の既存規則を維持する |
| CGI要求の内容符号化 | CGI-82: Content-Encoding: gzipが規則を満たせばHTTP_CONTENT_ENCODINGを設定し、gzip本文をWebserv側で展開しない。chunkedと併用した場合は転送符号化だけを復号して渡す |
| CGIエラーの受け渡し順序 | CGI-128・129: route即時応答では起動しない。start/CgiResult失敗で部分stdoutを使わず、成功時だけ解析する。正常なStatus: 404と実行失敗を区別し、504後の異常終了等で最初のコードを上書きしない |
| CGIのエラーページ適用境界 | CGI-130・132: 正常なCGIの404/599でもerror_pageを適用し、元コード・空も含むCGI説明文を維持する。200〜399では指定ファイルの読み取り・置換をしない。未指定・不在・読取失敗・非通常ファイルは内蔵HTML、空ファイルは空本文。再ルーティング・CGI実行・再帰的なエラー生成はしない |
| エラー本文とヘッダーの整合 | CGI-131・133: Content-Type=text/html・Content-Length=置換後の実長とし、CGI由来のContent-*・ETag・Last-Modified・Digest・Repr-Digestを除去する。Allow・WWW-Authenticate・Set-Cookie等は保持する。makeError/parse成功の各経路で置換は一度だけとなり、200〜399ではヘッダーを除去しない |
| CGI応答不能とfd設定失敗 | CGI-134: 切断時は応答を生成せず停止・回収する。送信失敗後に別応答を送らない。listenの設定失敗は起動失敗、acceptの設定失敗は接続を閉じ監視しない。CGIパイプの設定失敗は親500/子502となる |
| CGI起動失敗の子専用解放 | CGI-121・123: 複数の接続・他CGIが存在する状態でsignal/dup2/fcntl/close/chdir/execve失敗を注入する。子で一時資源・起動中CGI・他の管理データ・EventLoopを解放し、Webserv所有のヒープ割当てが未解放で残らないことを子プロセスも対象にしたメモリ検査で確認する。二重close/delete・解放後参照・追加確保/例外/ログ、kill/waitpid/shutdownが発生せず、親の接続・他CGIが継続し、親が失敗した子を502として回収することを確認する |
| CGI起動・計時の代替案の検証 | CGI-120〜127は現時点の採用案。子のsignal/dup2/fcntl/close/chdir/execve失敗を注入し、mainへのreturnと子専用の明示的解放で親用の応答・停止/回収・兄弟CGIへの操作・出力の重複が起きず、親が502として回収できるか確認する。/proc/uptimeの読取不可・読取/解析失敗・値の逆行は500で打ち切る。無負荷/実負荷で10秒の検出精度、静的配信への影響、追加I/O負荷を確認する。問題があれば代替方法を見直し、理由・影響・検証結果を関連資料へ記録する |
| CGIの停止と回収 | CGI-107〜111・118: 正常完了でEOFと回収の両方を確認し、切断/上限超過/期限超過では停止後に回収する。回収待ちも静的配信を継続し、既存の504等を上書きしない。SIGINT時も全直接の子を回収する。waitpidの失敗・close一度だけを注入試験で確認する |
| CGIパイプ通知と古いイベント | CGI-112・115: stdoutのPOLLIN/POLLHUPと残データを読み切る。各fd1回のI/O・部分書き込みの継続・I/O失敗502を確認する。同じpoll結果の後続通知で閉じたfd/削除済み管理データ/再利用fdを操作しない |
| CGI待機中の半閉鎖と追加送信 | CGI-113・114: 完成済み要求の後に送信側だけをshutdownしても応答を受け取れる。追加データを蓄積せず次要求を実行しない。切断検出時はCGIを停止・回収して結果を送らない |
| CGIのfd継承とSIGPIPE | CGI-116・117: Python側に他Clientのソケット/他CGIのパイプが残らず、stdin/stdout/stderrは使える。exec失敗時も資源を後始末する。親はSIGPIPEを無視し、PythonへはSIG_DFLを引き継ぐ |
| CGIデータの寿命・所有権 | CGI-99〜106: start後にCgiRequestを破棄しても本文を最後まで渡せる。本文は書き込み完了/打ち切りで解放し、結果取得は一度だけ。parseCgiOutput後にCgiResultを破棄してもHTTP応答が有効。Client破棄後に古いポインターや再利用されたfdへ結果を渡さない |
| CGI受け渡しAPI | CGI-91〜98: routeの即時応答/CGI要求と有効な出力引数、startが完了を待たないこと、takeResultが確定結果を一度だけ返すことを確認する。実行成功後でも不正出力はparseCgiOutputの502になる。getHeadersが重複を保持し、CGIのStatus説明文は空も保持する |
| CGIファイルパスと検査範囲 | CGI-84・85: root末尾の/の有無で同じ実行パスになること、aliasがprefixを置換することを確認する。test.py/extraはtest.pyだけを検査し、PATH_INFOの実在を要求しない |
| CGIファイル種別と権限 | CGI-86・87: .pyのディレクトリは実行せず既存処理、FIFO等は403。読み取り可能で実行ビットのない.pyは実行でき、R_OK失敗は拒否する。権限試験は権限を迂回しない実行ユーザーで行う |
| CGIファイルアクセス失敗 | CGI-88〜90: stat/access失敗のENOENT・ENOTDIRは404、EACCESは403、ENAMETOOLONGは414、その他は500。必要に応じ失敗を注入する。起動後のPython消失はexec失敗502、検査後の変化にもリトライしない |
| CGIとアップロードの分離 | locationと物理的な保存先を分け、アップロード先でCGIが実行されないことを確認する |
| CGI入力・パス | `/cgi/test.py`、`/cgi/test.py/extra`、`/cgi/test.py/extra/item`、`/cgi/test.py/extra?name=taro`のSCRIPT_NAME・PATH_INFO・QUERY_STRINGを確認する。POSTとHTTP/1.1 chunked入力は復号済みのボディをそのままstdinへ渡す。SERVER_PROTOCOLは受信した要求の版を使う |
| CGI出力 | Content-Type・空行・本文を解析し、通常の文書応答でStatus省略なら200、明示されたStatusはHTTPステータス行へ反映する。ヘッダーのLF/CRLFを受理し、本文中の空行を保持する。絶対URLのLocationだけなら302 |
| 出力の終端・長さ | stdoutのEOFと子の終了を両方確認してから応答する。Content-LengthなしでもEOFまで読み、本文の実長から生成する。指定値と実長が一致すれば成功、不一致なら502。204等はHTTP側の本文・Content-Length規則に従う |
| CGIのエラー | スクリプトなし404、読み取り不可403、pipe/fork失敗500、exec失敗・子の異常終了・不正なCGI出力502。失敗時に作成済みのfdを閉じ、子を回収する |
| CGI出力上限 | 1件あたりヘッダー8 KiB(8,192バイト)、ヘッダーを含むstdout全体8 MiB(8,388,608バイト)の境界を確認する。読み取り中の超過検出でCGIを打ち切り、後始末して502。入力ボディ上限は既存Configを使う |
| CGIタイムアウト | 起動から10秒で打ち切り504。継続出力でも期限を延長せず、EOF後に子が終了しない場合も対象とする。打ち切り後の子の異常終了で504を502へ変更しない |
| CGIの並行処理 | アプリケーション独自の同時実行数上限を設けず、件数を理由とした503や待ち行列を作らない。OS資源が利用できる環境で17本以上を件数だけで拒否しないことを確認する。OS資源不足によるpipe/fork失敗は500 |
| CGIとイベントループ | パイプ容量を超える入出力を並行処理でき、CGI実行中も静的配信できる。即時終了、クライアント切断、cwdからの相対ファイル読み込み、終了後にfdや未回収の子が残らないことを確認する |
| CGIのクエリ引数 | `?a+b`は拒否せずQUERY_STRING=`a+b`として渡し、コマンドライン引数へ展開しない |
| 不正なCGI出力 | 不正な値を推測・補正せず502。CGI-52〜74の構文・重複・数値・Statusの境界と本表の期待値に従う。Status: 404でも長さ不一致等なら404のerror_pageへ進めず502とする |
| 対象外のCGI応答 | CGI-62〜66: ローカル/相対/ネットワークパスのLocationは502とし内部で再処理しない。絶対URIのみ・空本文は302。Location・Status・Content-Typeを伴う形式は301/302/303/307/308のみ受理し、不完全な形式は502 |

## T2 nginx との挙動比較 【共, R:S, D:S, ★★】

- 同じ設定意図のnginx設定を用意し、同一リクエストでステータス/ヘッダー/ボディを比較する手順。
- 差が出やすい点: 末尾スラッシュ補完、405の`Allow`、404/403判定、autoindexの見た目、エラーページ、`Server`/`Date`、HTTP/1.1と1.0の応答差、`Connection` ヘッダー。
- 意図的な差異は仕様に「nginxと異なる点」として記録(評価時の説明材料)。

---

## 決定が必要な論点リスト(仕様作成フェーズで解消する)

### 外部制約・未解決の確認事項

| # | 論点 | 影響範囲 |
|---|---|---|
| Q1 | **関数制約とDateの暫定案**: C++98標準ライブラリとOS/POSIX等の外部関数を区別する(Requirements §4)。std::removeは2026-10-07のユーザー決定により使用可能とし、DELETEと途中ファイルの削除手段はHTTP-50で確定済み。_exit・clock_gettimeの使用不可はCGI-119の既決定を維持。Dateの暦日時取得手段はHTTP-56の別の設計判断として残し、std::removeの採用でDate生成まで確定したとは扱わない。経過時間はHTTP-70/CGI-124〜126の/proc/uptime採用案、ログはHTTP-73の日時省略を維持し、代替案は実環境で検証する | N7, C8, H8 |
| Q2 | read/write/recv/send後のerrno参照禁止は確定済み。stat/access/waitpidとstd::removeの失敗分類では合意済み規則を使う。std::remove自身の失敗直後のerrnoと、保存失敗のread/write後のerrnoを混同しない。残るaccept/poll等の失敗処理はN5で具体化する | N5 |
| Q3 | 評価で想定されるアップロードの手段(ブラウザのフォーム/curl) | C6 |
| Q4 | `Requirements.md` の「norminette不要」「Linux限定」の最終確認(キャンパスルール) | 全体 |

<a id="remaining-design"></a>

### rysatoのHTTP設計判断の合意状況(10項目)

以下はHTTP設計を10項目にまとめた管理単位。1/10〜6/10は2026-10-07に合意済み。7/10はstd::removeによる削除を含めHTTP-45〜50で合意済み、8/10は応答生成を合意済み(Date省略は暫定案)、9/10はHTTP API/データの保持を合意済み、10/10もHTTP-65〜74で合意済み(採用案を含む)。10項目の動作方針の判断は完了し、HTTP-50の削除手段も確定した。Dateは暫定、接続情報取得失敗時は新規接続だけを閉じる方針を採用済み、採用案の実環境検証は未実施。細かな判断を10件ずつまとめて承認する方式ではなく、1項目ずつ説明して合意する。rysato主担当8項目と共同2項目。Config・CGIの確定内容は再決定しない。

| 順番 | 状態 | 担当 | 決める内容 | 関連 |
|---|---|---|---|---|
| 1/10 | 合意済み | rysato | 要求行はHTTP-01〜05で合意済み。厳密な書式、メソッド、origin-form限定、版の検査、パス/query構文。実装・試験は未完了 | H1/H2 |
| 2/10 | 合意済み | rysato | 入力ヘッダーはHTTP-06〜11で合意済み。CLの数値、Content-Type/TEの単一行、TEの400/501、Connection共通検証、通常処理とCGI転送の区別。実装・試験は未完了 | H3/H4 |
| 3/10 | 合意済み | rysato | 本文/chunkedはHTTP-12〜19で合意済み。完成・逐次受信・サイズ/区切り・extension/trailer検査と読み捨て・付加情報上限・早期413・未完了EOF。タイムアウト詳細はHTTP-68〜70。実装・試験は未完了 | H4/H5 |
| 4/10 | 合意済み | rysato | 静的ファイルはHTTP-20〜26で合意済み。通常ファイル・失敗分類・実長の成功応答・固定MIME表・隠しファイル受付・Range/条件付き要求の範囲外。通常ファイルI/O等はHTTP-71・72の採用案。実装・試験は未完了 | C4/H10 |
| 5/10 | 合意済み | rysato | autoindexはHTTP-27〜32で合意済み。名前/リンクのみ・バイト順・隠し項目・HTML/URL変換・取得失敗・HTML本文8 MiB上限。on/offとindex優先は維持。実装・試験は未完了 | C5 |
| 6/10 | 合意済み | rysato | アップロードはHTTP-33〜44で合意済み。multipart単一ファイル・本文検証・basename・同名409・新規作成/失敗分類・200結果HTML・GET手動設定。途中ファイル削除と失敗時の扱いはstd::removeを使うHTTP-49・50で合意済み、I/O等はHTTP-71・72の採用案。実装・試験は未完了 | C6 |
| 7/10 | 合意済み | rysato | HTTP-45〜50で通常ファイルのみ・std::removeで削除・成功204・失敗分類・保存失敗時の後始末/ログを合意済み。空ディレクトリも削除前に拒否し、削除の戻り値と失敗直後のerrnoを確認する。実装・試験は未完了 | C7 |
| 8/10 | 合意済み(Date暫定) | rysato | HTTP-51〜58で基本形式・最終本文のCL/CT・405のAllow順・301/302の空本文・Server省略・エラー処理順/error_pageを合意済み。Date省略は暫定案で取得手段確認後に見直す。実装・試験は未完了 | H7/H8/H11 |
| 9/10 | 合意済み・関連APIの具体化中 | 共同 | HTTP-59〜64のうちHTTP-63を2026-11-15に更新。Routerの結果を3種類へ拡張しFileReadRequest出力引数を追加。Config・CGIプロセス側APIは維持。error_pageのprepareErrorPage/finishErrorPageとClientでの元応答保持は合意済み。makeError(code)は設定参照/本文生成/読み取りをせず共通準備経路へ渡す方針も合意済み。prepareErrorPageをClient側の共通送信準備で1回呼ぶ方針も合意済み。旧applyErrorPage/readErrorPageの廃止は合意済み。共通の入口はClient::prepareResponse(const HttpResponse&)として合意済み。実装・試験は未完了 | H4/N4 |
| 10/10 | 読み取り方式を変更・内部設計は継続 | 共同 | HTTP-71を2026-11-15に更新。全量読み取り後の送信は維持し、Client所有のFileReadTaskを各周最大1回・8 KiBで進める方式へ変更。ClientのREADING_FILEと静的ファイル読み取りの基本遷移、FileReadTaskの値メンバー保持、FileReadTask/Clientのコピー禁止も合意済み。読み取り期限・異常時の遷移・poll待ち・fd継承/解放・error_pageの異常時詳細遷移は未決定。書き込みはFileWriteTaskによる分割・4操作・内部状態・コピー禁止・ClientのWRITING_FILEと基本遷移まで合意済み。保存期限・異常経路と、一覧生成は2026-11-19に列挙/ソート/HTML生成の段階分割を採用し、所有/API/処理量などを具体化する。serializeの対策は未決定。その他の既存規則は維持。実装・試験は未完了 | N4/N6/N7/N9/N10 |

2026-11-23に接続情報の取得に失敗した場合の扱い(C-28・CGI-37、共同)を採用した。実装着手前の判断として挙げていたこの項目は完了し、結合試験で動作を確認する。Date省略(HTTP-56)は暫定案を使って実装を進め、変更する場合に再判断する。

送信位置の管理、EventLoopへの組み込み、周末削除、異常時の資源回収は、合意済みの動作を実装設計で具体化する。ファイル処理の期限と各周の処理量、/proc/uptime、[検証済みPWDを使う方式](05_config_agreement.md#config-pwd-resolution)は、実環境で検証し、見直しが必要になった時点で判断する。

2026-11-15の設計修正により、通常ファイルの読み取り方式・FileReadTaskの所有・Routerの3種類の結果を合意した。これは上記の初回合意を一部更新するもの。ClientのREADING_FILEと静的ファイル読み取りの基本遷移、FileReadTaskの値メンバー保持、FileReadTask/Clientのコピー禁止も合意済み。残る異常時の遷移/後始末・読み取り期限・bad_allocの具体的な捕捉位置と異常時の解放への組み込み・poll待ち・fd継承・error_page生成API/遷移と、アップロード保存/autoindex/serializeの扱いは[Requirementsの設計修正](../Requirements.md#file-read-design)で管理し、一つずつ決める。新方式の実装・検証は未着手。

HEAD/OPTIONS、Cookie/セッション、複数CGI言語は現在の必須設計の残件に含めない。CGIの採用案の動作確認、README・デモ・受入試験は別の実装/検証作業として管理する。

### 以前の論点IDとの対応

| ID | 現在の扱い |
|---|---|
| D1・D10 | Config合意済み。正本は05。listen競合拒否・仮想ホストなしも維持 |
| D2 | CGI APIとHTTPの公開API/データの保持(HTTP-59〜64)は合意済み。I/O・全体メモリ等はHTTP-65〜74で合意済み(採用案を含む) |
| D3 | 共通規則・要求行(HTTP-01〜05)・入力ヘッダー(HTTP-06〜11)・本文/chunked(HTTP-12〜19)は合意済み。実装・試験は未完了 |
| D4 | ConfigのGET/POST/DELETE限定は合意済み。通常静的配信のRange/条件付き要求はHTTP-25・26で範囲外と合意済み。RFC上の許容とチームの限定仕様を区別する |
| D5 | Expectは両版とも合意済み。HTTP/1.1は即時417(先行400/413優先)、HTTP/1.0は値によらず無視。100応答なし(CGI-76) |
| D6 | 要求行/ヘッダー・ボディ・CGIの既存上限とchunked付加情報(HTTP-17)は合意済み。バッファ/全体資源はHTTP-72の採用案 |
| D7 | 生成コードはHTTPの各合意に従う。応答生成はHTTP-51〜58、受信期限の408はHTTP-68で合意済み |
| D8 | アップロードの処理方式はHTTP-33〜44で合意済み。途中ファイル削除と失敗時の扱いはstd::removeを使うHTTP-49・50で合意済み、I/O/全体メモリはHTTP-71・72の採用案 |
| D9 | 早期エラー応答後の切断手順はHTTP-67の採用案 |
| D11 | CGI-01〜135の合意完了。CGI-120〜127の起動・計時方法は現時点の採用案で、実環境で検証し見直し得る。受入試験は未実施 |

---

## 進行計画(調査 → 仕様作成 → レビュー)

> 以下は**工数の相対バランス確認用**の初期見積もり。1日=実作業4h換算の仮定。rysatoの初期値はC11を含めていたため、必須範囲から外したCookie/セッションと今回限定したCGI範囲を踏まえ、実際の稼働に合わせて補正すること。

### 担当別の調査量(目安)

| 担当 | 主担当トピック | R合計(目安) | D合計(目安) |
|---|---|---|---|
| tasugiya | N1–N10、C8 | 約 30–40h | 約 25–35h |
| rysato | H1–H11、C1–C7、C9(C11は必須範囲外) | 初期値 約 40–55h・要補正 | 初期値 約 35–50h・要補正 |
| 共(重複) | N3–N5 の相互理解、H4境界、H7レビュー、C10、T1–T2 | 各自 +8–12h | 約 10h(合同) |

→ **rysatoの方が仕様の量が多い**(HTTP+設定+アップロード+CGI仕様)。均等化のため、**H7(ステータスコード表)の一部を、調査が軽いtasugiyaの余力に振る**、またはtasugiyaがN章を早く終えた段階でCGI(C8–C10)の調査主導を担う、といった調整を推奨。

### フェーズ

| フェーズ | 内容 | 成果物 |
|---|---|---|
| P0 確認(0.5日) | 外部確認事項(Q1–Q4)の確認依頼、本資料の担当調整 | Q1–Q4の回答メモ |
| P1 先行調査(★★★のみ) | tasugiya: N3–N5, N1–N2 / rysato: H1–H4, H6–H7, C1–C3 | 各トピック**1ページ要点メモ** |
| P2 共有(0.5日) | 重複部分(H4境界、N5方針、H7、C10)の相互レクチャー | 合意事項リスト(D2, D9 など) |
| P3 残り調査 | 必須範囲の★★/★のトピック。ボーナスは別途着手を決める | 要点メモ |
| P4 仕様作成 | 「規則/採用方針/受け入れテスト」の3点セットで `specs/` 等に記述。Config(D1)は合意済みとして仕様へ反映し、HTTP 10項目の合意・採用案を仕様へ反映し、Dateの暫定案・接続情報取得失敗時の扱い・採用案の検証を管理 | 仕様書一式、サンプル設定 |
| P5 相互レビュー | 互いの仕様を読み、実装前に矛盾(特にtasugiya↔rysatoの境界)を潰す | レビュー指摘ログ |

### 実装順序との対応(`Requirements.md` 実装順序 1–8)

| 実装フェーズ | 先に完了しておくべき仕様 |
|---|---|
| 1 ConfigParser / Hello World | C1, C2, N1, N2 |
| 2 EventLoop+Client | N3, N4, N5, N6, N7, N8 |
| 3 HttpRequestパーサー | H1, H2, H3, H4 |
| 4 Router+HttpResponse | H6, H7, H8, H10, C3, C4 |
| 5 エラー/autoindex/DELETE | H7, C5, C7(HTTP-45〜50) |
| 6 POST/chunked | H4, H5, C6 |
| 7 CGI | C8, C9, C10 |
| 8 タイムアウト/上限/負荷 | H9(1接続1要求), N7, N9, T1 |

**ポイント**: フェーズ7(CGI)の仕様は他と比べて結合が複雑なので、フェーズ4–6の実装中に仕様を詰めても間に合うが、**C10(非同期結果の返し方)はD2(シグネチャ)に影響するため、P4で先に確定**しておくこと。
