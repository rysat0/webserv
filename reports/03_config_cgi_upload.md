# 03 ルーティング・静的配信・アップロード・CGI

Routerは、解析済みの要求とConfigを使って、どの処理に進むかを決めます。即時応答で済む場合はHttpResponseを返し、ファイル処理やCGIが必要な場合は、Client側が次の処理を始めるための指示を返します。[RouterとClientの共有API](#cgi-api)

CGIの環境変数、入出力、失敗時の扱いは[CGIの決定表](../Requirements.md#cgi-decisions)を参照してください。HTTP/1.2〜1.9の要求にもHTTP/1.1の規則を適用し、SERVER_PROTOCOLには受信した表記を渡します。起動・計時の採用案は[C12](#cgi-details)にまとめています。C8〜C10には調査用の項目も含みます。

背景を調べるときは、NGINXの設定資料と、RFC 3875（CGI）、RFC 7578（multipart/form-data）、RFC 3986（URI）を参照します。Cookieの参考はRFC 6265です。

---

## C1 設定ファイル文法 【rysato, R:M, D:L, ★★★】

文法・既定値・起動エラーは合意済み(C-04〜07・37)。限定したserver/location構文を使い、NGINX全体との互換性・継承は持たせない。文法と設定例は [05](05_config_agreement.md#config-decisions) を参照する。

## C2 ディレクティブ一覧と検証規則 【rysato(主)/共, R:M, D:L, ★★★】

設定値・指定回数・既定値・起動時検証・組み合わせ・型/APIはC-01〜42で合意済み。一覧と詳細は [05](05_config_agreement.md#config-decisions) に一本化する。HTTP側の処理方式が未決定でもConfigの合意を未確定へ戻さない。

## C3 location マッチングとパス変換 【rysato, R:M, D:M, ★★★】

query分離→パスの1回デコード→正規化→文字列の最長前方一致→root/aliasによるパス変換の順に処理する(C-01・23〜25)。GETの末尾/補完、Hostの検証、補完URLの生成も合意済み(C-27〜30)。詳細と具体例は [05](05_config_agreement.md) を参照する。

symlinkを置かない運用前提はC-26、CGIのファイル・パス検証はCGI-84〜90に従う。通常静的配信はHTTP-20〜26で合意済み。アップロード・DELETEの検証の残る詳細は、各節と [04の残件](04_testing_and_workplan.md#remaining-design) で扱う。

## C4 静的ファイル配信 【rysato, R:M, D:M, ★★★】

**合意済み(HTTP-20〜26)**: 正本は [Requirementsの静的ファイル](../Requirements.md#http-static-files)。以下は理解のための要点。

- statで通常ファイル・ディレクトリを分け、FIFO/デバイス等は開かず403。空ファイルも200。ディレクトリはC-27→C-15の末尾/補完・index優先・未発見ならautoindexを維持する。存在するindexが読めない/通常ファイルでない場合は403とし、一覧へ切り替えない。選ばれたindexのCGI判定は維持する。
- stat/access(R_OK)の失敗は、ENOENT/ENOTDIR=404、EACCES/EPERM=403、ENAMETOOLONG=414、その他=500。事前確認後のopen/読み取り失敗は500。read等の後のerrno分岐は行わない。
- 読み取り成功を確認してから200を送り、実際の本文バイト数からContent-Lengthを作る。静的ファイルのバイト列は加工しない。通常ファイルI/O・保持・全体メモリはHTTP-71・72の採用案に従う。
- Content-TypeはHTTP-23の固定表を使う。MIMEの比較だけASCII大小文字非区別で、ファイルパス/CGI拡張子の比較を変更しない。
- 名前が.で始まる対象も通常どおり扱う。autoindexでもHTTP-27に従い隠し項目を表示する(.と..は除く)。symlinkを置かない運用前提と検出機能を省くC-26を維持する。statはリンク先を参照するため、この運用前提を外した場合のアクセス先を制限する機能ではない。[Linux stat](https://man7.org/linux/man-pages/man2/stat.2.html)
- 静的配信ではRange/条件付き要求を解釈せず、206/416・条件判定による304/412・ETag/Last-Modified生成を追加しない。Range無視はRFCで許容されるが、条件付き要求の一律省略にはRFC上の必須評価を外す判断も含む。課題向けの限定仕様とRFC全文準拠を区別する。CGIのヘッダー転送規則は維持する。

実装・受入試験は未完了。

## C5 autoindex 【rysato, R:S, D:S, ★★】

**合意済み(HTTP-27〜32)**: 正本は [Requirementsのautoindex](../Requirements.md#http-autoindex)。C-15・16・27のindex優先・on/off・HTML形式・末尾/補完を維持する。

- 各項目の名前とリンクだけを表示し、種類/サイズ/更新日時や項目ごとのstatは追加しない。隠し項目も載せるが.と..は除き、親へのリンクは作らない。空の一覧も200。掲載はアクセス成功を保証しない。
- 元の名前を大小文字を区別するバイト順で昇順にし、ディレクトリ優先や読み順によるソートをしない。
- 表示名はHTMLエスケープ、hrefは正規化済みURLパスと項目名をpercent-encodingして作る。物理パス/queryは使わない。ディレクトリへのリンクにも末尾/を追加せず、クリック後にC-27を適用する。
- opendir失敗は404/403/414/500へ分類、readdir途中/closedir失敗は500。readdirのNULLが正常終端かは直前のerrno=0と結果で判断する。列挙・生成・closeが成功してから200を返し、途中の一覧は成功応答にしない。
- 完成するHTML本文は8 MiBまで。列挙中からエスケープ後の長さを確認し、超過で打ち切り500。取得済み一覧を破棄してディレクトリを閉じる。ページ分割・件数上限・Config項目は追加しない。

実装・受入試験は未完了。全体メモリ/I/OはHTTP-71・72の採用案に従い、実負荷で検証する。

2026-11-17合意: 静的ファイル/error_pageの読み取りとアップロード保存は固定期限方式を使い、read/writeが進行しても期限を延長しない。Clientが各タスクのstart直前に開始時刻を記録し、開始準備/openを計測区間に含める。2026-11-18合意でerror_pageは自身のstart直前から新しい固定期限を設ける。error_pageの期限超過ではcancel後、元のステータス等を保持してfinishErrorPage(false, body)で内蔵HTMLを確定し、beginWritingResponseへ進む。2026-11-18に静的読み取り/保存/error_pageを各60秒の採用案とし、並行負荷試験で検証する。静的読み取り/保存の期限超過はClientがcancelでfd/部分本文/今回作成した途中ファイルを回収し、makeError(500)をprepareResponseへ渡す。error_pageは新しい60秒を使う。step前に期限確認し、超過なら呼び出さず用途別の回収/応答へ進む。step復帰後はCOMPLETEDを優先して成功経路、FAILEDは失敗経路、IN_PROGRESSなら時刻を再確認し超過時は回収/応答へ進む。start成功直後にも期限を確認し、超過ならcancel後に用途別の超過処理へ進む。start失敗は静的読み取り500、保存の返却コード、error_pageの内蔵HTMLという既存の失敗経路を使う。2026-11-18合意で進行可能なファイルタスクがある間はpoll待ち時間0とし、各周でイベント/期限を確認し各タスクを最大1回進める。タスクがなくなったら通常のpoll待ち時間へ戻す。イベント0件でもファイルタスクを進める。poll復帰後はソケット/CGIイベントを先に処理し、その後、生存中で継続する接続のファイルタスクを期限確認付きで進める。新規タスクは開始した周にstart/直後の期限確認を行い、最初のstepは次の周からとする。失敗からerror_pageへ移る場合や同じFileReadTaskの再利用も同じ契約に従う。終了確定時はClientをCLOSINGにしてタスクcancel/fd回収/CGI関連付け解除を行い、本体のdeleteとコンテナ除去はその周の最後にEventLoopが行う。後段はCLOSINGを対象から外す。2026-11-18合意でファイル処理の時刻取得/解析失敗や逆行はタスクcancel後CLOSINGへ進め、fd回収/CGI関連付け解除から周末削除へつなぐ。追加の500/error_page/内蔵HTMLを生成せず、保存成功済みファイルを保護する。2026-11-19合意でClientにfileTaskStartedThisRound_を持たせ、周頭にfalse、新規タスク開始時にtrueとする。後段のstepはtrue/CLOSINGを対象から外し、次周から進める。同じタスクのerror_page用再開始もtrueとする。2026-11-19合意で周頭リセット/poll/全体期限確認/ソケットとCGIイベント/ファイル処理/周末削除の順とする。期限処理で無効になったfdの古い通知を除外し、新規タスクは次周からstepする。操作APIと例外経路の詳細は未決定。[期限方式の正本](../Requirements.md#file-task-timeout-policy)を参照する。

2026-11-19合意: autoindexは列挙・ソート・HTML生成の段階ごとに途中状態を保持し、一定量でEventLoopへ戻す。HTTP-27〜32の表示/並び順/変換/8 MiB上限/失敗分類を適用する。2026-11-20にAutoindexTaskをコピー禁止とし、C++98のprivateなコピーコンストラクター/代入演算子を宣言して定義しない。ClientがAutoindexTaskを値メンバーとして所有し、開いているディレクトリ/項目一覧/ソート途中状態/生成位置/HTML本文/段階と後始末をまとめる。ソートはボトムアップのマージソートを採用し、区間幅/左右の位置/出力位置を保持して比較と移動を分割する。2026-11-20合意で名前一覧はstd::deque<std::string>に列挙順で保持し、名前を参照する0,1,2,...の番号をソートする。比較は対応する元の名前を使い、マージでは番号を移動し、HTML生成も整列済み番号から名前を参照する。番号列と作業領域の準備も分割設計へ含める。ソート段階は1回のstepで比較/番号書き込み合計256操作の採用案とし、比較1回/番号書き込み1回を各1操作として準備/初期化/残り番号の移動も数える。HTML生成は各stepの実追加最大8 KiBを採用案とし、固定HTML/表示名/リンクの変換後バイト数を数える。項目や変換の途中位置を保持して継続する。番号列とマージ用作業領域は2026-11-21合意でstd::vector<std::size_t>を1本ずつ使う。列挙とclosedirの成功後に空の2本へ項目数分をreserveする。マージ元の番号と最初のマージの出力はpush_backで段階的に追加し、1個につき1操作として1回のstepあたり合計256操作に含める。各パスの末尾で2本をswapし、2パス目以降は出力位置をsize()と別に管理して既存要素へ上書きする。上書きも1件につき1操作と数える。[番号列と作業領域の設計](../Requirements.md#autoindex-index-containers)を参照する。[段階分割の正本](../Requirements.md#autoindex-incremental-design)を参照する。

## C6 アップロード 【rysato, R:L, D:L, ★★★】

2026-11-16合意: 本文全体の検証後、RouterはFileWriteRequest(path/filename/body)とFILE_WRITE_REQUIREDを返す。Clientが値メンバーとして所有するFileWriteTask（2026-11-17合意でコピー禁止。C++98のprivateなコピーコンストラクター/代入演算子を宣言し定義しない）が新規作成・保存・後始末を担当し、各EventLoop周にwrite最大1回・最大8 KiBを進める。全量保存とclose成功後だけ200を生成する。保存本文は指示のbodyからタスクへswapで移し、移した指示側bodyは空とする。タスクは指示側本文への参照を残さない。開始APIはint start(FileWriteRequest&)とし、パス/保存名の保持とopen成功後だけ本文をswapする。0は開始準備完了、通常の開始失敗はHTTP-41のコードを返し、指示側本文を変更しない。進行APIはStepResult step()とし、IN_PROGRESS/COMPLETED/FAILEDを返す。各回write最大1回・最大8 KiB、全量保存とclose成功でCOMPLETED、writeの0/負値またはclose失敗でFAILEDとする。空本文はwriteせずclose成功で完了する。2026-11-17合意で、全量保存とclose成功をDONEとして確定し、本文/保存先パスを解放してからCOMPLETEDを返す。保存名だけをtakeFilenameまで保持する。結果取得はbool takeFilename(std::string&)とし、保存完了後の初回だけswapで保存名を渡してtrue、それ以外は出力不変でfalseとする。cancelはvoid cancel()を採用し、保存途中ならfdを閉じ今回作成した途中ファイルを削除、全量保存とclose成功後のファイルは残す。保持データを解放して未開始へ戻し、繰り返しても二重close/削除しない。デストラクターも同じ解放を行い例外を外へ出さない。内部状態はprivateなPhase(IDLE/WRITING/DONE/ERROR/TAKEN)で管理し、完了/失敗/取得済みのboolを重ねない。初期状態とcancel後はIDLE、start成功でWRITING、全量保存とclose成功でDONE、保存名取得でTAKEN、通常の開始/保存失敗でERRORとする。startはIDLEで開始し、それ以外は500でタスク/指示/ファイルを変更しない。再利用前にcancelでIDLEへ戻す。IDLEからの通常の開始失敗は、取得済みfdを一度だけ閉じ、タスクのパス/保存名/本文を解放し書き込み位置0・fd=-1でERRORへ進む。入力指示を維持し、後始末に必要なパスは削除試行/ログ記録後に解放する。stepはWRITINGで保存を進め、DONEはI/OせずCOMPLETED、ERROR/IDLE/TAKENはI/OせずFAILEDを返す。WRITING以外では状態/保持データを維持し、write/close/削除を繰り返さない。ClientのWRITING_FILEと基本遷移は2026-11-17に採用済み。開始成功後にClientがHttpRequest::releaseBody()で元本文を解放して保存中へ、未完了は維持、完了は保存名取得/makeUploadSuccessからprepareResponseへ、開始失敗は返却コード、保存失敗ではstep内でfdのclose、今回作成した途中ファイルの削除試行/失敗ログ、本文/パス/保存名の解放と位置0への復帰を済ませ、ERRORでFAILEDを返す。Clientは500をprepareResponseへ渡す。再step/cancel/破棄で同じ資源をclose/削除しない。期限と異常時の詳細遷移は未決定。[設計の正本](../Requirements.md#file-write-design)を参照する。

**合意済み(HTTP-33〜44)**: 正本は [Requirementsのアップロード](../Requirements.md#http-upload)。POST許可・upload_store有効・CGIとの分離を維持する。CGIへのPOSTには本節の受付制限を適用しない。

- 受付はmultipart/form-dataの1ファイル。ブラウザのフォーム/curl -Fに対応し、他形式・Content-Type欠落/空は415。ファイル0/2個以上は400。通常フォーム項目は検査して捨て、filename空は未選択、名前あり/中身0バイトは空ファイルとして保存する。
- boundaryは引用符付き/なし、RFCの1〜70文字、大小文字区別。区切り/終端を確認し、区切り行末のSP/HTABを受理、前後の付加部分を捨てる。欠落・重複・不正は400。全体を検査してから保存する。
- 各パートのContent-Disposition: form-data/nameを検査し、パートヘッダーはCRLF・折り返しなし・終端空行込み8 KiBで超過400。Content-Typeは保存に使わない。NUL/改行を含む本文を加工せず保存し、要求Content-Encodingが非空なら415、パートContent-Transfer-Encodingは省略/空/binary以外415。ネスト解析を追加しない。
- filenameの/・逆斜線の最後の名前だけ使う。空・.・..・制御文字は400。空白/非ASCIIを保持し、percent-encodingをデコードせず、upload_store＋名前へ保存する。POST先URLはlocation選択に使う。
- O_CREAT/O_EXCL/O_WRONLY、mode 0600で新規作成し、同名対象409。上書き/採番なし。openの権限/読み取り専用403、保存名のパス長400、保存先消失/その他500。短い正のwriteは位置を進めて残りを次の周へ回し、0以下/close失敗500。write後のerrno分岐なし。
- パートヘッダー/区切りも要求本文上限に含め、超過413。追加のファイルサイズ上限なし。本文/I/Oと全体メモリはHTTP-71・72の採用案に従う。
- 2026-11-16合意: FileWriteTaskは保存結果と保存名だけを渡し、HttpResponse/HTMLを生成しない。closeまで成功した後、rysato側のHttpResponse::makeUploadSuccess(const std::string& filename)が保存名をHTMLエスケープした200応答を作り、ClientがprepareResponseへ渡す。成功応答の生成APIは採用済み。タスクの保存名取得APIも採用済み。[分担の正本](../Requirements.md#file-write-success-response)を参照する。
- 成功は200と保存名の結果HTML(HTMLエスケープを適用)。201/LocationというRFCの推奨と区別し、公開URLを自動推定しない。GET公開はroot/aliasを保存先へ対応させ、upload_storeから自動設定しない。
- 保存途中の今回作成したファイルだけを、保存用fdの後始末後にstd::removeで削除する(HTTP-49・50)。削除失敗時は元の500を維持し、削除できなかったパスをログへ記録する。409等の既存対象は削除しない。

単一ファイル等の受付制限はチーム方針であり、課題から各形式を個別に免除されたという意味ではない。実装・受入試験は未完了。

## C7 DELETE 【rysato, R:S, D:S, ★★】

**std::removeによる削除手段を含め合意済み(HTTP-45〜50)。** 正本は [RequirementsのDELETE](../Requirements.md#http-delete)。

- 通常locationのroot/aliasでGETと同じ規則で対象を解決し、通常ファイルだけを削除する。ディレクトリ/その他の種別は403。再帰削除・indexへの置き換え・GET用の末尾スラッシュ補完は追加しない。
- 成功は204で本文/Content-Lengthなし。対象なし404、種別/権限/読み取り専用403、長すぎるパス414、その他500。allow_methodsによる405＋AllowとCGI用locationのDELETE禁止は維持する。
- 保存失敗時は今回作成したファイルだけを削除する。後始末にも失敗したら元の500を維持し、残ったパスをログへ記録する。無期限に再試行せず、409等の既存ファイルへ触れない。

2026-10-07のユーザー決定により、C++98標準ライブラリの`<cstdio>`にある`std::remove(const char*)`を使用可能とする。戻り値0で成功、非0で失敗。通常ファイルと確認してから呼び、空ディレクトリも削除しない。失敗直後のerrnoはHTTP-48に従って分類し、read/write後のerrnoによる分岐禁止とは区別する。対象ファイルの読み取り/書き込み権限を削除権限の代わりに検査しない。RFCのDELETEの意味と実ファイル削除案の違い、配信だけ隠す方式を無断で代替しないことは正本を参照。実装・受入試験は未完了。

## C8 CGIプロセス制御(システムコール面) 【tasugiya(主)/共, R:L, D:L, ★★★】

**理解すべきこと**

- pipe×2→fork→子のdup2・不要端close・chdir・execveで起動する。**_exitとclock_gettimeはユーザー確認により使用不可(CGI-119)**。子の失敗時の終了と計時は9/10の現時点の採用案で実装し、実際の動作確認で見直し得るものとして扱う。
- 親: 使わない端の close、**パイプfdをノンブロッキング化し poll に登録**(CGI出力の読み取り、ボディの書き込み。課題の「pipeは必ずpoll経由」要件)。
- **fork は CGI 実行にのみ使用可**。ここ以外で `fork` しない。
- `fork` 後に子で C++ オブジェクトが多数存在する状態 → 子は `execve` 直前までに**複雑な処理をしない**(メモリ確保など)、`envp` は fork 前に構築しておく(`char**` の組み立て・解放責任の明確化)。
- 子プロセス回収: `waitpid`(`WNOHANG`)、ゾンビ防止。タイムアウト時の `kill(pid, SIGKILL)` → `waitpid`。
- `execve`等の子側起動失敗は502。子の明示的解放とmainへのreturn、親のwaitpidによる検出は、C12の項目9/10の現時点の採用案に従う。
- インタプリタ/スクリプトの検証: インタプリタは存在・実行権限、Pythonへ引数として渡すスクリプトは通常ファイルであること・読み取り権限を確認する。スクリプト自身の実行権限は要求しない。
- fd継承はCGI-116で確定済み。listen/acceptソケットと全CGIパイプにF_SETFD/FD_CLOEXECを設定し、子の0/1/2だけを継承する。

## C9 CGI仕様(RFC 3875) 【rysato(主)/共, R:L, D:L, ★★★】

**理解すべきこと**

- **メタ変数**(RFC 3875 §4.1)。決定事項に挙がった `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR` + `HTTP_*`。それぞれの**値の作り方**:
  - `SCRIPT_NAME`/`PATH_INFO` の分離(`/cgi/test.py/extra/path` → script=`/cgi/test.py`、PATH_INFO=`/extra/path`)はCGI-25、各環境変数の規則はCGI-29・CGI-30で確定済み。
  - 必須部分はPythonのみとし、PHP固有の環境変数は追加しない(確定、C12)。基本的なPATH_INFO対応は確定済み。CGI-50の環境変数の限定に従い、`PATH_TRANSLATED`は追加しない。
  - `CONTENT_LENGTH` はCGI-31で確定済み。復号後の本文が1バイト以上ならそのバイト数、0バイトなら空文字列とする。`CONTENT_TYPE` は要求ヘッダーの値をパラメーターも含めて渡し、ヘッダーなしの場合は省略する(CGI-32)。`QUERY_STRING` は未デコードのまま(CGI-28)。
  - `HTTP_*` 変換はCGI-39で確定済み: ヘッダー名をASCII大文字化、`-`→`_`、接頭辞`HTTP_`。値は前後SP/HTAB除去後に大小文字を保持する。`Content-Type`/`Content-Length` は専用変数だけで渡し、HTTP_*から除外する(CGI-40)。`Proxy` ヘッダーはCGI-45で除外を確定済み。名前の文字範囲はCGI-47で確定済み。転送対象の同名ヘッダーの重複拒否はCGI-48、HTTP_*の空値省略はCGI-49で確定済み。入力ヘッダーの残る規則はCGI-75〜83で確定済み。受け渡しAPIはCGI-91〜98で合意済み(C12)。

- **リクエストボディの渡し方**: un-chunk 済みのボディを子のstdinへ全て書き、書き終えたらstdinを **close(EOF)**(課題要件)。ボディが大きいと pipe バッファ(64KB程度)を超える → **書き込みもノンブロッキング+POLLOUT**(死活: 子が出力を溜めて読まないとデッドロックし得る → 読み/書きを同時にpoll)。
- **CGI出力のパース**(§6): `CGI-response = document-response / local-redir-response / client-redir-response / client-redirdoc-response`。ヘッダー部(`Content-Type`, `Status`, `Location`, 任意ヘッダー)+空行+ボディ。
  - `Status: 404 Not Found` → HTTPのステータス行へ変換。通常の文書応答で省略された場合は200。リダイレクトは別の規則で扱う。
  - `Location:` のみ(ローカル絶対パス `/other`)はRFC上はサーバー内部リダイレクトだが、今回の実装対象外とする(確定、C12)。絶対URLのLocationのみは302として扱う(確定)。
  - 通常の文書応答で `Content-Type` がない場合は不正なCGI出力として502。Locationのみのリダイレクトとは区別する。
  - **ヘッダー区切りが LF/CRLF 両方**あり得る(Pythonの`print`はLF)。
  - **`Content-Length` が無ければ EOF を終端とする**(課題要件)。サーバー側は終端後に自分で `Content-Length` を付けて返す(接続は `Connection: close`)。

- 終了ステータス(非0)はCGI-19のエラー対応に従う。stderrはCGI-51に従いWebservのstderrを引き継ぎ、stdoutへ混ぜず専用の監視・収集処理も追加しない。

## C10 CGIとイベントループの統合 【共, R:M, D:M, ★★★】

- CGI状態を独立した `CgiProcess` に持たせ、EventLoopが **fd→オブジェクトの対応表**で引くことは決定済み。`Client::WAITING_CGI` は状態設計案。Clientとの関連付けやポインターの寿命管理はCGI-99〜106で合意済み(C12)。
- 複数のfd(CGIのstdin書込/stdout読取)を毎周のpollfd再構築で扱う方法。
- **失敗パターンの網羅**: execve失敗、CGIがすぐ終了、出力なし、出力途中でクライアント切断、タイムアウト(10秒→kill→504)、巨大出力(上限)、子が書込を詰まらせる、複数同時実行。アプリケーション独自のCGI同時実行数上限は設けず、pipe/fork失敗は500として後始末する(C12)。
- CGI結果のHttpResponseへの変換はrysatoのRouter::parseCgiOutput、即時応答/CGI要求の区別はRouter::routeが担当する(CGI-91〜98確定)。所有権・寿命はCGI-99〜106で確定済み。後始末はCGI-107〜118で確定済み。起動失敗検出・計時は項目9/10の現時点の採用案、エラー適用境界はCGI-128〜135の確定規則に従う。

## C11 Cookie/セッション(ボーナス) 【rysato, R:M, D:M, ★】

- `Set-Cookie`/`Cookie` の形式(RFC 6265)、属性(`Path`, `Expires`/`Max-Age`, `HttpOnly`, `SameSite`)。セッションIDの生成(乱数源の選定: C++98標準ライブラリの `std::rand` 等を含め検討する。Cookie/セッションは必須範囲外)、サーバー側セッション保管(メモリ上のmap)。
- ボーナスは**必須部分が完璧な場合のみ評価**されるため、優先度は最低。複数CGIタイプ対応(PHP等)も同様。

<a id="cgi-details"></a>

## C12 CGIの最小構成・合意済み規則の説明 【共】

記録日: 2026-10-04。CGI-25〜43追記: 2026-10-05。CGI-44〜83追記: 2026-10-06。最新の合意反映・資料整理: 2026-10-07。課題PDF Version 24.1と参考資料を踏まえ、必須部分の実装量を抑えるための合意状況を記録する。

**CGIの10項目は合意完了(CGI-01〜135)。** 起動・計時のCGI-120〜127は現時点の代替案で、実装・実際の動作で見直し得る。合意完了は実装・結合試験合格を意味しない。決定表の正本は [RequirementsのCGI節](../Requirements.md#cgi-decisions) とし、本書には規則の説明・具体例・共有APIを記録する。変更時は関連するHTTP・I/O・Config・テスト計画も合わせて更新する。

### 決定表と採用案の位置付け

[CGI-01〜135の決定表](../Requirements.md#cgi-decisions) を参照する。CGI-120〜127は現時点で採用する代替案であり、実装・実行環境・動作確認の結果で見直し得る。子の起動失敗時に所有メモリを明示的に解放し、親用の停止・回収処理を実行しない方針は合意済み。具体的な経路は項目9/10で説明する。

### 環境変数の合意済みの規則

下記11変数の値と設定・省略の規則は合意済み。CGI用環境はCGI-50に従い、これらと転送対象のHTTP_*だけから構築し、親の環境変数はコピーしない。SERVER_SOFTWAREはCGI-38で対象外とし、CGIへ渡す環境変数に含めない。HTTP_*の名前・値の変換はCGI-39、Content-Type・Content-LengthのHTTP_*からの除外はCGI-40、Transfer-Encodingの除外はCGI-41、Connection自体の除外はCGI-42、その他4つの通信制御用ヘッダーの除外はCGI-43、Connectionに列挙されたヘッダーの除外はCGI-44、Proxyの除外はCGI-45、認証用ヘッダーの除外はCGI-46、転送する名前の文字範囲はCGI-47、転送対象の同名ヘッダーの重複拒否はCGI-48、HTTP_*の空値省略はCGI-49で確定済み。入力ヘッダーの残る規則はCGI-75〜83で確定済み。環境変数の受け渡しAPIはCGI-91〜98、所有権・寿命はCGI-99〜106で合意済み。

| 変数 | 値の出所 | 空値/省略 | 合意ID |
|---|---|---|---|
| REQUEST_METHOD | 解析済みの要求メソッド。GET要求なら`GET`、POST要求なら`POST` | 必ず設定する。空文字列・省略にはしない | CGI-27 |
| QUERY_STRING | 最初の`?`より後ろの未デコードのクエリ。`?`自体は含めず、`+`や`%20`をそのまま保持する | 必ず設定する。クエリなし・末尾が`?`だけの場合は空文字列とし、省略しない | CGI-28 |
| PATH_INFO | C-24・C-25のデコード・正規化後のURLパスからCGI-25で切り分けた後続部分。スクリプト直後の`/`だけが残る場合は`/` | 必ず設定する。追加パスがなければ空文字列とし、省略しない | CGI-29 |
| SCRIPT_NAME | デコード・正規化後のスクリプトを示すURLパス。PATH_INFO・クエリ・ディスク上のパスは含めない。index経由ではURL上のディレクトリパスに選ばれたindexのファイル名を付ける | 必ず設定する。この対応範囲ではスクリプトを示す先頭`/`のパスとなり、空文字列・省略にはしない | CGI-30 |
| CONTENT_LENGTH | CGIへ渡す復号後の本文の実際のバイト数。1バイト以上なら10進数の文字列 | 必ず設定する。0バイトなら空文字列とし、省略しない。長さ指定なし・Content-Length: 0・空のchunked本文も同じ扱い | CGI-31 |
| CONTENT_TYPE | 要求のContent-Typeヘッダーの値。C-33による前後SP/HTAB除去後の値を、boundary等のパラメーターも含めて渡す | ヘッダーがあれば本文0バイトでも設定する。ヘッダーがなければ省略し、推測・既定値の補完はしない | CGI-32 |
| SERVER_PROTOCOL | 解析済みの要求バージョン。HTTP/1.0要求なら`HTTP/1.0`、HTTP/1.1要求なら`HTTP/1.1`。HTTP/1.2〜1.9も受信表記を保持する(HTTP-04)。応答側の固定値で上書きしない | 必ず設定する。空文字列・省略にはしない | CGI-33 |
| GATEWAY_INTERFACE | 固定文字列`CGI/1.1`。要求のHTTPバージョンによって変えない | 必ず設定する。空文字列・省略にはしない | CGI-34 |
| SERVER_PORT | その接続を受け付けたlistenのポート番号を10進数の文字列で渡す | 必ず設定する。80等の標準ポートでも省略せず、空文字列にはしない | CGI-35 |
| REMOTE_ADDR | acceptで取得した接続元のIPv4アドレスをドット区切りの10進数で渡す。ポート番号は含めず、逆引きDNSや転送ヘッダーによる上書きを行わない | 必ず設定する。空文字列・省略にはしない | CGI-36 |
| SERVER_NAME | 有効なHostのポート部分を除いたホスト名またはIPアドレス。名前の大小文字は保持する。HTTP/1.0でHostがなければ接続のサーバー側IPをgetsocknameで取得して使う | 必ず設定する。空文字列・省略にはしない | CGI-37 |

[RFC 3875 §4.1.12](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.12)はREQUEST_METHODに要求メソッドを設定することを必須としている。GET/POSTへの限定はCGI-09で定めたチームの対応範囲による。

[RFC 3875 §4.1.7](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.7)はQUERY_STRINGを必ず設定し、クエリがない場合は空文字列にすることを要求している。`/cgi/test.py?name=taro&x=a+b`では値を`name=taro&x=a+b`とし、`/cgi/test.py`と`/cgi/test.py?`では空文字列とする。クエリの解析はPython側で行う。

[RFC 3875 §4.1.5](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.5)はPATH_INFOにスクリプトより後ろのデコード済みのパスを渡すことを定め、空文字列と`/`を区別する。空の場合に省略せず設定するのはチームの採用仕様であり、[§4.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1)の空値に関する一般規則に沿う。`/cgi/test.py`と`/cgi/test.py?name=taro`では空文字列、`/cgi/test.py/`では`/`を渡す。

[RFC 3875 §4.1.13](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.13)はSCRIPT_NAMEにスクリプトを識別する未エンコードのURIパスを必ず設定し、PATH_INFOを含めないことを要求している。`/cgi/test.py/extra?name=taro`では`/cgi/test.py`を渡す。index経由の具体的な対応付けはチームの採用仕様とし、`/cgi/`で設定された`index.py`を実行する場合は`/cgi/index.py`を渡す。この値を設定するためのクライアント向けリダイレクトは行わない。

[RFC 3875 §4.1.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.2)はCONTENT_LENGTHを転送符号化等を除去した本文のバイト数とし、データがなければ空値または省略としている。チームでは0バイトの場合も変数を設定し、値を空文字列に揃える。`name=taro`なら`9`、同じ内容のchunked本文を復号した場合も`9`とする。要求ヘッダーの数字をそのままコピーせず、CGIへ渡す本文の実長から生成する。この規則はCGIの入力環境変数についてであり、CGI出力のContent-Length(CGI-17)とは区別する。

[RFC 3875 §4.1.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.3)は要求にContent-TypeヘッダーがあればCONTENT_TYPEを必ず設定することを要求し、この変数に既定値を設けていない。チームではヘッダーがなければ形式を推測せず、変数を省略する。`multipart/form-data; boundary=abc`ならboundaryを含む値を渡し、CGIへの受け渡しのためのmultipart分解は行わない(CGI-03)。要求Content-Typeの重複・空値・値の扱いはCGI-79・80で確定済み。入力の形式の解釈はCGIへ任せ、CGI出力のContent-Typeの構文検査とは区別する。

[RFC 3875 §4.1.16](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.16)はCGI要求に使うプロトコル名とバージョンの設定を必須としている。チームでは既存方針に従い受信した要求の版を使い、HTTP/1.1要求への応答をHTTP/1.0で生成する場合もSERVER_PROTOCOLは`HTTP/1.1`とする。

[RFC 3875 §4.1.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.4)は使用するCGIの仕様バージョンをGATEWAY_INTERFACEに設定することを必須としており、同RFCが定義するバージョンは1.1である。チームではHTTP/1.0・HTTP/1.1のどちらの要求でも固定値`CGI/1.1`を設定し、Config項目は追加しない。CGIのバージョンを示す値であり、RFC 3875の全機能への対応を新たに合意するものではない。

[RFC 3875 §4.1.15](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.15)は要求を受信したTCPポート番号の設定を必須とし、標準ポートでも省略しないことを要求している。チームではその接続を受け付けたlistenの既存ポート番号を使い、8080で受け付けた場合は`8080`、80で受け付けた場合は`80`を設定する。追加のConfig項目は設けない。

[RFC 3875 §4.1.8](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.8)は要求を送ったクライアントのネットワークアドレスの設定を必須としている。チームではIPv4の接続元を使い、ローカル接続なら`127.0.0.1`等を渡す。値は接続情報から取得し、X-Forwarded-For等を参照しない。この決定はREMOTE_ADDRの値の出所と形式についてであり、HTTP_*として渡すヘッダーの範囲を新たに確定するものではない。4オクテットの十進表記とConnectionInfoによる受け渡しはCGI-93で合意済み。

[RFC 3875 §4.1.14](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.14)は要求先のホスト名またはネットワークアドレスを必ず設定することを要求している。チームでは`Host: localhost:8080`なら`localhost`、`Host: example.com`なら`example.com`を渡す。HTTP/1.0でHostがない場合の接続情報は、末尾/補完のリダイレクトで確定したC-28と共通にできる。listenのワイルドカード値`0.0.0.0`ではなく、接続先の実際のサーバー側IPを使う。Hostの受付検証はC-28・C-30に従い、HTTP/1.1のHost欠落は従来どおり400とする。DNS照会・server_name設定・Hostによる仮想ホスト選択は追加しない。受け渡しAPIはCGI-93のConnectionInfoで合意済み。接続情報取得失敗時の扱いは、残るHTTP/ネットワーク境界の設計で明確にする。

**確定(CGI-38)**: 最小構成ではSERVER_SOFTWAREを省略し、固定のサーバー名・バージョンをCGIへ渡す処理は実装しない。課題PDFはこの変数を個別に列挙していない。[RFC 3875 §4.1.17](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.17)では必須だが、チームではRFC全体への準拠を目標とせず、この項目を対応範囲から外す。

**確定(CGI-39)**: 転送対象の要求ヘッダー名をASCII大文字に変換し、`-`を`_`に置き換えて先頭に`HTTP_`を付ける。`User-Agent: curl/8.0`なら`HTTP_USER_AGENT=curl/8.0`、`X-Test: AbC`なら`HTTP_X_TEST=AbC`となる。値はC-33の前後SP/HTAB除去後のものを使い、大小文字を保持する。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)がこの名前の変換規則を定めている。この合意だけで全要求ヘッダーの転送を決めるものではなく、転送対象・除外・重複はCGI-40〜49・75〜83に従う。名前をASCII英数字と-に限定するCGI-47により、変換後の名前の衝突を避ける。

**確定(CGI-40)**: Content-TypeとContent-Lengthは専用変数のCONTENT_TYPE・CONTENT_LENGTHで渡し、HTTP_CONTENT_TYPE・HTTP_CONTENT_LENGTHは設定しない。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)は、専用変数で利用できるヘッダーをHTTP_*から除外することを推奨している。本文長は要求ヘッダーの値ではなく、CGI-31に従って復号後の本文から生成する。CONTENT_TYPEのヘッダーなし時の省略はCGI-32を維持する。

**確定(CGI-41)**: 要求のTransfer-EncodingをHTTP_*へ転送せず、HTTP_TRANSFER_ENCODINGは設定しない。CGIがstdinから読むのはchunked復号後の本文であり、チャンクのサイズ行や区切りは含まない。[RFC 3875 §4.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.2)は転送符号化を除去しCONTENT_LENGTHを計算し直すことを要求している。チームでは復号前の転送形式をCGIへ伝える処理を省く。本文の受け渡しはCGI-03、本文長はCGI-31、stdinのEOF通知は既存方針を維持する。

**確定(CGI-42)**: 要求のConnectionヘッダー自体はHTTP_*へ転送せず、HTTP_CONNECTIONを設定しない。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)は、Connectionのようなクライアントとの通信に関するヘッダーを除外してよいとしている。`Connection: close`でも`Connection: keep-alive`でも、接続管理はWebservが担当し、既存の1接続1要求・応答後切断の方針を維持する。Keep-Alive・TE・Upgrade・Proxy-Connectionの除外はCGI-43で確定済み。Connectionに列挙された他のヘッダー名のHTTP_*からの除外はCGI-44で確定済み。

**確定(CGI-43)**: Keep-Alive・TE・Upgrade・Proxy-ConnectionをHTTP_*から除外する。[RFC 9110 §7.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-7.6.1)はHTTPメッセージを転送する際にこれらを除去・置換すべきヘッダーとして挙げており、チームではこの区別をCGIへの受け渡しにも用いる。ヘッダー名の大小文字を区別せずに除外し、HTTP_KEEP_ALIVE・HTTP_TE・HTTP_UPGRADE・HTTP_PROXY_CONNECTIONを設定しない。各ヘッダーの機能や、要求受付時の検証・応答規則を新たに決める合意ではない。

**確定(CGI-44)**: Connectionの値に列挙されたヘッダー名もHTTP_*から除外する。例えば`Connection: X-Test`と`X-Test: AbC`があればHTTP_X_TESTを設定しない。Connectionの値をカンマで区切り、各名前の前後SP/HTABを除去して大小文字を区別せず照合する。[RFC 9110 §7.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-7.6.1)はHTTPでメッセージを転送する際、Connectionに列挙されたヘッダーも除外することを要求しており、チームでは同じ区別をHTTP_*の転送規則として採用する。本文の受信処理や専用環境変数の規則は維持する。要求Connectionの値の検証・重複はCGI-78で確定済み。

**確定(CGI-45)**: ProxyヘッダーはCGIへ転送せず、HTTP_PROXYを設定しない。HTTP_PROXYは外向き通信のプロキシ設定に使われる環境変数名でもあり、この名前の衝突による問題が過去に[httpoxy](https://httpoxy.org/)として報告されている。課題PDFの個別指定ではなく、チームで衝突を避けるために採用した除外規則。Proxyの存在だけを理由に要求を拒否する分岐は追加せず、HTTP要求の一般的な検証やその他の既存エラー規則は維持する。

**確定(CGI-46)**: Authorization・Proxy-AuthorizationをHTTP_*から除外し、HTTP_AUTHORIZATION・HTTP_PROXY_AUTHORIZATIONを設定しない。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)はAuthorizationのような認証情報を含むヘッダーの除外を推奨している。チームではCGI-06の認証機能を追加しない方針に従い、これらの存在だけを理由とする拒否や認証情報の解釈も追加しない。HTTP要求の一般的な検証やその他の既存エラー規則は維持する。

**確定(CGI-47)**: HTTP_*へ転送するヘッダー名はASCII英数字と`-`だけで構成されるものに限定し、CGI-40〜46の除外規則を適用する。`X-Test`は文字範囲を満たし、`X_Test`やその他の記号を含む名前はCGIへの転送だけを省略する。HTTP要求としてはC-34のtoken検証を維持し、この文字範囲の違いだけで要求を拒否しない。変換前の`_`を転送対象から外すことで、`X-Test`と`X_Test`が同じHTTP_X_TESTになる衝突を避ける。転送対象の同名ヘッダーの重複拒否はCGI-48、HTTP_*の空値省略はCGI-49で確定済み。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)は全要求ヘッダーの環境変数化を必須としておらず、この文字範囲はチームが最小構成のために採用する制限である。

**確定(CGI-48)**: CGIへの転送対象となる同名ヘッダーが複数行あれば、起動前に400を返す。例えば`X-Test: one`と`X-Test: two`、名前の大小文字だけが異なる行、同じ値を持つ2行も拒否する。名前の比較はC-34に従って大小文字を区別しない。合意済みの除外規則とCGI-47の文字範囲を適用した転送対象についての規則であり、Host・Content-Length等の個別検証は既存方針を維持する。除外対象や通常配信の要求にこの重複拒否を一律に適用する合意ではない。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)は同名ヘッダーを意味が同じ単一値へ結合することを要求するが、チームでは最小構成のため重複を受理しない制限を採用する。課題PDFの明示要件ではなく、結合・先勝ち・後勝ちを実装しない判断である。

**確定(CGI-49)**: 転送対象ヘッダーの前後SP/HTABを除去した値が空なら、対応するHTTP_*は省略する。`X-Test:`、空白だけの`X-Test:   `、X-Testがない場合はいずれもHTTP_X_TESTを設定しない。空文字列の変数を生成する処理は設けない。[RFC 3875 §4.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1)は空値と省略を区別せず、任意のメタ変数が空なら省略してよいとしている。対象はHTTP層の検証を通ったヘッダーであり、Host等の個別検証は維持する。空値もヘッダーの行としては存在するため、転送対象の同名ヘッダーが複数行あればCGI-48の重複拒否を適用してから空値の省略を判断する。

**確定(CGI-50)**: `execve`へ渡す`envp`は、合意済みの11種類の環境変数と転送対象のHTTP_*だけから構築する。CONTENT_TYPE等の省略規則も維持し、常に11個すべてを設定する意味ではない。親プロセスの環境をコピーしないため、起動元のPATH・HOME・PYTHONPATH・HTTP_PROXY等は渡さず、親にREQUEST_METHOD等があっても要求から生成した値を使う。PythonとスクリプトはCGI-26の絶対パスで指定するため、起動対象の検索用PATHは追加しない。環境変数を追加・上書きするConfig項目も設けない。[Linux execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)では、envpで新しいプログラムに渡す環境を指定できる。課題PDFは要求情報をCGIから利用できることを求めるが、親の環境の継承方法は指定しておらず、本決定は最小構成のためのチーム方針である。envpの組み立て・寿命・所有権はCGI-99〜106、担当境界はCGI-91〜98で合意済み。

**確定(CGI-75〜83、項目4/10)**: 要求ヘッダーのCGIへの受け渡しを次の規則で確定する。

- CGI-39〜49の名前・除外・重複・空値の規則を満たす要求ヘッダーを転送し、Accept等の個別の許可一覧は作らない。例えばHostはHTTP_HOSTへポート付きの値を渡し、SERVER_NAMEのホスト部分のみの規則とは区別する。
- Trailer・ExpectもHTTP_*から除外する。本文後のtrailerはHTTP層で読み取り・構文検査し、環境変数に追加しない。初期ヘッダーを上書きしない。HTTP/1.1のExpectは既存の417でCGIを起動せず、HTTP/1.0では値によらず無視する。
- 要求Connectionの値はHTTP-10により通常要求でも同じ検証を行う。各行をカンマで区切り、要素の前後SP/HTABを除去してヘッダー名のtokenとして検証する。不正な非空要素は400、空要素は無視する。複数行の場合は全行の名前を除外対象に加える。CGI-44の専用変数・本文受信を変更しない方針を維持する。
- 要求ヘッダーの共通の値の文字検査として、HTAB以外の制御文字(0x00〜0x1F)とDEL(0x7F)を400にする。0x80〜0xFFは変換せず保持する。転送から除外するヘッダーもこの共通検査を受ける。ボディには適用しない。
- HTTP-07によりContent-Typeの最大1行を全要求へ適用する。CGIへの要求でも同値・空値を含む重複は400とする。単一の値は前後SP/HTAB除去後にCONTENT_TYPEへ渡す。空ならCONTENT_TYPEを空文字列で設定し、ヘッダーなしなら省略する。本文の形式やパラメーターの解釈はCGIへ任せ、入力のメディアタイプの構文・登録一覧の照合をCGIへの受け渡しのために追加しない。通常のアップロード側の検証は別の既存仕様に従う。
- 転送除外対象の重複を一律に拒否する分岐は追加しない。Host・Content-Length等の個別検証、Connectionの全行処理とCGI要求のContent-Typeの重複拒否は維持する。
- 要求のContent-Encodingは転送規則を満たせばHTTP_CONTENT_ENCODINGへ渡す。Webservがgzip等を展開する機能は追加せず、chunked等の転送符号化の復号だけを済ませた本文を渡す。
- CGI環境変数に渡す要求情報に実NULバイトが残れば起動前に400とし、C文字列として値が途中で切れることを防ぐ。クエリの文字列%00は未デコードのまま渡せる。URLパスの%00を400とするC-24と、本文を加工しないCGI-03は維持する。

転送範囲は[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)、内容符号化を展開しない方針は[§4.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.2)、値の文字検査は[RFC 9110 §5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.5)、Connectionの空要素は[§5.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.1)を参考にする。課題PDFの個別指定ではなく、Trailer・Expectの除外、trailerを環境変数に追加しないこと、CGI要求のContent-Typeの重複拒否等は最小構成のためのチーム方針である。

### CGIのファイル・パス検証の合意済みの動作

**確定(CGI-84〜90、項目5/10)**: 実行するスクリプトまでを検証し、PATH_INFOに対応するファイルを探す処理は追加しない。

- rootは正規化済みURI全体を追加し、root末尾とURI先頭で重なるスラッシュを1個にする。aliasはlocation prefixを置換する。aliasのprefixと値の末尾スラッシュ必須、Config相対パスの設定ファイル基準、URLの1回だけのデコード・正規化は維持する。追加のファイル名文字制限やsymlink検出は設けず、C-26の運用前提を維持する。
- 例えばrootが`/srv/www`の`/cgi/test.py/extra`では、`/srv/www/cgi/test.py`が設定拡張子の最初の通常ファイルならこれを実行し、`/extra`はPATH_INFOとして渡す。`/srv/www/cgi/test.py/extra`の存在・権限は検査しない。スクリプトのディレクトリをcwdとするCGI-26も維持する。
- statで種別を検査し、ディレクトリは実行せず既存のディレクトリ・index規則を適用する。通常ファイルでもディレクトリでもないFIFO等は403。スクリプトはaccessのR_OKで読み取りを確認し、X_OKは要求しない。内容の構文検査・試し実行は省く。
- PythonはC-40の起動時検証を維持し、要求ごとの事前検査を追加しない。実際のexecve失敗は502。検査後にファイル・権限が変化してもリトライせず、既存の起動・異常終了エラーに従う。子のchdir・dup2等の失敗検出・終了方法は9/10で決める。
- stat/accessが失敗した直後にerrnoを読み、ENOENT・ENOTDIRは404、EACCESは403、ENAMETOOLONGは414、その他は500にする。read/write(recv/send)後にerrnoで分岐する処理には用いない。

種別と失敗理由は[Linux stat(2)](https://man7.org/linux/man-pages/man2/stat.2.html)、R_OKとX_OKの区別は[Linux access(2)](https://man7.org/linux/man-pages/man2/access.2.html)を参照する。課題PDFの許可関数にはstat・accessが含まれ、errnoの禁止はread/write後の挙動変更について記されている。HTTPコードへの分類、要求時のPython事前検査とリトライの省略はチーム方針であり、課題PDFの個別指定ではない。

### CGI出力ヘッダー検証の合意済みの動作

**確定(CGI-52)**: CGI出力のContent-Type・Status・Locationは、それぞれ最大1行とする。名前はASCIIの大小文字を区別せずに比較し、同じ名前が複数行あれば値が同じでも502とする。例えば`Content-Type: text/plain`と`content-type: text/html`、同値のStatusを2行、同値のLocationを2行も拒否する。結合・先勝ち・後勝ちの処理は追加しない。[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)は、この3種類をCGI専用の応答ヘッダーと定義し、それぞれの重複を禁止している。課題PDFの個別指定ではなく、公式リファレンスとCGI-24の不正出力を補正しない方針に従って採用する検証規則である。応答の種類ごとの必須・省略規則は既存の合意を維持し、他のヘッダーの重複はCGI-69、Content-Lengthの数値検証はCGI-67で確定済み。

**確定(CGI-53)**: CGI出力のヘッダー部分では、SP/HTABで始まる空でない行を不正出力として502にする。前のヘッダー値への結合や字下げの除去は行わず、最初のヘッダー行の字下げも拒否する。SP/HTABだけの行も502とし、改行を除いて完全に空の行だけでヘッダーを終了する。例えば`Content-Type: text/plain;`の次行に` charset=UTF-8`が続く出力は受理しない。[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)は各ヘッダーを1行で記述し、継続行を使用できないとしている。課題PDFの個別指定ではなく、公式リファレンスとCGI-24の不正出力を補正しない方針に沿う検証規則である。改行はCGI-15に従ってLF/CRLFを受理し、ヘッダー終端後の本文にはこの判定を適用せず空白や改行を保持する。HTTP要求に対するC-33の400とは別に、CGI出力のエラーは502とする。

**確定(CGI-54)**: CGI出力のヘッダー名とコロンの間にSP/HTABがあれば502にし、空白を除去して受理する処理は作らない。`Content-Type: text/plain`はこの検査を通るが、`Content-Type : text/plain`やコロン直前にタブがある行は拒否する。[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)はコロンと値の間の空白を認める一方、ヘッダー名とコロンの間の空白を認めていない。課題PDFの個別指定ではなく、公式リファレンスとCGI-24の不正出力を補正しない方針に沿う検証規則である。対象はヘッダー部分のみとし、本文は変更しない。ヘッダー名の文字範囲はCGI-55で確定済み。値の前後空白の処理はCGI-58で確定済み。

**確定(CGI-55)**: CGI出力のヘッダー名は空でないASCIIのtokenとして検証し、不正なら502にする。英数字とtokenで認められた記号を受理するため、`X-Test`と`X_Test`は名前の検査を通るが、`X Test`、空の名前、制御文字・非ASCII文字等を含む名前は拒否する。名前を補正する処理は追加しない。[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)はfield-nameをtokenと定義し、[§2.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-2.2)が文字範囲を規定している。課題PDFの個別指定ではなく、公式リファレンスに沿う検証規則である。C-34のHTTP要求ヘッダー名と同じ文字判定を利用できる。CGI-47の要求ヘッダーをHTTP_*へ転送する文字範囲とは適用箇所が異なり、転送規則やその他の出力ヘッダーの採否は変更しない。

**確定(CGI-56〜61、項目1/10)**: 出力の基本構文・値・Content-Type・Statusを次の規則で検証する。

- 各ヘッダー行を最初のコロンで分ける。コロンがない行や、ヘッダー終端の空行がないままEOFになった出力は502とする。行ごとにLF/CRLFを受理し、混在も許容する。単独CRは拒否する。ヘッダー終端後の本文にはこれらの判定を適用しない。
- 値の前後SP/HTABだけを除去し、内部の空白・タブと大小文字を保持する。HTAB以外の制御文字(0x00〜0x1F)とDEL(0x7F)は502とする。0x80〜0xFFは変換せず保持し、ヘッダーごとの追加の構文検証を行う。空値は省略と同じ扱いとし、重複検査は空値の省略より先に行う。
- Content-Typeのtype/subtypeは双方が空でないtokenとし、付いているパラメーターは名前と値の構文を検証する。値はtokenまたはquoted-stringとし、構文上許されるSP/HTAB、引用・エスケープも扱う。未知の種類やパラメーターも構文が正しければ受理し、登録一覧の照合や本文からの形式推測・文字コード変換は行わない。通常の文書応答で空のContent-Typeは必須ヘッダー欠落として502とする。
- StatusはASCII数字3桁と直後の区切りSP、説明文の形式を検証する。区切りをHTABで代用しない。説明文は空でもよいため、末尾空白除去前の値で区切りSPを確認し、その後に説明文の前後空白を除去する。`Status: 200 `は構文を満たし、`Status: 200`は区切りSP欠落として502となる。コードはユーザーの指定に従い100〜599を受理し、099・600・4桁等は502とする。説明文は検証後にHTTPステータス行へ使う。通常の文書応答でStatus省略・空値ならCGI-14の200を維持する。

構文・空値・Content-Type・Statusは[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)、値の文字の扱いは[RFC 9110 §5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.5)、コード範囲は[§15](https://www.rfc-editor.org/rfc/rfc9110.html#section-15)を参考にする。課題PDFの個別指定ではなく、公式リファレンスとCGI-24に沿って採用する検証規則である。混在改行の許容と不正出力への502はチーム方針。**1xxはコードの範囲検査を通すが、1xxだけで終了するCGIはCGI-74に従い502とする。** 1xxは中間応答であり、HTTP/1.0要求への送出制限と最終応答が必要なHTTPの規則に従って決定した([RFC 9110 §15.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.2))。本決定は、CGIの1xxを最終応答としてそのまま送出する合意ではない。

### Location・リダイレクトの合意済みの動作

**確定(CGI-62〜66、項目2/10)**: Locationがある応答は、本構成では次のリダイレクトの形式として扱う。

- 前後SP/HTAB除去後の非空Locationは、スキーム付きの絶対URIと任意のフラグメントとして一般構文を検証する。スキームはHTTP/HTTPSに限定せず、[RFC 3986](https://www.rfc-editor.org/rfc/rfc3986.html)の一般URI構文を基準とする。空白・不正な%表記等は502とする。値をデコード・パス正規化せず、DNS照会や宛先への接続確認も行わない。重複・空値はCGI-52・59の規則を維持する。
- 有効なLocationだけがあり、本文が空なら302へ変換する。例えば`Location: https://example.com/next`と終端空行だけの出力は、この形式となる。サーバー独自のCGI拡張ヘッダーは定義しないため、Location単独の形式に追加の有効なCGI出力ヘッダーを認める合意ではない。
- Location・Status・Content-Typeがある形式では、Statusを301・302・303・307・308に限定し、指定されたコードと本文を返す。本文は空でもよい。その他の応答ヘッダーの転送はCGI-69〜72に従う。
- Locationに本文だけ・Statusだけ・Content-Typeだけを伴う等、上の形式に合わなければ502とする。StatusやContent-Typeを補完しない。Locationのない通常の文書応答で受理するStatusの100〜599は維持する。
- スキームのない`/other`・`other`・`//example.com/`等の非空Locationは502とする。内部再処理や絶対URIへの補完、外部向け302への変換は行わない。

応答形式は[RFC 3875 §6.2.2〜6.2.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.2.2)、Locationの定義は[§6.3.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3.2)を参考にする。課題PDFの個別指定ではなく、対応する本文付きリダイレクトのコードを5種類に限定し、対象外・不正な形式は502とするチーム方針である。

### HTTP応答への変換の合意済みの動作

**確定(CGI-67〜74、項目3/10)**: CGIの出力を検証してから、WebservがHTTP応答を組み立てる。

- 非空のContent-LengthはASCII数字だけの非負整数として検証し、0・先頭ゼロを受理する。符号・カンマ・内部空白・桁あふれは502。CGI本文の実長と比較し、不一致も502とする。CGIからの宣言値はHTTPへ直接コピーせず、実際に送るHTTP本文の長さをWebservが生成する。
- すべての出力ヘッダーは、名前のASCII大小文字を区別せず最大1行とする。同値や空値の重複も、空値省略・転送除外より先に拒否する。複数のSet-Cookieを受理・結合する特例も設けない。これでCGI-52の3種類以外の重複規則も確定した。
- Connection・Keep-Alive・TE・Upgrade・Proxy-ConnectionをHTTPへ転送せず、Connectionに列挙された名前も除外する。列挙された名前をカンマで区切り前後SP/HTABを除去して大小文字を区別せず照合する。名前として不正な非空要素は不正出力として扱う。Status・Content-Type・Location・Content-Lengthが列挙されていたら、専用の処理に必要な項目との矛盾として502。WebservはConnection: closeを付ける。
- 非空Transfer-EncodingをCGIが返したら502とし、CGI出力をchunkedとして復号する処理は追加しない。空値はCGI-59の省略規則に従う。入力要求のchunked復号とは別の規則である。
- 専用処理・除外対象以外のヘッダーは、構文と値の検証を通れば値を保持して転送する。Webserv側で同名ヘッダーを追加生成しない。Status自体は転送せずHTTPステータス行へ反映し、Content-TypeとLocationは合意済みの応答形式に従う。Location単独かどうかはCGI-63の有効なヘッダーの条件で判断し、転送除外によって不完全な応答形式を受理する補正は行わない。
- 204・205・304でCGI本文があれば502。本文が空なら204・304のHTTP応答にはContent-Lengthを付けず、205にはContent-Length: 0を付ける。CGIの長さ指定があれば先に通常の実長照合を行い、不一致を無視しない。304で実際には送らない表現の長さをContent-Lengthに指定する特例は追加せず、CGI-17の実長照合を維持する。
- Statusコードの数値検査は100〜599を維持するが、1xxだけで終了するCGIは502とする。1xxを最終応答として返さず、中間応答と最終応答を複数回扱う仕組みは追加しない。

長さと本文の制約は[RFC 9110 §8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6)・[§15](https://www.rfc-editor.org/rfc/rfc9110.html#section-15)、接続用ヘッダーは[RFC 3875 §6.3.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3.4)を参考にする。全ヘッダーの重複拒否、CGI出力の非空Transfer-Encodingの拒否、304の長さ特例を設けないこと、1xxだけで終了するCGIの502は最小構成のためのチーム方針である。Configのerror_pageの規則は維持し、CGIの結果とサーバー側のエラーの受け渡し・適用境界は10/10のCGI-128〜135に従う。

### stderrの合意済みの動作

**確定(CGI-51)**: CGIの標準エラー出力(fd 2)はWebservのstderrをそのまま引き継ぎ、CGIのstdoutへ付け替えない。通常の端末起動なら同じ端末へ出力され、Webservのstderrをリダイレクトして起動した場合はその出力先を引き継ぐ。stderr用のパイプ・poll登録・収集バッファ・ログファイル管理は追加しない。stderrに出力があることだけでは502にせず、終了状態・stdoutのCGI応答・上限・タイムアウトは既存の規則で検証する。[RFC 3875 §6.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.1)は通常の応答経路をstdoutとしている。課題PDFにstderr専用の収集機能の指定はなく、本決定は最小構成のためのチーム方針である。CGI-50の環境変数の非継承とは別の、ファイルディスクリプターの扱いである。

### 実行パス・cwdの合意済みの動作

**確定(CGI-26)**: CGIの子プロセスで、実行するスクリプトが置かれたディレクトリへchdirしてからPythonを起動する。Pythonとスクリプトはともに絶対パスで指定する。例えばスクリプトが`/srv/cgi/tools/test.py`ならcwdは`/srv/cgi/tools`とし、Pythonの`open("data.txt")`は`/srv/cgi/tools/data.txt`を参照する。PATH_INFOはcwdの決定に使わず、cwdを指定するConfig項目も追加しない。

課題PDFは相対ファイルアクセスのために正しいディレクトリでCGIを実行することを要求し、[RFC 3875 §7.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-7.2)もUNIXでスクリプトの置かれたディレクトリをcwdにすることを推奨している。実行パス・cwdの受け渡しAPIと担当境界はCGI-91〜98で合意済み。chdir失敗時の検出・終了はCGI-120〜127の現時点の採用案に従う。

### 後続パスの合意済みの動作

**確定(CGI-25)**: C-24・C-25に従ってデコード・正規化し、locationを選択した後、そのlocationのroot/aliasで対応するパスを先頭から辿る。設定拡張子で終わる実在する通常ファイルに最初に到達したところをスクリプトとし、それより後ろをPATH_INFOとして渡す。例えばtest.pyが通常ファイルなら、`/cgi/test.py/extra/a.py`でも実行対象はtest.py、PATH_INFOは`/extra/a.py`となる。tools.pyがディレクトリなら、名前の末尾が.pyでも実行せず、その配下のパスを辿る。後続部分はファイルとして探索しない。設定されたindexにもCGI-10の判定を適用する。

この特定手順はチームの採用仕様。RFC 3875 §3.2–3.3はパスをスクリプトと後続部分に分ける考え方を示し、具体的な対応付けはサーバーの設定・実装に委ねている。実行パス・cwdの動作はCGI-26で確定済み。PATH_INFOがない場合の空値/省略はCGI-29で確定済み。アクセス失敗の分類はCGI-84〜90に従う。

次の例では、CGIを有効にしたlocationで `.py` を設定し、対応する `test.py` が実在する通常ファイルであることを前提とする。

| 要求 | 実行するスクリプト | SCRIPT_NAME | PATH_INFOの内容 | QUERY_STRING |
|---|---|---|---|---|
| `/cgi/test.py` | `test.py` | `/cgi/test.py` | 空文字列 | 空文字列 |
| `/cgi/test.py/` | `test.py` | `/cgi/test.py` | `/` | 空文字列 |
| `/cgi/test.py/extra` | `test.py` | `/cgi/test.py` | `/extra` | 空文字列 |
| `/cgi/test.py/extra/item` | `test.py` | `/cgi/test.py` | `/extra/item` | 空文字列 |
| `/cgi/test.py/extra?name=taro` | `test.py` | `/cgi/test.py` | `/extra` | `name=taro` |

SCRIPT_NAMEはURL上のパスであり、実行ファイルのディスク上の絶対パスとは区別する。後続部分をディスク上のファイルとして探す処理はWebservに追加しない。追加パスなしの場合もPATH_INFOを省略せず、空文字列で設定する(CGI-29)。

基本的な後続パス対応はチームの設計判断。課題PDFの「リクエスト情報・引数をCGIから利用できるようにする」という要件に沿わせるために採用したもので、PDFにPATH_INFO対応が独立した必須項目として明記されているという意味ではない。

### CGI出力の合意済みの動作と根拠

| 内容 | 根拠・位置付け |
|---|---|
| 長さがないCGI出力をEOFまで読む | 課題PDFの明示要件 |
| Content-Type・空行・本文を解釈する | 通常のCGI文書応答の基本。RFC 3875 §6.2.1 |
| Statusを反映し、通常の文書応答で省略時は200とする | RFC 3875 §6.2.1・§6.3.3 |
| ヘッダーのLF/CRLFを両方受理する | UNIX向けCGI仕様に沿う。RFC 3875 §7.2 |
| 絶対URLのLocationだけなら302にする | RFC 3875 §6.2.3のクライアント向けリダイレクト |
| 全出力を上限付きで蓄積し、EOFと子の終了を確認してからHTTP応答を作る | チームの確定済み実装方針(CGI-04)。課題が一括蓄積を指定しているわけではない |
| Content-Lengthと本文の実長が不一致なら502にする | チームのエラー処理方針として今回確定。課題指定のエラーコードではない |
| CGIの内部リダイレクトを省く | 実装量を抑えるため今回確定した機能制限。RFC 3875 §6.2.2にある機能を対象外とする |

Content-Lengthはヘッダーを除いた本文のバイト数。本文に含まれる空行は本文の終端ではない。EOFも出力中の文字列ではなく、出力側が閉じられ、残りのデータを読み終えた状態を指す。CGIのHTTP Statusと、子プロセスの終了コードは別々に扱う。

CGI-17で生成するContent-Lengthは、通常の本文付き応答についての規則。204等の本文・Content-Lengthの可否は既存のHTTP規則を維持する。

### エラー・上限・タイムアウトの確定内容

| 状況 | 応答・処理 |
|---|---|
| スクリプトなし / 読み取り不可 | 404 / 403 |
| サーバー側のpipe・fork失敗 | 作成済みの資源を片付け、500 |
| exec失敗・子の異常終了・不正なCGI出力 | 502 |
| CGIヘッダーまたは出力全体のサイズ上限超過 | CGIを打ち切り、fdの後始末・子の回収を行って502 |
| 起動から10秒超過 | CGIを打ち切り、fdの後始末・子の回収を行って504。出力が続いていても延長しない |

| 項目 | 採用する値・方針 |
|---|---|
| CGIヘッダー上限(1件あたり) | 8 KiB = 8,192バイト |
| CGI出力全体の上限(1件あたり、ヘッダーを含むstdout) | 8 MiB = 8,388,608バイト |
| CGI同時実行数の固定上限 | 設けない |
| CGIの時間制限 | 起動から10秒。出力による延長なし |
| 入力ボディ上限 | 既存の `client_max_body_size` を使う |

サイズの数値は課題指定値ではなく、チームが採用した初期値。出力を全て蓄積した後ではなく、読み取り中にサイズ上限を検査する。同時実行数の固定上限を設けない方針でも、起動失敗・切断時の後始末と子プロセスの回収は行う。

タイムアウトや出力上限超過で子を終了させた場合は、最初の失敗原因を保持する。タイムアウト後の子の異常終了を理由に504を502へ上書きしない。CGIが正常に返した `Status: 404` 等は、子プロセス自体の異常終了とは区別する。

### 維持する課題要件・既存の決定

- 課題要件: 拡張子によるCGI実行、リクエスト情報・引数の受け渡し、chunkedの復号、入力/出力のEOFの扱い、相対ファイルにアクセスできる作業ディレクトリ、最低1種類のCGIを満たす。
- 課題要件: 親サーバーはノンブロッキングで動かし、待ちが発生するパイプI/Oも同じpoll等で管理する。入力書き込みと出力読み取りを並行して進め、部分読み書き、切断、子プロセスの後始末を省かない。
- 既決定: CGI起動から10秒でkill + 504。出力があっても期限を延長しない(CGI-22)。
- 既決定: `SERVER_PROTOCOL` はCGI-33に従い受信した要求のHTTPバージョンを必ず設定する。chunkedの場合も `CONTENT_LENGTH` はCGI-31に従い、復号後の本文が1バイト以上ならそのバイト数、0バイトなら空文字列。入力を全て書いたら親側のstdin用パイプを閉じる。
- 既決定: HTTP/1.0で応答し、1接続1要求、`Connection: close`。本文とContent-Lengthの可否は204等のステータスに応じて既存のHTTP規則を使う。
- Configの既決定: `root` / `alias` の意味と、設定中の相対パスを設定ファイルのディレクトリ基準で解決する規則は維持する。CGI実行時の作業ディレクトリとは区別する。

### CGIの合意状況(10項目)

CGIの残件を10項目にまとめて1項目ずつ合意した記録。全項目の合意は完了している。CGI-IDは決定の参照に使い、実装・検証の進捗は04で管理する。

| 順番 | 状態 | 決める範囲 |
|---|---|---|
| 1/10 | 確定 | 出力ヘッダーの基本構文・値、Content-Type・Statusの検証(CGI-56〜61) |
| 2/10 | 確定 | Locationの検証とリダイレクト全体の扱い(CGI-62〜66) |
| 3/10 | 確定 | Content-Length・その他の出力ヘッダー・HTTP応答への変換(CGI-67〜74)。1xxの扱い、本文とContent-Lengthの可否も含む |
| 4/10 | 確定 | CGIへ渡す入力ヘッダーの残る規則(CGI-75〜83) |
| 5/10 | 確定 | ファイル・パスの検証とアクセス失敗(CGI-84〜90) |
| 6/10 | 確定 | 担当境界と受け渡しAPI(実行パス・cwdを含む、CGI-91〜98) |
| 7/10 | 確定 | データ・環境変数の所有権と寿命(CGI-99〜106) |
| 8/10 | 確定 | 切断・終了時の後始末とfd継承(CGI-107〜118) |
| 9/10 | 現時点の採用案・検証後に見直し可 | 子の起動失敗の検出・終了方法と時間計測(CGI-119〜127) |
| 10/10 | 確定 | エラーの受け渡しと結合時の完了条件(CGI-128〜135) |

<a id="cgi-api"></a>

2026-11-16更新: CGI有効locationへの許可済みPOSTで対象が実ディレクトリなら、末尾/補完リダイレクトをせず設定indexを探し、CGI対象なら元のPOST本文を渡して実行する。別locationへ再ルーティングしない。実在する対象または選択indexがCGI非対象なら405とし、GET許可ありならAllow: GET、POSTのみ許可なら空のAllowを返す。処理できないPOSTはAllowへ含めない。要求対象のディレクトリは実在するが設定indexが不在なら403とし、autoindexがonでも一覧へ進まない。対象ディレクトリ自体の不在は既存どおり404とする。[index不在時の正本](../Requirements.md#cgi-post-missing-index)を参照する。[CGI非対象POSTの正本](../Requirements.md#cgi-post-non-script)も参照する。[ディレクトリPOSTの正本](../Requirements.md#cgi-post-directory)を参照する。

2026-11-16更新: Router内のprivateな補助型CgiTargetをscriptPath/scriptName/pathInfoの3文字列で採用した。GET/POST処理内のローカル変数とし、fd・本文・Config参照を持たせず、Pythonパス/cwdも重複保持しない。CgiRequestなどの公開共有型は変更しない。[補助型の正本](../Requirements.md#router-target-api)を参照する。findCgiScriptの宣言と返却契約も採用済み。戻り値0かつfound=trueでtarget有効、0かつfalseはCGI非該当、HTTPエラーコードかつfalseは確認失敗で通常配信へ切り替えない。index選択はこの関数へ含めずGET/POSTハンドラーで行う。prepareCgiRequestもprivateなconst関数として宣言を採用済み。要求/location/接続情報/確定済みtargetからCgiRequestを生成し、成功0のときだけ出力が有効、検証失敗はHTTPエラーコードを返す。対象の再探索・Pythonの要求ごとの再検査・fork/execは行わない。GET/POST/DELETEハンドラーにはServerConfigを渡さない方針も2026-11-16に合意済み。メソッド許可/location選択はRouter本体、error_pageの設定参照は後続のClient側とし、ハンドラーは選択済みLocationConfigを使う。3ハンドラーの宣言も採用済み。privateなconst関数としてRouteResultを返し、GETはrequest/location/connectionからresponse/cgi/file/autoindex、POSTは同入力からresponse/cgi/write、DELETEはrequest/locationからresponseだけを出力する。戻り値に対応する出力だけが有効。DELETEは成功/失敗ともRESPONSE_READYとし、エラー本文は後続の共通送信準備で確定する。その他の共通補助関数の正式な宣言は提案段階。

### 担当境界と受け渡しAPIの合意事項(項目6/10)

**確定(CGI-91〜98、項目6/10)**: 以下の担当境界・型・APIで受け渡す。課題PDFとRFCはCGIへの入力・出力を定めるが、Webserv内部のクラス名・担当・APIは指定しない。

2026-11-15更新: 通常ファイル読み取りの分割に合わせ、RouteResultにFILE_READ_REQUIRED、routeの末尾にFileReadRequest出力引数を追加した。2026-11-16には分割保存用のFILE_WRITE_REQUIREDとFileWriteRequest出力引数も末尾へ追加した。本節を更新後の共有宣言の正本とする。CGIの要求/結果の型とプロセス側APIは維持する。FileReadTaskの所有・読み取り方式と未決定事項は[Requirementsの設計修正](../Requirements.md#file-read-design)を参照する。error_pageはClientの値メンバーHttpResponse pendingResponse_が元の応答を保持し、HttpResponse::prepareErrorPageで読み取り指示を作る方針・APIを合意済み。[準備APIの正本](../Requirements.md#error-page-prepare)を参照する。読み取り結果を受けるHttpResponse::finishErrorPage(bool, const std::string&)も合意済み。成功なら取得本文、失敗なら元コードの内蔵HTMLで確定する。makeError(code)はコードと標準説明文の元応答だけを作り、設定を受け取らず、本文生成/読み取りを行わない。生成したエラーと検証済みCGI応答を共通のprepareErrorPage経路へ渡す。RESPONSE_READYも本文確定前のエラー応答を含むため、直ちに送信できる保証とはしない。prepareErrorPageはClient側の共通送信準備で1回呼び、finishErrorPage後は再準備せず、ClientのprivateなbeginWritingResponseでserialize・送信バッファ準備を行い、受け渡し成功直後にpendingResponse_.reset()で元応答の保持領域を解放する。送信期限の起点はバッファ準備時とし、WRITING_RESPONSEへ進む。実際のsendはpoll通知後に行う。旧applyErrorPage/readErrorPageは廃止する。共通の入口はClient::prepareResponse(const HttpResponse&)として合意済み。応答をpendingResponse_へコピー後にprepareErrorPageを1回呼び、falseならserialize/送信、trueならERROR_PAGEで読み取りへ進む。std::bad_alloc時は資源回収と接続closeへ進め、500や内蔵HTMLの再生成は行わない。具体的な捕捉位置/回収経路などは設計継続中であり、下記の既存経路の説明を新方式の完成済みAPIとは扱わない。

| 担当 | 処理 |
|---|---|
| rysato / HttpRequest・Router | 要求の構文・重複を検証し、location・スクリプト・cwdを解決してCGI環境の内容を生成する。完成済み本文とともにCgiRequestにまとめる。戻ってきたstdoutをCGIとして解析・検証しHttpResponseを生成する |
| tasugiya / EventLoop・CgiExecutor・CgiProcess | 接続のIPv4・実際のサーバーIPv4とポートを取得しConnectionInfoで渡す。CgiRequestを使って起動し、pollでパイプI/O・上限・期限・子の終了を管理する。成功時のstdoutか失敗時のコードをCgiResultで返す |

EventLoopは接続先情報の取得に成功した接続をClientとして登録する。Routerには有効なConnectionInfoを渡す。[接続先情報の取得失敗時の契約](../Requirements.md#getsockname-failure)を参照する。

プロセス側はConfigのlocationやHTTPヘッダーを再解釈しない。argvは既存のPython絶対パスとスクリプト絶対パスから生成し、別の可変引数リストは受け渡さない。親のcwdは変更しない。IPv4の文字列化は4オクテットの十進表記とし、DNSや許可関数一覧にないinet_ntop/inet_ntoaを追加しない。

```cpp
struct ConnectionInfo {
    std::string remoteAddr;       // 接続元IPv4、ポートなし
    std::string serverAddr;       // 実際の接続先IPv4、0.0.0.0で代用しない
    unsigned short serverPort;    // 実際のlistenポート
};

struct CgiRequest {
    std::string interpreterPath;  // Config解決済みPython絶対パス
    std::string scriptPath;       // 解決・検証済みスクリプト絶対パス
    std::string workingDirectory; // scriptPathの親ディレクトリ
    std::vector<std::string> environment; // NAME=value。省略規則を反映済み
    std::string body;             // chunked復号済み、バイナリを保持
};

struct CgiResult {
    std::string stdoutData;       // 正常終了時のヘッダーを含む出力全体
    int errorStatus;              // 0: 実行成功、非0: サーバー側のエラーコード
};

struct FileReadRequest {
    std::string path;         // 読み取り対象の絶対パス
    std::string contentType;  // 配信時のContent-Type
};

// 2026-11-16追加。bodyはタスクへswapで移し、引き渡し後は空とする。
struct FileWriteRequest {
    std::string path;      // 保存先の絶対パス
    std::string filename;  // 検証済み保存名。成功時の結果HTMLに使用
    std::string body;      // multipartから取り出したファイル本文。タスクへの引き渡しで消費
};

struct AutoindexRequest {
    std::string path;     // 対象ディレクトリの絶対パス
    std::string urlPath;  // リンク生成用の正規化済みURLパス
};

enum RouteResult {
    RESPONSE_READY, CGI_REQUIRED, FILE_READ_REQUIRED, FILE_WRITE_REQUIRED,
    AUTOINDEX_REQUIRED
};

// Router(rysato)。CGI起動前の拒否もRESPONSE_READYでHTTP応答を返す。
RouteResult Router::route(const HttpRequest& request,
                         const ServerConfig& config,
                         const ConnectionInfo& connection,
                         HttpResponse& response, CgiRequest& cgi,
                         FileReadRequest& file, FileWriteRequest& write,
                         AutoindexRequest& autoindex);
// RESPONSE_READYならresponse、CGI_REQUIREDならcgi、
// FILE_READ_REQUIREDならfile、FILE_WRITE_REQUIREDならwrite、
// AUTOINDEX_REQUIREDならautoindexだけを有効とする。

// CgiExecutor(tasugiya)。0は起動手続き成功、非0は開始時のエラーコード。
// CGIの完了を待たずに返り、以後はEventLoopが進行する。
int CgiExecutor::start(const CgiRequest& request, CgiProcess& process);

// CgiProcess(tasugiya)。結果が未確定ならfalse。確定後に一度だけtrue。
bool CgiProcess::takeResult(CgiResult& result);

// Router(rysato)。実行成功後のstdoutを検証し、0または不正出力の502を返す。
// 0のときだけresponseが有効。成功後に共通のerror_page準備経路へ渡す。
int Router::parseCgiOutput(const std::string& output, HttpResponse& response);
```

2026-11-19合意: AUTOINDEX_REQUIREDではAutoindexRequest(path/urlPath)だけが有効。Routerは既存のindex優先/autoindex設定に従って一覧生成を選び、Clientが値メンバーのAutoindexTaskを開始する。routeとhandleGetの末尾にAutoindexRequest& autoindexを追加する。指示はfdやDIR*を持たず、タスクは必要情報を値で保持して一時指示への参照を残さない。開始APIはint AutoindexTask::start(const AutoindexRequest&)を採用する。path/urlPathをタスクへコピーしてopendirし、成功0、opendir失敗はHTTP-31の404/403/414/500を返す。列挙/ソート/HTML生成は後続のstepで進める。進行はStepResult step()を採用し、内部で列挙/ソート/HTML生成を段階的に進める。一定量でIN_PROGRESS、全工程とclosedir成功でCOMPLETED、HTTP-31/32の通常の生成失敗でFAILEDを返す。取得APIはbool takeBody(std::string&)とし、完了後の初回だけHTML本文をswapで渡してtrue、それ以外は出力不変でfalseを返す。Client側は200/text/html応答を組み立てる。ClientはtakeBodyで取得した完成HTMLをHttpResponse::setBodyBySwap(body, "text/html")で応答へ移し、prepareResponseBySwapへ渡す。応答全体をpendingResponse_とswapし、共通送信準備へ進む。[応答の所有権移譲](../Requirements.md#response-ownership-handoff)を参照する。引き渡しに成功するとbodyは空になる。[本文の引き渡し](../Requirements.md#autoindex-response-body-swap)を参照する。ディレクトリは列挙の正常終端直後/ソート前にclosedirし、成功後にソートへ進む。DIR*を所有状態から外して一度だけ閉じ、失敗は取得済み一覧を破棄してFAILED/500とする。列挙は1回のstepでreaddir最大32回の採用案とし、.と../終端/失敗の呼び出しも数える。終端/失敗/上限超過で打ち切り、終端未確認なら保持してIN_PROGRESSを返す。段階完了時は次段階の開始状態にしてIN_PROGRESSで戻る。列挙終端/closedir成功後のソートと、ソート完了後のHTML生成は次周から進める。空/1項目も同じ段階切り替えとし、HTML生成完了時は完成本文だけを保持して名前/番号列/作業領域/パスとURL/不要な継続情報を解放または初期化し、DONEでCOMPLETEDを返す。ソートは合計256操作、HTML生成は実追加最大8 KiBの採用案とする。回収APIはvoid cancel()を採用する。保持DIR*をNULLにして一度だけclosedirし、失敗でも再試行しない。名前/番号列/作業領域/HTML/保持パスとURL/位置を解放または初期化して未開始へ戻す。再呼び出し可能で、takeBodyは出力不変のfalseとなり、デストラクターも同じ回収を例外を外へ出さず行う。内部状態はprivateなPhase(IDLE/ENUMERATING/SORTING/BUILDING/DONE/ERROR/TAKEN)を採用する。初期/cancel後はIDLE、start成功でENUMERATING、列挙とclose成功でSORTING、ソート完了でBUILDING、全工程成功でDONE、本文取得でTAKEN、通常の失敗でERRORとする。startはIDLEでのみ開始し、それ以外は500を返してPhase/DIR*/保持データ/入力指示を維持する。拒否の500でERRORへ変更せず、再利用前にcancelでIDLEへ戻す。2026-11-20合意でIDLEからの通常の開始失敗は、取得済みDIR*をNULLにして一度だけcloseし、タスクの保持情報/本文/作業領域を解放して継続位置を初期化し、ERRORで元の開始失敗コードを返す。入力指示は全フィールドを維持し、後続のcancel/破棄で二重回収しない。stepはENUMERATING/SORTING/BUILDINGで該当段階を進める。DONEは処理せずCOMPLETED、ERROR/IDLE/TAKENは処理せずFAILEDを返し、いずれも状態/保持データを維持する。IDLE/TAKENでERRORへ変更せず、DONEの本文取得は初回だけとする。通常の生成失敗はstep内でDIR*の一度だけのcloseと名前/番号列/作業領域/HTML/保持情報の解放を済ませ、ERRORでFAILEDを返す。Clientは500を共通送信準備へ渡し、再step/cancel/破棄で二重回収しない。2026-11-20合意でstart成功後はClientのGENERATING_AUTOINDEXへ進み、IN_PROGRESSは維持、COMPLETEDはtakeBodyから200/text/html応答を生成してprepareResponseへ渡す。開始失敗は返却コード、生成失敗は500を共通送信準備へ渡す。2026-11-20合意でautoindexもstart直前から全段階通算60秒の固定期限、開始周のstep省略/次周から各周最大1回/poll待ち0/startとstep前後の確認/COMPLETED優先を適用する。期限超過はcancel後500、計時失敗はcancel後CLOSINGへ進める。error_pageは新しい60秒を使う。異常経路の具体的な組み込みは後続設計で定める。

FILE_READ_REQUIREDではRouterが対象の解決・種別/権限確認までを行い、FileReadRequestへパスとContent-Typeを値として格納する。FileReadRequest自体はfdを持たない。Clientの値メンバーfileTask_として保持するFileReadTaskがopen/read/closeを担当し、指示の必要な情報を自分で保持する。FileReadTask/Clientはコピー禁止。未開始のタスクはfd=-1・空本文とし、Clientと一緒に生成・破棄する。一時的なFileReadRequestの参照・ポインターは残さない。この結果でCGIを起動したり、まだ無効なresponseを送信したりしない。

実行成功とは正常な子の終了とstdoutのEOFが確認できたことを指し、出力内容の妥当性はparseCgiOutputで別途判定する。失敗時は部分的なstdoutをHTTPへ送らない。生のwaitpidの終了状態はtasugiya側で解釈し、HTTP側へ渡す必須データを増やさない。失敗原因とコードの対応・error_page適用は10/10のCGI-128〜135に従い、結果を確定できる後始末の条件は8/10のCGI-107〜118で確定済み。

既存のHttpRequestの単一値getterだけでは重複・全行処理を実装できないため、`const std::vector<std::pair<std::string, std::string> >& getHeaders() const`も公開する。名前はASCII小文字、値は合意済みの前後SP/HTAB除去後で、全行を保持する。辞書への上書き・結合で重複を失わず、Connectionの全行処理・転送対象の重複検査に用いる。

HttpResponseには`void setStatus(int code, const std::string& reason)`を追加し、CGIの検証済み説明文(空も含む)をそのままステータス行に反映する。既存の`setStatus(int)`はサーバーが生成する応答の標準文言に使う。CGIの出力検証規則を変更する合意ではない。

CgiRequest/CgiResultの所有権・コピー/参照・envpへの変換と寿命は7/10のCGI-99〜106で確定済み。起動時の具体的な失敗検出は9/10に従う。この合意はCGIの受け渡しに必要な公開部分を定める。HTTP全体の責務・公開API・保持はHTTP-59〜64で合意済みで、[Requirements](../Requirements.md#http-api)を参照する。I/O・全体メモリはHTTP-65〜74で合意済み(採用案を含む)。正本は [Requirementsのネットワーク境界](../Requirements.md#http-network-boundary)。

根拠: [RFC 3875 §3.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-3.1)のサーバーの責任、[§4](https://www.rfc-editor.org/rfc/rfc3875.html#section-4)の入力、[§6.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.1)の出力処理、および[Linux execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)のargv/envpを踏まえた担当・APIのチーム合意。

### データ・環境変数の所有権と寿命の合意事項(項目7/10)

**確定(CGI-99〜106、項目7/10)**: 6/10のAPIを維持して以下の所有権・寿命で管理する。課題PDFは接続・メモリの後始末を要求するが、内部データをコピーするか参照するかは指定していない。

| データ・資源 | 所有者と保持期間 |
|---|---|
| CgiRequest | Routerを呼ぶ側(EventLoop)が一時的な値として持つ。route完了からstartが返るまで保持し、その後は破棄できる。startは親側でrequestへの参照・ポインターを保存しない |
| CGIの入力本文 | start内でCgiProcessのstd::stringへコピーする。入力全体と書き込み済み位置(size_t)を保持し、部分書き込みごとに位置だけ進める。送信完了または打ち切りで不要になったら空stringとのswapでバッファを解放する |
| Python/script/cwd・環境変数 | CgiRequestのstring/vectorが所有する。fork前にargv/envpのポインター配列を作る。子ではfork時のメモリをexecveの成功または起動失敗による終了まで保持し、親はstartが返れば一時データを解放できる。CgiProcessに環境変数のコピーを持たせない |
| stdout蓄積 | CgiProcessが所有する。takeResultがtrueを返す時、成功ならstringのswapでCgiResultへ移す。失敗なら部分出力を破棄し、空のstdoutDataとエラーコードを返す |
| CgiResult | 呼び出し側(EventLoop)が所有する一時的な値。parseCgiOutputを呼ぶ間保持する。HttpResponseは生成したヘッダー・本文を自分の値として保持し、解析後にCgiResultを破棄できる |
| ClientとCgiProcess | EventLoopが両方を所有する。CgiProcessに対応先Clientへの非所有ポインターを1つ持たせる。Client破棄前にEventLoopが関連付けを外してNULLにし、切断後に古いClientを参照しない。クライアントfdだけで結果の返却先を探さない |

argvはPython・script・NULLの3要素、envpはNAME=valueへのポインター一覧と末尾NULLとする。文字列・環境変数一覧を先に完成させ、ポインター配列を作った後は元の文字列やvectorを変更しない。execveのchar*型へ合わせる場合も、c_str()の内容を書き換えない。ポインター配列はtasugiya側のstart内の固定配列/std::vector<char*>で管理し、文字列ごとのnew[]や手動解放を追加しない。

fork後は親と子のメモリが独立しているため、親がCgiRequestやenvpの一時配列を破棄しても、起動中の子の配列・文字列は失われない。execve成功時はプログラムのメモリが置き換わる。失敗時の終了方法は9/10で決める。[Linux fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)、[Linux execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)

CgiProcessはC++98のprivateな未定義コピーコンストラクター・代入演算子でコピー不可とする。pid・fdの所有者を増やさず、EventLoopの管理表にはポインターを保持する。ClientもEventLoop以外から削除しない。両者の削除・fd監視解除・子の回収の手順は8/10のCGI-107〜118に従う。CgiProcessからClientをdeleteする処理は作らない。ConfigはC-42の長寿命const参照を維持する。

getHeadersのconst参照はHttpRequestが存在し、ヘッダーを変更しない間だけ使用する。rysatoはrouteの処理中に環境変数の文字列へコピーし、startやCgiProcessにヘッダーの参照を残さない。コピーを減らすための本文所有権の移動・共有管理は追加せず、合意済みのconst CgiRequest&のstart APIを維持する。

takeResultはfalseなら出力引数を変更せず、trueなら以前の内容を置き換える。成功時のstdoutバッファはswapで引き渡し、同じ結果の2回目の取得はfalseとする。関連先Clientが消えた実行は後始末を完了して結果を破棄し、HTTP出力の解析・送信へ進まない。打ち切り・回収の具体的な手順は8/10のCGI-107〜118に従う。

### 切断・終了時の後始末とfd継承の合意事項(項目8/10)

**確定(CGI-107〜118、項目8/10)**: 7/10の所有関係を維持して以下の後始末を行う。課題PDFのノンブロッキング・クライアント切断処理・単一poll・要求を無期限に待たせない要件に従う。SIGKILLの選択、回収方法、fdの継承防止方法はチーム方針である。

| 状況 | 方針 |
|---|---|
| 通常の完了 | 入力完了で親側stdinパイプを閉じる。stdoutはEOFまで読み、子の正常終了・回収も確認してから結果を返す。子が先に終了しても、未読stdoutを捨てない |
| クライアント切断を検出 | 対応先Clientとの関連付けを外し、CGIを打ち切る。Clientへ応答を作らず、CGIの管理データは子の回収まで保持する |
| タイムアウト・出力上限・実行中のI/O異常 | パイプ監視を解除して閉じ、未回収の直接の子が残っていればSIGKILLで停止する。最初の失敗理由を保持し、回収後に接続が残っていれば確定済みのエラーを返す |
| SIGINTによるWebserv終了 | 新規受付・新規CGI起動を止め、listen・Clientを閉じて関連付けを外す。実行中CGIを打ち切り、すべての直接の子を回収してからWebservを終了する |

**親側の後始末の順序**

1. 中止/失敗を記録し、不要なHTTP応答の生成・送信を止める。最初の失敗理由を上書きしない。
2. パイプのpoll監視を解除し、保持fdを-1にしてから各fdを一度だけcloseする。入力・部分出力の不要なバッファも解放する。Linuxのcloseは再試行せず、再利用されたfdを誤って閉じない。
3. EventLoopがそのpidに対してwaitpid(pid, &status, WNOHANG)を呼ぶ。回収済みならkillしない。未回収なら必要なSIGKILLを送り、後続のループでもWNOHANGで回収する。停止成功後のkillを繰り返さず、回収済みpidを再利用してkillしない。
4. fdの後始末・子の回収・結果の引き渡しまたは破棄がすべて済んでからCgiProcessを削除する。

waitpidの回収はEventLoopだけが行い、pid>0の直接の子を指定する。戻り値0なら管理データを残して次のループへ進む。SIGCHLDのハンドラー・SIG_IGNによる自動回収は追加しない。失敗直後のerrnoがEINTRなら次のループで再試行し、ECHILDなら回収対象がないと記録して以後そのpidをkillせず、正常終了を確認できた扱いにもせず管理エラー500とする。その他の失敗も最初のエラーがなければ500を記録し、管理情報を残して再確認する。killのESRCHだけで回収済みとは扱わずwaitpidで確認し、その他の失敗は管理エラーとして保持する。read/write後のerrno参照には流用しない。

回収待ちでもブロッキングwaitpidは使わず、同じEventLoopを継続する。CGIが実行中または回収待ちならpollの待ち時間を最大100 msとし、パイプが閉じて新しいI/Oイベントがなくても終了・期限を定期確認する。SIGINTの終了処理も同じループの終了段階で行い、未回収の子がある間に単純にループを抜けない。デストラクターでの待機や別のpollループは追加しない。

**I/Oとイベントの境界**

- stdoutのPOLLHUPだけでは出力を捨てない。POLLIN/POLLHUPを受けたpollの後に読み進め、readの0をEOFとして記録する。1回の通知でread/writeを無制限に繰り返さず、各対象fdに1回ずつ処理し、残りは次のpollへ戻す。
- パイプのreadが負、または未送信本文に対するwriteが0以下なら502で打ち切る。errnoでEAGAIN/EINTR等を区別して再試行しない。正の部分書き込みは書き込み位置を進める。パイプのPOLLERR/POLLNVALも502とし、入力が残る書き込み側のPOLLHUPも打ち切り対象とする。
- 完成済みの要求でrecvが0になったことだけでは、応答を受け取れない完全切断と判断しない。送信側だけを閉じたクライアントにも応答できるよう、CGIを継続する。受信EOFを記録してPOLLIN監視を止め、POLLHUP/POLLERRやsendの失敗等で接続が使えないと分かった時に中止する。要求受信中のEOFは既存の不完全要求の処理に従う。
- WAITING_CGI中もクライアントのpoll通知を扱う。POLLIN時の追加データは固定サイズの一時バッファで読み捨て、次の要求として蓄積・解析しない。1接続1要求の仕様を維持する。
- 現在のpoll結果には閉じたfdの古い通知が残り得る。処理前に対応する管理オブジェクトと現fdが有効か確認し、閉じたものはスキップする。Client本体の削除はCLOSINGとしてその周の最後まで、CgiProcessの削除は現在のpoll結果を処理し終えた時点まで遅らせ、古い通知で解放済みオブジェクトや再利用されたfdを操作しない。

**子に残すfd**

listenソケット・acceptしたソケット・全CGIパイプに、生成時にfcntl(fd, F_SETFD, FD_CLOEXEC)を設定する。2026-11-18合意でFileReadTask/FileWriteTaskの通常ファイルfdにもopen直後に設定する。Pythonのexecve成功時に不要なfdが閉じるため、他Clientの接続や他CGIのパイプをPythonへ残さない。現在のCGIのstdin/stdoutはdup2で0/1へ接続し、元のパイプ端・使わない端を閉じる。0/1のFD_CLOEXECは解除する。同じfdへのdup2でも確実に解除し、継承するstderrの2もFD_CLOEXECを解除する。子の起動失敗時は9/10の方法で終了してfdを解放する。

このF_SETFDはLinux前提の方針。課題のfcntlの追加制限は「For MacOS only」の節であり、Linuxでの本方針に適用しない。起動時・accept時・CGI起動時の設定失敗は成功扱いにせず、作成済み資源を後始末する。失敗のコードはCGI-129・134に従う。全fdの一覧をstartへ渡す追加APIや/proc走査は設けない。

親のSIGPIPE無視は維持し、子はexecve前にSIGPIPEをSIG_DFLへ戻す。回収のためのSIGCHLD無視は行わない。シグナル設定・dup2・chdir・execve等の子側失敗の検出と終了は、項目9/10の現時点の採用案に従う。対象はWebservが直接起動したCGIの子で、孫プロセスの列挙・プロセスグループの管理は追加しない。

根拠: [Linux wait(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html)のWNOHANGと子の回収、[kill(2)](https://man7.org/linux/man-pages/man2/kill.2.html)、[pipe(7)](https://man7.org/linux/man-pages/man7/pipe.7.html)のEOFと不要なfd、[poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)のPOLLHUPと残データ、[close(2)](https://man7.org/linux/man-pages/man2/close.2.html)の再試行の扱い、[F_GETFD/F_SETFD](https://man7.org/linux/man-pages/man2/F_GETFD.2const.html)と[dup(2)](https://man7.org/linux/man-pages/man2/dup.2.html)のFD_CLOEXEC、および[signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)のSIGKILLと無視されたシグナルの継承。

### 子の起動失敗・終了方法と時間計測の現時点の採用案(項目9/10)

**制約は確定(CGI-119)**: ユーザー確認により_exit・clock_gettimeは使用不可。

**現時点の採用案(CGI-120〜127)**: 子の起動失敗時はmainまで戻り、子専用の明示的な資源解放とEventLoopのdeleteを終えてreturn 127で終了する。経過時間は/proc/uptimeから取得する。これは禁止関数を使わないための代替案として現時点で採用するものであり、実装・実行環境・実際の動作確認によって、方法や具体的な処理が変わる可能性がある。動作確認済み・最終固定の仕様とは扱わない。問題が分かった場合は、課題要件・禁止関数の制約とBare minimumの原則を満たす方法へ見直す。

検証する点は、子の起動失敗が親用のHTTP処理・cleanupを通らず502として回収されること、子が所有メモリ・一時資源を解放でき、デストラクターや出力バッファが兄弟CGIや親の接続へ影響しないこと、/proc/uptimeの利用可否・読取失敗時の挙動、実負荷下のタイムアウト精度・追加I/Oの影響とする。変更時は理由・影響する仕様/API・検証結果をRequirements/03/02/04に揃えて記録する。時間上限10秒や既存のエラー方針まで無条件に変更する意味ではない。

time・gettimeofday・clock・exit等の許可一覧にない外部関数の明示呼び出しも追加しない。

**起動失敗の通知と子の終了**

- fork前に本文コピー・argv/envp・パイプ・親側のノンブロッキング設定・FD_CLOEXECを準備する。親側のpipe/fork/fcntl失敗は500とし、作成済み資源を片付ける。子を作った後に失敗した場合の回収は8/10に従う。
- 子ではSIGPIPE復元、dup2・標準fdの継承設定・不要端close、chdir、execveを行う。いずれかが失敗したら、専用の「このプロセスはCGIの子」フラグを用いて、一時fdを解放してmainまでreturnで戻り、子専用の明示的解放後にmainのreturn 127で終了する。子での追加メモリ確保・例外による脱出・ログ出力は行わない。
- CgiExecutorに`static bool isChildProcess()`を追加する。forkが0を返した分岐でのみフラグをtrueにする。親のフラグはfalseのままとなる。子側のstart失敗の戻り値は502とし、呼び出し元はこのフラグを最初に確認して即座にreturnする。HTTPのエラー応答生成へ進めない。
- EventLoopのstart呼び出し箇所・その上位のイベント処理・runは、子のフラグがtrueなら残りのイベント処理、HTTP応答生成、親用cleanupを行わずmainへ戻る。mainを再度呼ぶのではなく、現在の呼び出し経路をreturnで抜ける。
- mainはEventLoopをポインターで所有する。子が起動に失敗した場合は、mainでEventLoop::cleanupInChild()を呼び、EventLoopをdeleteしてからreturn 127する。親は従来どおり8/10の停止・回収が済んでからEventLoopをdeleteする。Webservの停止・回収をグローバルなデストラクターや終了コールバックへ登録しない。
- cleanupInChildは子フラグがtrueの場合だけ実行する。まず複製された全CGIのpidを無効化し、Clientへの非所有ポインターを外す。その後、子に複製された管理対象fdをcloseし、CgiProcess・Client・EventLoopが所有するlisten管理オブジェクトをdeleteして管理表を空にする。メンバーのstring/vector等は所有オブジェクトの破棄で解放する。Config等の自動変数もmainまでのreturnとmain終了時の通常のデストラクターで解放する。各所有者に到達できないヒープ割当てを残さない。
- 子専用解放ではkill・waitpid・shutdown・read/write/recv/send・poll・応答生成・ログを呼ばない。forkで複製されたfdはcloseだけで解放する。shutdownは親と共有されたソケットの接続状態へ影響するため、ClientやServerSocketを含むデストラクターからも呼ばない。
- デストラクターは所有fd・メモリの解放だけを行い、CGIの停止・回収を行わない。親のkill/waitpidは明示的なEventLoop::cleanupに置き、親は回収済みのCgiProcessだけを削除する。子はpidを無効にした複製データを削除する。closeはCGI-110に従いfdを-1にして一度だけ行い、cleanupInChild後のdeleteで二重close/deleteを起こさない。
- 起動対象のCgiProcessはstartの呼び出し前にEventLoopの所有管理へ登録する。親の開始失敗時には既存の後始末規則で取り除き、子ではmainの解放経路から到達できるようにする。CgiExecutorが持つ未移管の起動用一時fdは、子の失敗分岐でcloseしてからreturnする。dup2で標準入力/出力へ移したパイプfdも子の失敗時に解放し、同じfdを管理データから再度closeしない。一時argv/envp・CgiRequest等の自動変数は呼び出し経路を通常のreturnで抜ける際に破棄し、イベント処理の途中でEventLoopや実行中の呼び出し先をdeleteしない。
- fork後の子専用解放は追加メモリ確保・ログ出力・例外送出を伴わない処理とする。バッファは既存所有オブジェクトのdelete/デストラクターで解放し、解放用の追加一覧・コピーを生成しない。
- fork前に使用するC++のcout/cerr/clogの出力バッファをflushして空にし、子の起動失敗後に親の未出力データを再出力しない。mainからのreturnは通常のデストラクター・標準ストリーム終了処理を伴うため、この分岐とバッファ管理を必須とする。明示的な_exit/exit呼び出しは使わない。
- 親は既存のwaitpidの終了状態で検出する。WIFEXITEDかつWEXITSTATUSが0なら実行成功候補とし、非0またはシグナル終了は502。ただし中止時の504等が先に記録済みなら上書きしない。Python自身の終了コード127も502でよく、exec失敗だけを別コードへ分ける必要がないため、起動失敗通知用の追加パイプは設けない。

```cpp
// CGI専用の子で起動が失敗した時だけ、runが途中でreturnする。
EventLoop* loop = new EventLoop(config);
loop->run();
if (CgiExecutor::isChildProcess()) {
    loop->cleanupInChild(); // 子の複製資源だけを解放。kill/waitpid/shutdownなし
    delete loop;
    return 127;
}
// 親: run内のCGI停止・回収が済んでいる
delete loop;
return 0;
```

この変更は、終了時にOSが回収することだけに依存せず、Webservが所有するメモリを子自身で解放するための合意。fork後はメモリ空間が分離されるため子のdeleteは親のオブジェクトを解放しないが、ソケットへの操作は共有資源に影響し得る。子専用解放でcloseだけを使い、shutdownや兄弟CGIへのkill/waitpidを避ける理由は[fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)のメモリ分離とfdの複製を参照する。実際のメモリ・fd解放と親への影響は04で検証する。

mainのreturnによる通常終了は[C++規格草案N1905 §3.6.1/5](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1905.pdf#page=65)を参考にする。これは後年の草案であり、C++98の新機能を追加する提案ではない。fork後のメモリ分離は[fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)、成功時に戻らないexecveは[execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)、終了状態の判定は[wait(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html)に従う。return時にも通常の終了処理が走るため、_exitと同じ性質の終了だとは扱わない。

**10秒の経過時間の取得**

- Linuxの固定パス`/proc/uptime`の最初の数値を現在の起動後経過秒数として使う。2番目のCPUアイドル時間は使わない。`12345.67 8901.23`なら現在値は12345.67秒となる。時刻関数や外部コマンドは呼ばず、設定項目も増やさない。
- `open(O_RDONLY | O_NONBLOCK)`・固定サイズ256バイトのread・closeで先頭の完全な数値トークンを取得する。毎回開き直して先頭を読み、保持fdやlseekを追加しない。数値はASCIIの非負小数として検証し、C++98のistringstream等で有限のdoubleへ変換する。readの0/負、完全な数値が読めない、変換失敗は計時失敗とし、read後のerrno参照・再試行は行わない。
- このprocファイルはLinuxで通常ファイル型(S_IFREG)として登録され、ソケット/パイプのように相手からのデータ到着を待つものではない。通常ファイルの読み取りとして扱い、単一pollのソケット/パイプ監視は維持する。課題PDFが/proc/uptimeを個別に指定・承認していると主張するものではなく、Linuxのファイル型と処理内容に基づくチーム案である。
- fork直前に現在値を読み、CgiProcessに開始値を保存する。EventLoopはCGI実行中に各周1回取得した現在値を全CGIの判定に共有し、`現在値 - 開始値 >= 10.0`なら504で打ち切る。途中の入出力で開始値を変更せず、EOF後に子が終了しない場合も対象とする。
- 8/10の最大100 msのpoll待ちを維持する。/proc/uptimeの表示精度は0.01秒であり、タイムアウトはこの精度とイベントループの確認周期に従って検出する。10秒ちょうどにシグナルが届く実時間保証はしない。
- CGI開始前の取得失敗は500で起動せず、実行中の取得失敗・値の逆行は該当する未完了CGIを500で打ち切る。先に記録した失敗は保持する。取得できない時に期限なしで継続するフォールバックは設けない。ConfigのC-40の検証項目を増やす合意ではない。
- Linuxの/procが利用できる実行環境を前提にする。壁時計ではなく起動後の経過時間を用い、サスペンド時間も含む。ファイルの再読みによるI/Oは増えるが、禁止された時刻関数を使わずに10秒を測る方法とする。通常接続の計時はHTTP-70で同じ取得方法を採用する。受信/送信の60秒はHTTP-68・69、ログの日時省略はHTTP-73に従う。起動後の経過時間からDateの暦日時を生成せず、HTTP-56の暫定案を維持する。

根拠: [Linux proc_uptime(5)](https://man7.org/linux/man-pages/man5/proc_uptime.5.html)の2つの数値とサスペンド時間、[カーネルのuptime.c](https://github.com/torvalds/linux/blob/master/fs/proc/uptime.c)の起動後時間と小数2桁の出力、[generic.cの通常ファイル登録](https://github.com/torvalds/linux/blob/master/fs/proc/generic.c)。明示的に使うopen/read/close・fork/dup2/chdir/execve/fcntl/signal/waitpidは課題PDFの許可一覧内。mainのreturnは言語の通常の終了方法であり、明示的なexitの追加使用ではない。

### エラーの受け渡しと結合時の完了条件の合意事項(項目10/10)

**確定(CGI-128〜135)**: 最後の10/10を合意済み。これで10項目の合意は完了。9/10の代替案の実環境検証と実装・結合試験はこれから行う。

課題PDFは未指定時のデフォルトエラーページを要求するが、CGIの正常なエラー応答を置き換えるか、内部のAPI・エラーの受け渡し方法までは指定しない。[RFC 3875 §6.3.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3.3)はCGIのエラーステータスとスクリプト自体の失敗を区別し、[§6.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.4)は本文を変更せず返すことを推奨する。本方針では確定済みConfig C-20の400〜599のエラーページ方針をCGIにも適用する。これはチームの選択であり、RFCや課題がCGI本文の置換を要求しているという意味ではない。

1. **受け渡しの順序**: rysato側のrouteが即時応答を返した場合はCGIを起動しない。tasugiya側のstart失敗はそのコードでmakeErrorを生成する。非同期のCgiResultはerrorStatusが非0なら部分stdoutを解析せずmakeErrorへ渡す。0ならrysato側のparseCgiOutputで全出力を検証し、解析失敗は502、成功はCGIの応答として扱う。失敗時のresponse出力引数は使用しない。
2. **失敗コード**: スクリプト未発見404・読み取り不可403・パス長414等の既存分類を維持する。親側のpipe/fork/fcntl・計時・管理の失敗は500、子側の準備/exec失敗・非0終了/シグナル終了・パイプI/O失敗・出力上限超過・不正出力は502、10秒経過は504。複数の原因が発生しても最初に確定したコードを維持する。正常に返されたStatus: 404等はCgiResultの実行失敗ではない。
3. **error_pageの適用**: 出力検証後の最終コードが400〜599なら、CGIが正常に返したエラーも含め、指定ファイルの本文へ置き換える。未指定・読み取り失敗なら内蔵HTMLにする。元のコードを維持し、有効なCGIのStatus説明文は空文字列も保持する。200〜399には適用しない。1xxは既決定のCGI-74により先に502となる。不正出力をerror_pageで覆って元のCGIコードのまま受理しない。
4. **置換する本文とヘッダー**: エラーページはHTMLとして扱い、Content-Typeはtext/html、Content-Lengthは置換後のバイト数で生成する。元本文の情報を残さないため、CGI由来のContent-*、ETag、Last-Modified、Digest、Repr-Digestを大小文字を区別せず除去する。この一律除去は最小構成のチーム方針。その他の既存転送対象ヘッダー(Allow、WWW-Authenticate、Set-Cookie等)は保持する。200〜399の応答にはこの除去を行わない。
5. **エラーページの読み取り**: 設定済みのファイルパスをそのまま使い、実行時に通常ファイルであることを確認して読む。空ファイルも読み取り成功なら空本文として採用する。失敗時は元コードの内蔵HTMLへ一度だけフォールバックし、再ルーティング・CGI実行・再帰的なエラー生成を行わない。CGIの8 MiB制限は元のstdoutに適用し、エラーページに新しいCGI専用上限は追加しない。
6. **内部の共通処理(2026-11-15更新)**: makeError(code)で元応答だけを作り、その後にprepareErrorPageで内蔵HTMLを確定するか読み取り指示を作る。読み取りを行った場合はfinishErrorPageで本文を確定する。正常なCGIの検証済み応答はmakeErrorで作り直さず同じ準備経路へ渡す。prepareErrorPageはClient側の共通送信準備に集約し、RouterやCGI出力パーサーから個別に呼ばない。falseならserialize/送信、trueならFileReadTaskへ進み、読み取り結果をfinishErrorPageで確定した後は再準備せず送信する。旧applyErrorPage/readErrorPageは廃止し、内蔵HTML生成と本文関連ヘッダー除去の補助関数を共用する。共通の入口はClient::prepareResponse(const HttpResponse&)とし、コピー保持後に準備を1回行う。error_page適用は一度だけとし、CGI専用に重複実装しない。
7. **応答を返せない場合**: クライアント切断時は停止・回収だけを行い、応答を生成しない。送信失敗後に別のエラー応答を追加送信しない。listen側のCLOEXEC設定失敗は起動失敗、acceptしたfdの設定失敗はその接続を閉じて監視へ登録しない。CGIパイプの設定失敗は上記の親500/子502の分類を使う。
8. **結合の完了条件**: 04のCGI受入項目を実装後に満たすことを条件とする。正常なGET/POST・chunked入力、入出力の並行処理、出力検証・エラーコード・上限・期限、切断/半閉鎖/SIGINT、fd継承・子の回収・寿命、CGI実行中の静的配信を確認する。加えて正常なCGIの404/599の置換、200の非置換、エラーページ未指定/読取失敗/空ファイル、元本文の符号化等の除去とAllow等の保持、不正な404出力が502になることを確認する。9/10の代替案は実環境で検証し、必要なら方法を見直す。10項目の合意完了と、未実装の結合試験合格は別に扱う。

内部リダイレクトを対象外とするため、本チームは「RFC 3875を参考に対応範囲を限定」とする。課題PDFに個別の対応範囲が明記されていない機能を、課題から免除されたとは断定しない。

### 合意済みの分担・受け渡しと未実施の結合テスト

担当・APIは項目6/10、所有権・寿命は7/10、後始末は8/10、起動・計時の採用案は9/10、エラー適用境界は10/10を参照する。以下は未実施の結合テストであり、未チェックは未合意を意味しない。受入項目の一覧は [04](04_testing_and_workplan.md) にまとめる。

- [ ] **共通の結合テスト**: GETのクエリ、POST/HTTP1.1 chunkedのボディ、cwdからの相対ファイル読み込み、パイプ容量を超える入出力、出力形式の異常、即時終了、タイムアウト、クライアント切断、各上限への到達を確認する。上記の後続パス4例、index.pyの実行、CGI用locationへのDELETEが405になること、アップロード先でCGIが実行されないことも含める。CGI実行中も静的配信でき、終了後にfdや未回収の子が残らないことを完了条件にする。
- [ ] **合意済み出力のテスト**: Status省略の文書応答が200、明示したStatusの反映、LF/CRLFの両形式、絶対URLのLocationのみで302、Content-LengthなしでEOFまでの受信、長さ一致時の成功・不一致時の502を確認する。本文中の空行も保持する。内部リダイレクトは実行されないことを確認し、ローカルLocationはCGI-65に従って502とする。
- [ ] **クエリと不正出力のテスト**: `?a+b` がQUERY_STRING=`a+b`として届き、スクリプトのコマンドライン引数に展開されないことを確認する。不正なCGI出力は補正せず502とする。重複・数値・Statusの境界ケースはCGI-52〜74と04の期待値に従う。
- [ ] **エラー・上限のテスト**: ヘッダー8 KiB・出力全体8 MiBの境界と超過時の502、継続的に出力するCGIでも起動から10秒で504になること、pipe/fork失敗時の500と後始末を確認する。OS資源が利用可能な環境では、17本以上の同時実行を件数だけで拒否しないことも確認する。
- [ ] **エラーページの結合テスト**: CGI-130〜133に従い、正常なCGIの404/599は本文置換、200は非置換、未指定/読取失敗は内蔵HTML、空ファイルは空本文となることを確認する。元のコード・CGI説明文の保持、元本文関連ヘッダーの除去とAllow等の保持、置換後の長さ、不正な404出力が502になることも確認する。

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
| C4–C5, C7 | S〜M | 設計は合意済み。C7の削除手段はHTTP-50のstd::removeに確定し、実装・試験は未完了 |
| C6(アップロード) | L | RFC読み込み+設計判断(方式・サニタイズ・メモリ) |
| C8–C10(CGI) | XL | 共同。tasugiya=C8、rysato=C9を主に調査し、C10の結合設計を合同で行う |
| C11 | M | 後回し可 |

2026-11-17合意: HttpRequest::releaseBody()はCOMPLETEで本文の保持領域を解放し、INCOMPLETE/ERRORでは全保持データと状態を維持して戻る。[本文解放APIの正本](../Requirements.md#request-release-body-api)を参照する。

2026-11-18合意: FileReadTask/FileWriteTaskはopen直後にF_SETFD/FD_CLOEXECを設定し、設定成功後にstart成功を返す。保存本文のswapも設定成功後とする。設定失敗はfdを一度だけ閉じ、保存で今回作成した途中ファイルを削除し、指示側本文を維持して500、読み取りはfalseを返す。CGI子の起動失敗では複製fd/メモリを解放し、親が保存中のファイルを子から削除しない。2026-11-19にpublicなvoid FileWriteTask::cleanupInChild()を採用した。複製fdを一度だけ閉じてfd=-1、保持データを解放し位置0/IDLEへ戻す。途中ファイル削除責任を外し、以後のcancel/破棄でも親のファイルを削除しない。再呼び出し可能で追加確保/ログ/外へ出る例外を避ける。2026-11-19にpublicなvoid Client::cleanupInChild()を採用し、EventLoopの子専用経路がClient破棄前に呼ぶ。Client内で読み取りcancel/保存cleanupInChild/複製ソケットfdのcloseを行い、残る所有メモリを続く破棄までに回収する。再呼び出しと破棄で二重close/削除を防ぐ。[継承防止の正本](../Requirements.md#file-task-cloexec)を参照する。

2026-11-21合意で、共通送信準備ではステータス行とヘッダーだけを生成し、保持済みの本文を別に送る。POLLOUT通知後に既存のsend上限を守り、ヘッダーを送り切ってから本文へ進む。送信期限は準備完了から通算60秒とする。2026-11-23合意で、送信中はClientのpendingResponse_に確定済み本文を保持し、そこから直接送る。送信完了または打ち切り後に本文と送信ヘッダーを回収する。HttpResponseのserializeHeaders()でステータス行・ヘッダー・終端の空行を生成し、getBody()で送信対象の本文をconst参照で取得する。本文の要否とContent-LengthはHttpResponseが決める。[送信用API](../Requirements.md#response-send-accessors)を参照する。
