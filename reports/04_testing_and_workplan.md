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
| 不正なCGI出力 | 不正な値を推測・補正して受理せず502にする。具体的な重複・数値・Statusの境界ケースは03の検証規則を確定してから追加する |
| 対象外のCGI応答 | ローカルパスのLocationだけを受けても内部で再処理しない。その場合の応答コードや、LocationにStatus・本文を伴う形式の扱いは、03の未確定事項を決めてから期待値を追加する |

## T2 nginx との挙動比較 【共, R:S, D:S, ★★】

- 同じ設定意図のnginx設定を用意し、同一リクエストでステータス/ヘッダー/ボディを比較する手順。
- 差が出やすい点: 末尾スラッシュ補完、405の`Allow`、404/403判定、autoindexの見た目、エラーページ、`Server`/`Date`、HTTP/1.1と1.0の応答差、`Connection` ヘッダー。
- 意図的な差異は仕様に「nginxと異なる点」として記録(評価時の説明材料)。

---

## 決定が必要な論点リスト(仕様作成フェーズで解消する)

### 課題文・運営への確認が必要(最優先。回答が設計を左右する)

| # | 論点 | 影響範囲 |
|---|---|---|
| Q1 | **許可関数にない関数の使用可否**: 時刻(`time`/`gettimeofday`/`strftime`、タイムアウトと`Date`ヘッダーとログに使用)、`unlink`/`remove`(DELETE)、`_exit`、`inet_ntop`/`inet_ntoa`、`stat`系以外のFS関数。C++標準ライブラリ(`<ctime>`,`<fstream>`,`<sstream>`等)側で代替できるか。Cookie用の`rand`等は必須範囲外 | N7, C7, C8, N1 |
| Q2 | errno禁止の範囲(read/write/recv/send後のみか、accept/poll/fork 等も含むか) | N5 |
| Q3 | 評価で想定されるアップロードの手段(ブラウザのフォーム/curl) | C6 |
| Q4 | `Requirements.md` の「norminette不要」「Linux限定」の最終確認(キャンパスルール) | 全体 |

### チーム内で決める(`Requirements.md` の未決定・新規論点)

| # | 論点 | 関連 |
|---|---|---|
| D1(確定) | Configの6項目はC-37〜C-42で合意済み。文法・location設定値・listen省略値・起動時ファイル検証・設定の組み合わせ・型/APIを05とRequirementsに揃える。実装・検証は別作業。HTTP/Router/CGIのAPIはD2/D11に残る | C1/C2/C3/05 |
| D2 | HttpRequest/HttpResponse の具体的シグネチャ(特にRouterの戻り値とCGI非同期結果の扱い) | C10 |
| D3 | HTTP要求の残る受付規則(absolute-form、Host/Content-Length以外の重複、値の残る検証規則等)。Content-Lengthの最大1行・同値重複/カンマ入り400はC-35、ヘッダー名のtoken検証・ASCII小文字保持はC-34、折り返し拒否・コロン直前の空白拒否・値の前後SP/HTAB除去はC-33で確定済み。要求行・ヘッダーのCRLF限定と不正改行400、分割されたCRLFの待機はC-31で確定済み。CGI出力ヘッダーのLF/CRLF受理は別の確定規則 | H1–H3 |
| D4 | 現行Configの許可メソッドはGET/POST/DELETEに限定(C-17)。将来HEAD/OPTIONSを追加するならHTTP/Configの両方を変更する。Range/条件付きリクエストをスコープ外にするかは未確定 | H6/C4 |
| D5 | HTTP/1.0の`Expect: 100-continue`以外のExpectの扱い。HTTP/1.1のExpectは即時417(先行する400/413を優先)、100応答なし。HTTP/1.0の100-continueは無視する方針まで確定済み | H9 |
| D6 | C-32: 要求行8 KiB・ヘッダー合計32 KiBと超過414/431は確定済み。既存Configの復号後ボディ上限・超過413も維持。残るのは受信/送信バッファの保持方法、chunkedの付加情報の制限、全接続合計の資源管理等。CGIはヘッダー8 KiB・stdout全体8 MiB、同時実行数の独自上限なしで確定済み | N4/N9 |
| D7 | ステータスコードの生成対象の確定(H7の分類結果) | H7 |
| D8 | アップロード方式(multipartのみ/生ボディ併用)とファイル名規則 | C6 |
| D9 | 413などエラー応答後の接続処理(読み捨て/shutdown/closeの手順)。1接続1要求・応答後closeは確定済み | N6/H4 |
| D10(確定) | C-12: 既定値補完後の同一IPv4:portと同じポートでの0.0.0.0/特定IPの競合は起動エラー。異なる特定IP同士は同じポートも設定可。server_name・Hostによる仮想ホスト選択は実装しない | N2/05 |
| D11 | CGIの残る詳細: ファイルパスの結合/検証の残る詳細、実行パス・cwd、環境変数の空値/省略とHTTP_*の転送、出力の検証規則、Location+Status+本文、対象外の内部リダイレクト出力への応答コード、stderrの扱い。スクリプト特定手順はCGI-25で確定済み。C-24・C-25のURL処理とC-26のsymlink運用前提、基本的PATH_INFO対応・内部リダイレクトの対象外化・エラーと上限の対応表・クエリ引数展開の省略・不正出力を補正せず502にする方針も確定済み | C9/C12 |

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
