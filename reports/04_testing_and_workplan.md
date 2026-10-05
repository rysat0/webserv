# 04 検証方法・進行計画・決定が必要な論点

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

### 確定事項から作る受け入れテスト

以下は [Requirements.md](../Requirements.md) と [03のCGI合意事項](03_config_cgi_upload.md) に合わせた検証計画。テストを実行済みという意味ではない。

| 対象 | 確認する動作 |
|---|---|
| HTTPの接続とバージョン | HTTP/1.0・必要なHTTP/1.1要求にHTTP/1.0で応答し、1接続1要求でcloseする。HTTP/1.1 chunked入力は復号する。HTTP/1.0のTransfer-Encoding、およびTransfer-EncodingとContent-Lengthの併記は400 |
| Expect | HTTP/1.1のExpectはヘッダー受信後に417(先に確定した400/413を優先)とし、100応答を送らない。HTTP/1.0の`Expect: 100-continue`は無視する |
| Configのパス | rootはURI全体を追加し、aliasはlocationのprefixを置換する。同じlocationで両方を明示した場合は記述順によらず起動エラー(C-08)。aliasはディレクトリ対応のみとし、prefixと値の末尾 `/` が片方または両方欠けていれば自動補完せず起動エラー(C-09)。相対パスは設定ファイルのディレクトリ基準。error_pageは直接ファイルを読み、未指定・読み取り失敗時は元のステータスの内蔵ページ |
| Configの省略 | index省略でindex.html、autoindex省略でoff、ボディ上限省略で1 MiB、配信locationのroot/alias両方省略で設定ファイル基準のhtml。alias明示時はrootを補完しない。明示値を優先し、不正な明示値は既定値で救済しない。listen省略時は起動権限に関係なく0.0.0.0:8000(C-39)。明示値を優先し、複数serverの省略による同一listen先はC-12で拒否。bind失敗時に別ポートへ変更しない。upload_store省略時はWebservによるアップロード保存を行わず、保存先を自動補完しない。cgi_extension省略時はCGIを実行せず、拡張子とPythonのパスを自動補完しない(C-07)。CGI用locationでupload_storeを省略してもPOSTボディをCGIへ渡せる |
| Configのlisten書式 | IPv4:portを受理する(C-10)。IPv4の各数値0〜255、ポート1〜65535の境界を確認し、範囲外・構成要素の欠落・ポートのみ・ホスト名・IPv6・追加オプションは起動エラーにする。書式の受理と実際のbind成功は区別する |
| Configのlisten指定回数 | 1server内でlistenが0回なら既定値、1回なら指定値を使う(C-11)。2回以上は同じ値でも異なる値でも起動エラー。serverを分けた8080/8081で異なるコンテンツを配信できることを確認する |
| Configのlisten競合・仮想ホスト | 同一IPv4:port、同じポートでの0.0.0.0/特定IP、listen省略で生じる競合は起動エラー(C-12)。異なるポート、異なる特定IP同士は設定として受理する。server_name指定は起動エラー。Hostでサイトを切り替えず、HTTP/1.1のHost欠落・重複・不正は引き続き400 |
| Configのボディ上限 | 正の十進整数のバイト数を受理し、省略時は1 MiB(C-13)。0・符号・小数・単位接尾辞・整数型の範囲超過は起動エラー。復号後ボディが上限と同じなら受理、1バイト超過なら413。multipartはファイル以外の区切り等もボディサイズに含める |
| Configの指定回数 | 通常の設定項目を同じブロックに2回書いたら、値が同じでも起動エラー(C-14)。別ブロックでの指定は可。同じserver内でerror_pageの異なるコード・locationの異なるprefixは受理し、同一コード・同一prefixは拒否する。allow_methodsの複数行を結合しない |
| Configのindex | ファイル名1個を受理し、省略時index.html(C-15)。複数候補・`/`を含むパス・`.` / `..` は起動エラー。index不在でも起動可。ディレクトリへのGETでindexがあれば配信/既存CGI判定、なければautoindexがonなら一覧・offなら403。ディレクトリ不在は404。読み取り不可のindexから一覧へ切り替えない |
| Configのautoindex | 小文字on/offの1引数だけ受理し、省略時off(C-16)。空・複数引数・ON/true/1は起動エラー。autoindex_format等の未対応設定も拒否する。onでもindexを優先し、index不在時はHTMLの一覧を返す。offでも読み取り可能な個別ファイルへのGETは配信する |
| Configのallow_methods | 大文字GET/POST/DELETEの空白区切り・重複なし・1個以上を受理する(C-17)。POST単独ならGETを自動追加しない。空・重複・小文字・カンマ区切り・HEAD/PUT等は起動エラー。記述順を変えても許可集合は同じ |
| Configのupload_store | 省略/offで保存無効、既存ディレクトリのパス1個で有効(C-18)。絶対・設定ファイル基準の相対パスを確認。空・複数引数・不在パス・通常ファイルは起動エラーとし、自動作成しない。GETの公開先を自動設定しない。保存失敗や命名規則の試験は別途 |
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
| HTTP要求ヘッダー名 | C-34: 1.0/1.1とも大小文字の異なる名前を個別に送って同じ名前として読む。ASCII英数字とRFC tokenの全記号を受理。空名・名前内の空白/非ASCII/スラッシュ・コロンなしは400。名前をASCII小文字で保持し、値のAbC123や値内のコロンは保持。重複はHostのC-28・C-30とContent-LengthのC-35に従い、その他は別途確定する |
| 要求のContent-Lengthの重複 | C-35: 1.0/1.1とも正常な1行は受理。同値/異値の2行・名前の大小文字違いの2行・10, 10や10, 20などカンマ入りは400。別受信に分かれた重複も検出し、ボディ完了を待たずエラー応答へ進む。結合や先勝ち/後勝ちで通さない |
| 長さ指定のないPOST | C-36: 1.0/1.1ともCL/TEが両方なければ空ボディでヘッダー終端時に受信完了し411にしない。Content-Length: 0も空。CGIへ進む場合はstdinがEOFになる。正のCL/chunkedは従来どおり受信し、不正値を省略扱いしない |
| 自動補完URLの再エンコード | C-29: 内部パス/hello world・/a+b・/name?part・/literal%20を、それぞれ/hello%20world/・/a%2Bb/・/name%3Fpart/・/literal%2520/にする。ASCII英数字・-._~・区切り/は維持し、その他はバイト単位の大文字%XX。?name=a+bや?x=%20は元のまま付ける。元要求/literal%2520は1回デコード→出力時1回エンコードで/literal%2520/となる。Configのreturnは変換しない |
| Configの基本文法 | server/location・`;`・`{}`・空白/タブ/改行区切り・`#`コメントを受理する。改行位置を変えても同じ設定として読む。引用符・エスケープ・変数・include・正規表現・入れ子location・http/events等の未対応構文、`;`の欠落・未閉じブロックは理由を表示して起動を中止する。reload機能は設けない |
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
| CGI出力のStatus検証 | CGI-61: ASCII数字3桁の100〜599がコードの範囲検査を通り、099・600・4桁・非数字は502。コード直後のSP必須、空の説明文の受理、説明文の使用、通常文書応答のStatus省略・空値時の200を確認する。1xxのHTTP応答への変換テストは残項目3/10の合意後に確定する |
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

### 課題文・運営への確認が必要(最優先。回答が設計を左右する)

| # | 論点 | 影響範囲 |
|---|---|---|
| Q1 | **許可関数にない関数の使用可否**: 時刻(`time`/`gettimeofday`/`strftime`、タイムアウトと`Date`ヘッダーとログに使用)、`unlink`/`remove`(DELETE)、`inet_ntop`/`inet_ntoa`、`stat`系以外のFS関数。C++標準ライブラリ(`<ctime>`,`<fstream>`,`<sstream>`等)側で代替できるか。CGI-119により_exit・clock_gettimeは使用不可とユーザー確認済み。CGIの代替案は9/10で現時点の案として採用し、実装・動作確認で見直し得る(CGI-120〜127)。Cookie用の`rand`等は必須範囲外 | N7, C7, C8, N1 |
| Q2 | errno禁止の範囲(read/write/recv/send後のみか、accept/poll/fork 等も含むか) | N5 |
| Q3 | 評価で想定されるアップロードの手段(ブラウザのフォーム/curl) | C6 |
| Q4 | `Requirements.md` の「norminette不要」「Linux限定」の最終確認(キャンパスルール) | 全体 |

### チーム内で決める(`Requirements.md` の未決定・新規論点)

| # | 論点 | 関連 |
|---|---|---|
| D1(確定) | Configの6項目はC-37〜C-42で合意済み。文法・location設定値・listen省略値・起動時ファイル検証・設定の組み合わせ・型/APIを05とRequirementsに揃える。実装・検証は別作業。HTTP/Router/CGIのAPIはD2/D11に残る | C1/C2/C3/05 |
| D2 | HTTP全体の残るAPI・内部実装。Routerの戻り値とCGI非同期結果の公開型・APIはCGI-91〜98で確定済み。所有権・寿命はCGI-99〜106で確定済み | C10 |
| D3 | HTTP要求の残る受付規則(absolute-form、Host/Content-Length以外の重複、値の残る検証規則等)。Content-Lengthの最大1行・同値重複/カンマ入り400はC-35、ヘッダー名のtoken検証・ASCII小文字保持はC-34、折り返し拒否・コロン直前の空白拒否・値の前後SP/HTAB除去はC-33で確定済み。要求行・ヘッダーのCRLF限定と不正改行400、分割されたCRLFの待機はC-31で確定済み。CGI出力ヘッダーのLF/CRLF受理は別の確定規則 | H1–H3 |
| D4 | 現行Configの許可メソッドはGET/POST/DELETEに限定(C-17)。将来HEAD/OPTIONSを追加するならHTTP/Configの両方を変更する。Range/条件付きリクエストをスコープ外にするかは未確定 | H6/C4 |
| D5 | HTTP/1.0の`Expect: 100-continue`以外のExpectの扱い。HTTP/1.1のExpectは即時417(先行する400/413を優先)、100応答なし。HTTP/1.0の100-continueは無視する方針まで確定済み | H9 |
| D6 | C-32: 要求行8 KiB・ヘッダー合計32 KiBと超過414/431は確定済み。既存Configの復号後ボディ上限・超過413も維持。残るのは受信/送信バッファの保持方法、chunkedの付加情報の制限、全接続合計の資源管理等。CGIはヘッダー8 KiB・stdout全体8 MiB、同時実行数の独自上限なしで確定済み | N4/N9 |
| D7 | ステータスコードの生成対象の確定(H7の分類結果) | H7 |
| D8 | アップロード方式(multipartのみ/生ボディ併用)とファイル名規則 | C6 |
| D9 | 413などエラー応答後の接続処理(読み捨て/shutdown/closeの手順)。1接続1要求・応答後closeは確定済み | N6/H4 |
| D10(確定) | C-12: 既定値補完後の同一IPv4:portと同じポートでの0.0.0.0/特定IPの競合は起動エラー。異なる特定IP同士は同じポートも設定可。server_name・Hostによる仮想ホスト選択は実装しない | N2/05 |
| D11 | CGIの10項目は合意完了(CGI-01〜135)。エラー適用境界・結合の完了条件は10/10のCGI-128〜135で確定済み。起動・計時のCGI-120〜127は現時点の代替案で、実装・実際の動作で見直し得る。今後は実装と本書のCGI受入項目・代替案の実環境検証を行う。合意完了を試験合格とは扱わない | C9/C12 |

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
| P4 仕様作成 | 「規則/採用方針/受け入れテスト」の3点セットで `specs/` 等に記述。Config(D1)は合意済みとして仕様へ反映し、HTTP/Routerの未決定(D2)を確定 | 仕様書一式、サンプル設定 |
| P5 相互レビュー | 互いの仕様を読み、実装前に矛盾(特にtasugiya↔rysatoの境界)を潰す | レビュー指摘ログ |

### 実装順序との対応(`Requirements.md` 実装順序 1–8)

| 実装フェーズ | 先に完了しておくべき仕様 |
|---|---|
| 1 ConfigParser / Hello World | C1, C2, N1, N2 |
| 2 EventLoop+Client | N3, N4, N5, N6, N7, N8 |
| 3 HttpRequestパーサー | H1, H2, H3, H4 |
| 4 Router+HttpResponse | H6, H7, H8, H10, C3, C4 |
| 5 エラー/autoindex/DELETE | H7, C5, C7, Q1回答 |
| 6 POST/chunked | H4, H5, C6 |
| 7 CGI | C8, C9, C10 |
| 8 タイムアウト/上限/負荷 | H9(1接続1要求), N7, N9, T1 |

**ポイント**: フェーズ7(CGI)の仕様は他と比べて結合が複雑なので、フェーズ4–6の実装中に仕様を詰めても間に合うが、**C10(非同期結果の返し方)はD2(シグネチャ)に影響するため、P4で先に確定**しておくこと。
