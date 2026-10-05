# 03 設定・ルーティング・静的配信・アップロード・CGI — 理解すべきこと

一次情報: nginx ドキュメント(`ngx_http_core_module` の server/location/root/index/autoindex/error_page/client_max_body_size/limit_except/return)、RFC 3875(CGI)、RFC 7578(multipart/form-data)、RFC 3986、RFC 6265(ボーナス)。

CGIの確定事項は [Requirements.md](../Requirements.md) と末尾の「C12 CGIの最小構成・決定事項と残る合意」に揃えて記録する。C8〜C10には調査項目も含まれ、列挙された機能すべてを実装するという意味ではない。

---

## C1 設定ファイル文法 【rysato, R:M, D:L, ★★★】

**理解すべきこと**
- 本チームの基本文法は確定済み(05のC-04)。ファイル直下のserverとその中のlocation、`directive arg1 arg2;`、`{}`、`#`コメント、空白・タブ・改行区切りを使う。引用符・エスケープ・変数・include・正規表現・locationの入れ子・reload・http/eventsの外枠は省く。
- **確定(C-37)**: serverを1個以上、各serverにlocationを1個以上必須とする。ブロック後の`;`は拒否し、server/locationを含む設定名は大小区別、同じ階層内の記述順は不問。特殊location(= / ^~ / ~ / ~* / 名前付き)とif/rewrite/try_filesは起動エラー。空locationは既定値で文法上受理する。実在・権限の起動時検証とは区別する。
- 字句解析(トークン種別: WORD, `{`, `}`, `;`)と構文解析の分離(決定: 1クラス、private関数で分離)。
- **確定(C-42)**: ConfigParser::parse(const std::string&)が既定値補完・パス解決・検証済みのConfigを値で返す。設定読込/解析/検証エラーはstd::runtime_errorでmainへ伝え、mainが理由を表示して起動を中止する。型はstring/int/size_t/vector/set/mapとenum、CGI設定は文字列2つの1組。getterのみ公開し、mainのconst Configをtasugiyaがconst参照/ポインタで使う。Configは参照元より長く生存させる。
- 文法エラー・未対応の書き方は理由を表示して起動を中止する(行番号なし、確定済み)。未閉じブロックや未知設定を補正・無視しない。各ディレクティブの必須性・回数・引数数・数値範囲等は05の確定事項に従う。
- client_max_body_sizeは正の十進整数・バイト単位のみ、単位接尾辞と桁あふれは拒否(C-13で確定)。保持型はstd::size_tで確定(C-42)。
- 設定ファイルのデフォルトパス `./conf/default.conf`(決定)、引数>1個や読めないファイルのエラー。

**資料化する内容**: 文法のBNF風定義、エラー種別一覧、**サンプル設定ファイル(評価デモ用に複数必要)**の設計方針。

## C2 ディレクティブ一覧と検証規則 【rysato(主)/共, R:M, D:L, ★★★】

課題§7の10項目をディレクティブに落とす。決定事項は「継承なし・省略時は原則NGINXの既定値を適用(listenのみC-39の固定値)」「NGINX風+独自名の混在」、root/aliasの区別、error_pageの直接ファイル指定、相対パスの設定ファイル基準。明示offを強制しない。独自項目の省略値はallow_methodsがGETのみ、upload_store / cgi_extensionが無効で確定済み。Configの型/APIはC-42で確定し、[Config合意資料](05_config_agreement.md)に揃える。

**表に起こす内容(各行: 名前 / 階層 / 型 / 必須か / 検証 / nginx対応名)**

| 課題要件 | 階層 | 叩き台名 | 備考 |
|---|---|---|---|
| listen interface:port | server | `listen` | 明示時はIPv4:portのみ(確定、C-10)。IPv4の各数値は0〜255、ポートは1〜65535。他の書式・追加オプション・不正値は起動エラー。省略時は起動権限によらず0.0.0.0:8000(C-39)。明示は1serverに最大1回とし、同じ値でも2回以上は起動エラー(確定、C-11)。複数ポートはserverを分ける。同一IPv4:portと同じポートでの0.0.0.0/特定IPの競合は既定値補完後に検証し起動エラー(確定、C-12)。権限判定は行わず、bind失敗は起動エラー。別ポートへの自動変更なし(C-39) |
| デフォルトエラーページ | server | `error_page` | ファイルを直接指定し、未指定・読み取り失敗時は内蔵ページ(確定)。複数指定可・同じserver内の同一コード重複は起動エラー(確定、C-14)。server内に100〜599の3桁コード1個＋ファイルパス1個(C-20)。複数コード・ステータス変更・不正値/引数数は起動エラー。HTTPの本文禁止規則を優先する。本文差し替えは400〜599のみ。100〜399の設定は受理するが応答に適用しない(確定、C-20追記) |
| ボディ最大サイズ | server | `client_max_body_size` | 省略時1 MiB。明示値は正の十進整数・バイト単位のみ(確定、C-13)。0・符号・小数・単位接尾辞・桁あふれは起動エラー。復号後のボディ全体に適用し、上限と同じサイズは受理、超過は413。明示は1serverに最大1回(確定、C-14)。保持型はstd::size_t(C-42)、詳細は05を参照 |
| 許可メソッド | location | `allow_methods` | 省略時GETのみ許可(確定)。POST・DELETEは必要なlocationで明示する。CGI用locationのGET/POST限定・DELETEへの405は維持。明示は1locationに最大1回(確定、C-14)。大文字GET/POST/DELETEを空白区切りで重複なく1個以上指定し、空・重複・小文字・カンマ区切り・その他の値は起動エラー(確定、C-17)。順序に意味はなく、GET等を自動追加しない |
| リダイレクト | location | `return`(確定、C-21) | location内に301/302＋ホスト部分を持つ固定HTTP(S)絶対URLの2引数。相対URL・他コード・不正な値/引数数は起動エラー。変数・元URI/queryの自動追加・本文指定は省く(H11) |
| ルート対応付け | location | `root` / `alias` | rootはURI全体を追加、aliasはprefix置換(確定)。両方省略時はroot=`html`を設定ファイル基準で解決。alias明示時はrootを補完しない。同じlocationへの併記は設定エラーで起動中止(確定、C-08)。aliasはディレクトリ対応に限定し、prefixと値の末尾 `/` がなければ補完せず起動エラー(確定、C-09) |
| autoindex | location | `autoindex on/off` | 小文字on/offの1引数のみ、省略時off、HTML固定(確定、C-16)。不正値・引数数は起動エラー。形式・サイズ表記・時刻表示等の追加Configは作らない |
| デフォルトファイル | location | `index` | 省略時index.html。ファイル名1個のみで、複数候補・`/`を含むパス・`.` / `..` は起動エラー(確定、C-15)。存在は起動条件にせず、要求時の未発見はautoindexへ。選ばれたindex.pyにも既存のCGI判定を適用する |
| アップロード許可/保存先 | location | `upload_store`(確定、C-18) | 小文字offまたは保存先パス1個。省略時は保存無効・保存先の自動補完なし(C-07)。絶対パスと設定ファイル基準の相対パスを受理。起動時に既存ディレクトリであることを検証し、自動作成しない。GET公開先はroot/aliasで別途指定。CGIとlocation・保存先を分ける |
| CGI | location | `cgi_extension .py /usr/bin/python3`(決定済み) | 必須部分はPythonのみ。省略時はCGI無効・拡張子とPythonのパスの自動補完なし(確定、C-07)。小文字offまたは「拡張子＋Python絶対パス」1組のみ(C-19)。拡張子は`.`＋ASCII英数字1文字以上で大小区別。不正な拡張子・引数数・相対パスは起動エラー。Pythonの通常ファイル確認とaccess(X_OK)はC-40で確定 |
| server_name(対象外) | server | `server_name` | Hostによる仮想ホスト選択とともに実装しない(確定、C-12)。指定時は設定エラー。HTTP/1.1のHost検証は維持 |

**確定(C-22)**: returnがあるlocationはリダイレクト専用。allow_methodsは併記可・省略時GETのみとし、許可メソッドを確認してからリダイレクトする。禁止された対応メソッドは405とAllowで応答する。root / alias / index / autoindex / upload_store / cgi_extensionはoffでも併記を拒否し、配信設定の既定値補完・配信パス検証は行わない。

- **確定(C-41)**: 同じlocation内の設定を、既定値補完後の有効/無効と許可メソッドで検証する。配信locationでPOSTを許可するならupload_storeまたはcgi_extensionのどちらか1つが有効であることを要求し、両方無効なら起動エラーとする。同じlocationでupload_storeとcgi_extensionが両方有効なら、POST許可の有無によらず起動エラー。CGI有効locationのallow_methodsにDELETEが含まれる場合も起動エラーとする。省略/offは無効として扱い、設定行の存在だけで有効とは判定しない。returnのリダイレクト専用locationはPOST処理先の検証対象外とし、C-22の併記禁止規則を維持する。CGIやuploadを有効にしてもPOSTは自動許可せず、GETのみでCGIを使う設定やupload有効かつPOST不許可の設定も受理する。起動後のCGI用locationへのDELETE要求は引き続き405とし、CGIファイルの削除へ進めない。CGIの拡張子判定、uploadとCGIのディスク上の保存先を分ける既決定も維持する。

**検証(validation)の確認項目**: 既定値補完後の検証、`location` パスが `/` 始まりか、同一server内の同一prefixの重複location拒否(C-14で確定)、root/aliasのディレクトリ存在(C-40)、`return` と配信ディレクティブの併記禁止(offでも拒否、C-22で確定)、補完したlistenを含む重複・競合検証(C-12で規則は確定済み)。既定値のある項目の省略自体はエラーにしない。

- **確定(C-40)**: 起動時は、配信locationのroot/aliasと有効なupload_storeについて、パス解決・既定値補完後にstatで存在とディレクトリ種別を確認し、確認失敗・非ディレクトリなら設定エラーで起動を中止する。root省略時のhtmlにも適用し、conf/default.confならconf/htmlが必要になる。ディレクトリや親ディレクトリは自動作成しない。ディレクトリの読み取り・書き込み等の権限を追加のaccess検査で事前確認せず、実際の配信・一覧取得・保存時に失敗を処理する。有効なCGI設定のPythonはstatで存在する通常ファイルであること、access(path, X_OK)の事前確認が成功することを要求し、失敗したら起動エラーとする。起動時にPythonを試験実行しない。indexの存在は起動条件にせず、error_pageの不在・読み取り不可も起動エラーにしない。これらは要求時に既存のindex/autoindex・内蔵エラーページの規則で処理する。個々のHTML・画像・CGIスクリプトを起動時に全件検査しない。リダイレクト専用locationはC-22に従い配信設定を補完・検証しない。起動時検査は実行時の成功を保証するものではなく、ファイル消失・権限変更やexecve等の失敗は引き続き要求時に処理する。symlinkの運用前提と検出機能を省くC-26は維持する。

## C3 location マッチングとパス変換 【rysato, R:M, D:M, ★★★】

- **確定(C-23)**: queryを除いたURLパスに対する文字列前方一致とし、一致する最も長いprefixを選ぶ。設定順には依存せず、パス要素の境界は一致条件にしない。`/a` は `/a`・`/a/file.txt`・`/abc` に一致する。ディレクトリ配下を指定するには `/a/` と書き、これは `/a/file.txt` には一致するが `/a`・`/abc` には一致しない。`/`・`/a`・`/a/b/` があれば、`/a/b/file.txt` には `/a/b/`、`/abc` には `/a` を選ぶ。URIの分離・デコード・処理順序はC-24で確定。正規化はC-25で確定。末尾スラッシュ補完リダイレクトはC-27に従う。
- **確定(C-38)**: Configのlocation prefixは先頭`/`を必須とし、ASCII英数字(A-Z / a-z / 0-9)と `/ - . _ ~` だけを受理する。連続する`//`、パス要素全体が`.`または`..`の指定は設定エラーで起動を中止する。queryや%エンコード、非ASCII文字など許可外の文字を受け付けず、設定値をデコード・正規化して救済しない。`/`・`/images/`・`/cgi-bin/`・`/v1.0/`・`/.hidden/`・`/file..txt`は受理する。prefixの比較はOSによらず大小文字を区別し、`/Images/`と`/images/`は別のprefixとする。`location /`は推奨するが必須ではなく、一致するlocationがなければ404。各serverにlocationを1個以上要求するC-37は維持する。alias使用時のprefix末尾`/`必須(C-09)も維持し、root使用時は末尾`/`を必須にしない。この文字制限はConfigのprefixだけに適用し、HTTP要求URLやファイル名の受付範囲には一律適用しない。要求側はC-24・C-25のデコード・正規化後にC-23の最長前方一致で照合する。
- URL→ファイルパス変換はrootでURI全体を追加、aliasでprefixを置換する(確定)。URL正規化はC-25に従う。symlinkの運用前提と制限はC-26に従う。CGIの結合・検証はCGI-84〜90で確定済み。通常配信等の残る詳細はH2と合わせて決める。
- **確定(C-24)**: 最初のリテラルの `?` でパスとqueryを分離し、パスの `%XX` を1回だけデコードする。その後にパスを正規化してからlocationを選ぶ。queryは未デコードのままCGIのQUERY_STRINGへ渡す。パスでは `%20` は空白、`%2F` / `%2f` は区切りの `/` となり、`+` は空白へ変換しない。`%252e` は `%2e` までとし、HTTP/Router/CGIへの受け渡しで二重デコードしない。復号して生じた `?` をqueryの区切りとして再解釈しない。パスにある `%` / `%2` / `%GG` 等の不正なパーセント表記と `%00` (NUL) は400とする。この400規則はパスについての合意であり、queryの受付検証を新たに定めるものではない。`.` / `..` / 連続スラッシュの正規化はC-25に従い、symlinkの運用前提と制限はC-26に従う。その他の不正パスとファイルパスの結合・検証の詳細は別途確定する。
- **確定(C-25)**: C-24で1回デコードしたURLパスをlocation選択前に正規化する。連続する `/` は1個にまとめ、`/` で区切った要素全体が `.` なら除去、`..` なら直前の要素を1個取り除く。URLの先頭の `/` より上へ戻ろうとした時点で400とする。`file..txt`・`.hidden` 等は変更しない。末尾 `/` と、末尾 `/.`・`/..` が表すディレクトリの末尾 `/` を保つ。例: `/a//b.txt`・`/a/./b.txt`・`/a/x/../b.txt` は `/a/b.txt`、`/a/../b.txt` は `/b.txt`、`/a/.`・`/a/b/..` は `/a/`、`/a/..` は `/`。`/../secret.txt`・`/a/../../secret.txt` は400。`/a/%2e%2e/b.txt` はデコード後に `/b.txt` としてlocationを選ぶ。この処理はURL文字列の整理であり、symlinkの扱いと公開範囲の保証の制限はC-26に従う。ファイルパスの結合・検証の残る詳細は別途確定する。末尾 `/` のない実ディレクトリへのGETはC-27に従う。
- **確定(C-15)**: 存在するディレクトリへのGET要求はindexを探し、なければautoindexがonなら一覧、offなら403。ディレクトリ自体がなければ404。indexの読み取り不可は一覧表示への切り替え条件にしない。末尾`/`なしの実ディレクトリへのGETはC-27の301を先に返す。この判断フローがRouterの中核仕様。
- **確定(C-27)**: GETでlocationを選択し許可メソッドを確認した後、選択した配信locationのroot/aliasで解決した対象が実ディレクトリで、正規化後のURLパスが `/` で終わらなければ、末尾 `/` 付きURLへ301を返す。queryは保持し、例えば `/docs?lang=ja` の移動先のパスとqueryは `/docs/?lang=ja` とする。末尾 `/` 付きの要求でC-15のindex→未発見ならautoindexの処理へ進む。POST・DELETEにはこの自動補完を適用せず、各メソッドの処理規則に従う。returnがあるlocationはC-21・C-22の指定を使う。location選択は変更せず、`location /docs/` を `/docs` に一致させる特別扱いは追加しない。Locationのホスト名・ポートの出所はC-28に従う。パスの再エンコードとqueryの保持はC-29に従う。
- **確定(C-28)**: C-27の自動補完のLocationは `http://` で始まる絶対URLとする。有効なHostがあればホスト名と明示ポートを使い、HostにポートがなければURLにも追加しない(HTTPの既定80番)。HTTP/1.0でHostがない場合だけ、acceptした接続のサーバー側IP・ポートをgetsocknameで取得して使う。listenのワイルドカード値0.0.0.0やクライアント側IP・ポートを代用しない。HTTP/1.0・1.1ともHostの重複・不正は400、HTTP/1.1のHost欠落も400。tasugiyaが接続のサーバー側IP・ポートを取得して渡し、rysatoがHostの検証とURL組み立てを担当する。Forwarded / X-Forwarded-*を参照せず、ConfigのreturnはC-21の固定URLを使う。パスの再エンコードはC-29に従う。Hostの受付範囲はC-30に従う。接続情報の受け渡しAPI・取得失敗時の扱いは別途確定する。
- **確定(C-30)**: Host値は前後のSP/HTABを取り除いてから、IPv4またはASCIIホスト名と任意の `:port` として検証する。ホスト名はASCII英数字と `-` からなる空でない要素を `.` で区切り、各要素の先頭・末尾は英数字とする。localhostのような単一要素、大文字・小文字、末尾の `.` 1個を受理する。IPv4はドット区切りの4個の十進数で各0〜255。ポートは省略可、明示するなら十進数の1〜65535で空値は拒否する。Host全体の空値、途中の空白・タブ、重複、不正な値、スキーム・パス・ユーザー情報付きの値は400。IPv6の角括弧表記とHost内のパーセント表記は受付対象外として400にする。DNSへの問い合わせや名前の実在確認は行わず、Hostによる仮想ホスト選択も行わない。HTTP/1.0のHost省略可、HTTP/1.1の欠落400、自動補完URLでの使用方法はC-28を維持する。この受付範囲は本課題用の限定仕様であり、RFCのHost構文全体を実装するものではない。
- **確定(C-29)**: C-27の自動補完のLocationを生成するときは、デコード・正規化済みのパスに末尾 `/` を付け、ASCII英数字(A-Z / a-z / 0-9)・`- . _ ~` と区切りの `/` だけをそのまま出力する。それ以外はバイトごとに大文字16進の `%XX` へ変換する。例: 内部パス `/hello world` → `/hello%20world/`、`/a+b` → `/a%2Bb/`、`/name?part` → `/name%3Fpart/`、`/literal%20` → `/literal%2520/`。URLとして既にエンコード済みの文字列へ重ねて適用せず、内部のデコード済みパスから一度生成する。queryは元の表記を保持して後ろに付け、デコードも再エンコードも行わない。例えばHost localhost:8080への `/hello%20world?name=a+b` は `http://localhost:8080/hello%20world/?name=a+b` へ案内する。Configのreturnの固定URLはこの変換の対象にしない。

## C4 静的ファイル配信 【rysato, R:M, D:M, ★★★】

- `stat`/`access` による存在・種別(通常ファイル/ディレクトリ)・読み取り権限の判定と、**結果→ステータス対応**(なし=404、権限なし=403、ディレクトリ→C3)。
- `Content-Type`(H10)、`Content-Length`、`Last-Modified`(任意)。大きなファイルの扱い(N4/N10)。
- 条件付きリクエスト(`If-Modified-Since`、304)、Range(206)は**課題必須ではない**。スコープ外として明文化するかを決める(ブラウザ動画再生などでRangeが使われる点に注意)。
- **確定(C-26)**: 公開・CGI・アップロード領域は二人が管理し、配下にsymlinkを置かない運用前提とする。Webservにはsymlinkの検出・拒否機能を実装せず、関連するConfig項目も増やさない。symlinkが置かれた場合はOSがリンク先を辿り得るため、公開領域外へのアクセスを完全に遮断する仕様とはしない。URL由来のパスにはC-24・C-25のデコード・正規化を適用し、その処理だけでsymlink経由の領域外アクセスを防げるとは扱わない。CGIのファイル存在・種別・アクセス失敗とroot/aliasのパス結合はCGI-84〜90で確定済み。通常配信・アップロード名等の残る実装規則は別途確定する。
- 隠しファイルの配信・一覧表示は未確定。参考: NGINXは既定でsymlinkを検査せず許可する。[NGINX disable_symlinks](https://nginx.org/en/docs/http/ngx_http_core_module.html#disable_symlinks)。課題PDF印刷ページ6にはstatはあるがlstat / realpath / openat / fstatatは記載されていない。statはリンク先を参照する。[stat / lstat](https://man7.org/linux/man-pages/man2/stat.2.html)。

## C5 autoindex 【rysato, R:S, D:S, ★★】

- **確定(C-16)**: 設定は小文字on/offのみ・HTML出力固定。indexがない場合に一覧を返し、offでも個別ファイルへのアクセスは禁止しない。以下の並び順・表示項目等は残る実装仕様として決める。

- `opendir/readdir/closedir`、`stat` で種別・サイズ・更新時刻。`.` と `..` の扱い、ソート順(ディレクトリ先頭など)。
- **HTMLエスケープ**(ファイル名に `<>&"` が含まれる場合のXSS対策)と、URLエンコード(リンクのhref)。
- 結果の生成方法(HTML文字列組み立て)。巨大ディレクトリ時のサイズ上限。

## C6 アップロード 【rysato, R:L, D:L, ★★★】

**理解すべきこと**
- アップロード方式の選択肢: (a) `multipart/form-data`(ブラウザのフォーム標準)、(b) 生のボディ(`curl --data-binary`、`PUT`的な)。**課題は「クライアントがファイルをアップロードできる」とのみ規定**→評価でブラウザ/curlどちらを使うか不明のため、両方の対応範囲を決める(推奨: multipart対応+生ボディ対応の可否を判断)。
- **multipart/form-data の構造**(RFC 7578 / RFC 2046 §5.1): `Content-Type: multipart/form-data; boundary=XXXX`、各パートは `--boundary CRLF part-headers CRLF CRLF data CRLF`、終端 `--boundary--`。パートヘッダー `Content-Disposition: form-data; name="..."; filename="..."`、`Content-Type`。
  - boundary のクォート有無、境界文字列の検索がバイナリ安全(`\0`を含むデータ)であること(`std::string`で可だが `c_str()` 依存を避ける)。
  - 複数ファイル/ファイル以外のフィールドの扱い。
- 保存先と**ファイル名の扱い**: クライアント指定の `filename` を信用しない(`../` 等のトラバーサル、特殊文字、既存ファイル上書き、拡張子による実行(CGI拡張子のアップロード=リモートコード実行))。サニタイズ規則・衝突時の命名(連番/タイムスタンプ)を決める。
- ボディ全体をメモリに保持するか(`HttpRequest::getBody()` は std::string 返却=全メモリ保持の設計)。上限は `client_max_body_size` で制御。大きいファイルでのメモリ・性能の許容範囲を明文化。
- 応答: 201 + `Location`(保存先URL)、または 200 + 結果HTML。失敗時: 413(上限超過)、400(multipart不正)、403/500(書込失敗)、415(未対応のメディア型)。
- **確定(C-18)**: 保存先は起動時に存在・ディレクトリ種別を検証し、自動作成しない。ディレクトリへの追加の権限事前検査は行わない(C-40)。実際の保存失敗の扱いは別途確定する。

## C7 DELETE 【rysato, R:S, D:S, ★★】

- 対象解決は静的配信と同じ(C3)。ファイルのみ許可か、ディレクトリ削除を許すか(許す場合 `rmdir` 相当は許可関数に**無い**: `unlink`/`remove`/`rmdir` は許可リストに載っていない → **DELETEの実装手段を確認**。許可関数に削除系がない点は要確認事項)。
- 結果: 成功 204(または200)、404、403、405。
- 削除対象を限定するか(例: アップロード先ディレクトリ配下のみ)—セキュリティ上、全ファイル削除可能は危険なので `allow_methods` と location 設計で制御する旨を明記。

> **確認事項**: 許可関数リストに `unlink`/`remove` が含まれていない。DELETEでファイルを実際に削除する方法を課題/運営に確認する(`Requirements.md` を再確認)。解決しないとC7の仕様が書けない。

## C8 CGIプロセス制御(システムコール面) 【tasugiya(主)/共, R:L, D:L, ★★★】

**理解すべきこと**
- pipe×2→fork→子のdup2・不要端close・chdir・execveで起動する。**_exitとclock_gettimeはユーザー確認により使用不可(CGI-119)**。子の失敗時の終了と計時は9/10の現時点の採用案で実装し、実際の動作確認で見直し得るものとして扱う。
- 親: 使わない端の close、**パイプfdをノンブロッキング化し poll に登録**(CGI出力の読み取り、ボディの書き込み。課題の「pipeは必ずpoll経由」要件)。
- **fork は CGI 実行にのみ使用可**。ここ以外で `fork` しない。
- `fork` 後に子で C++ オブジェクトが多数存在する状態 → 子は `execve` 直前までに**複雑な処理をしない**(メモリ確保など)、`envp` は fork 前に構築しておく(`char**` の組み立て・解放責任の明確化)。
- 子プロセス回収: `waitpid`(`WNOHANG`)、ゾンビ防止。タイムアウト時の `kill(pid, SIGKILL)` → `waitpid`。
- `execve` 失敗の検知方法(子の終了方法と親への通知方法を確定する。応答は502で確定、C12)。
- インタプリタ/スクリプトの検証: インタプリタは存在・実行権限、Pythonへ引数として渡すスクリプトは通常ファイルであること・読み取り権限を確認する。スクリプト自身の実行権限は要求しない。
- fd継承はCGI-116で確定済み。listen/acceptソケットと全CGIパイプにF_SETFD/FD_CLOEXECを設定し、子の0/1/2だけを継承する。

## C9 CGI仕様(RFC 3875) 【rysato(主)/共, R:L, D:L, ★★★】

**理解すべきこと**
- **メタ変数**(RFC 3875 §4.1)。決定事項に挙がった `REQUEST_METHOD, SCRIPT_NAME, PATH_INFO, QUERY_STRING, CONTENT_LENGTH, CONTENT_TYPE, SERVER_PROTOCOL, GATEWAY_INTERFACE, SERVER_NAME, SERVER_PORT, REMOTE_ADDR` + `HTTP_*`。それぞれの**値の作り方**:
  - `SCRIPT_NAME`/`PATH_INFO` の分離(`/cgi/test.py/extra/path` → script=`/cgi/test.py`、PATH_INFO=`/extra/path`)はCGI-25、各環境変数の規則はCGI-29・CGI-30で確定済み。
  - 必須部分はPythonのみとし、PHP固有の環境変数は追加しない(確定、C12)。基本的なPATH_INFO対応は確定済み。CGI-50の環境変数の限定に従い、`PATH_TRANSLATED`は追加しない。
  - `CONTENT_LENGTH` はCGI-31で確定済み。復号後の本文が1バイト以上ならそのバイト数、0バイトなら空文字列とする。`CONTENT_TYPE` は要求ヘッダーの値をパラメーターも含めて渡し、ヘッダーなしの場合は省略する(CGI-32)。`QUERY_STRING` は未デコードのまま(CGI-28)。
  - `HTTP_*` 変換はCGI-39で確定済み: ヘッダー名をASCII大文字化、`-`→`_`、接頭辞`HTTP_`。値は前後SP/HTAB除去後に大小文字を保持する。`Content-Type`/`Content-Length` は専用変数だけで渡し、HTTP_*から除外する(CGI-40)。`Proxy` ヘッダーはCGI-45で除外を確定済み。名前の文字範囲はCGI-47で確定済み。転送対象の同名ヘッダーの重複拒否はCGI-48、HTTP_*の空値省略はCGI-49で確定済み。入力ヘッダーの残る規則はCGI-75〜83で確定済み。受け渡しAPIはC12で確定する。
- **リクエストボディの渡し方**: un-chunk 済みのボディを子のstdinへ全て書き、書き終えたらstdinを **close(EOF)**(課題要件)。ボディが大きいと pipe バッファ(64KB程度)を超える → **書き込みもノンブロッキング+POLLOUT**(死活: 子が出力を溜めて読まないとデッドロックし得る → 読み/書きを同時にpoll)。
- **CGI出力のパース**(§6): `CGI-response = document-response / local-redir-response / client-redir-response / client-redirdoc-response`。ヘッダー部(`Content-Type`, `Status`, `Location`, 任意ヘッダー)+空行+ボディ。
  - `Status: 404 Not Found` → HTTPのステータス行へ変換。通常の文書応答で省略された場合は200。リダイレクトは別の規則で扱う。
  - `Location:` のみ(ローカル絶対パス `/other`)はRFC上はサーバー内部リダイレクトだが、今回の実装対象外とする(確定、C12)。絶対URLのLocationのみは302として扱う(確定)。
  - 通常の文書応答で `Content-Type` がない場合は不正なCGI出力として502。Locationのみのリダイレクトとは区別する。
  - **ヘッダー区切りが LF/CRLF 両方**あり得る(Pythonの`print`はLF)。
  - **`Content-Length` が無ければ EOF を終端とする**(課題要件)。サーバー側は終端後に自分で `Content-Length` を付けて返す(接続は `Connection: close`)。
- 終了ステータス(非0)はCGI-19のエラー対応に従う。stderrはCGI-51に従いWebservのstderrを引き継ぎ、stdoutへ混ぜず専用の監視・収集処理も追加しない。

## C10 CGIとイベントループの統合 【共, R:M, D:M, ★★★】

- CGI状態を独立した `CgiProcess` に持たせ、EventLoopが **fd→オブジェクトの対応表**で引くことは決定済み。`Client::WAITING_CGI` は状態設計案。Clientとの関連付けやポインタの寿命管理はC12で確定する。
- 複数のfd(CGIのstdin書込/stdout読取)を毎周のpollfd再構築で扱う方法。
- **失敗パターンの網羅**: execve失敗、CGIがすぐ終了、出力なし、出力途中でクライアント切断、タイムアウト(10秒→kill→504)、巨大出力(上限)、子が書込を詰まらせる、複数同時実行。アプリケーション独自のCGI同時実行数上限は設けず、pipe/fork失敗は500として後始末する(C12)。
- CGI結果のHttpResponseへの変換はrysatoのRouter::parseCgiOutput、即時応答/CGI要求の区別はRouter::routeが担当する(CGI-91〜98確定)。所有権・寿命はCGI-99〜106で確定済み。後始末はCGI-107〜118で確定済み。起動失敗検出等はC12の9/10以降で合意する。

## C11 Cookie/セッション(ボーナス) 【rysato, R:M, D:M, ★】

- `Set-Cookie`/`Cookie` の形式(RFC 6265)、属性(`Path`, `Expires`/`Max-Age`, `HttpOnly`, `SameSite`)。セッションIDの生成(乱数源: 許可関数に `rand` 系があるか要確認)、サーバー側セッション保管(メモリ上のmap)。
- ボーナスは**必須部分が完璧な場合のみ評価**されるため、優先度は最低。複数CGIタイプ対応(PHP等)も同様。

## C12 CGIの最小構成・決定事項と残る合意 【共】

記録日: 2026-10-04。CGI-25〜43追記: 2026-10-05。CGI-44〜83追記: 2026-10-06。課題PDF Version 24.1と参考資料を踏まえ、必須部分の実装量を抑えるための合意状況を記録する。

**CGIの10項目は合意完了(CGI-01〜135)。** 起動・計時のCGI-120〜127は現時点の代替案で、実装・実際の動作で見直し得る。合意完了は実装・結合試験合格を意味しない。確定事項は [Requirements.md](../Requirements.md) のCGI節と同じID・内容で記録する。変更時は関連するHTTP・I/O・Config・テスト計画も合わせて更新する。

### 決めたこと：確定事項

CGI-120〜127は**現時点で採用する代替案**。実装・実行環境・動作確認の結果で見直し得るもので、動作確認済み・最終固定の仕様とは扱わない。詳細は03の残項目9/10に従う。子の起動失敗時にWebservが所有するメモリを明示的に解放し、親用の停止・回収処理を実行しない方針は合意済み。具体的な終了・解放経路は代替案として実装・動作確認で検証する。

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
| CGI-39 | HTTP_*の変換 | 転送対象の要求ヘッダー名をASCII大文字に変換し、-を_へ置き換え、先頭にHTTP_を付けて環境変数名にする。値はHTTPパーサーで前後のSP/HTABを除去したものを使い、大小文字を保持する。転送対象・除外・重複・変換後の名前の衝突は別途確定する |
| CGI-40 | 専用変数と重なるヘッダーの除外 | Content-TypeとContent-LengthはCGI-31・32の専用変数だけで渡す。HTTP_CONTENT_TYPEとHTTP_CONTENT_LENGTHは設定しない。専用変数の値と設定・省略の規則は既存の合意を維持する |
| CGI-41 | Transfer-Encodingの除外 | 要求のTransfer-EncodingはHTTP_*へ転送せず、HTTP_TRANSFER_ENCODINGを設定しない。CGIには復号後の本文を渡し、本文長とstdinのEOFは合意済みの規則に従う |
| CGI-42 | Connectionの除外 | 要求のConnectionヘッダー自体はHTTP_*へ転送せず、HTTP_CONNECTIONを設定しない。要求の値によらず、1接続1要求・応答後に切断する既存方針を維持する |
| CGI-43 | その他の通信制御用ヘッダーの除外 | Keep-Alive・TE・Upgrade・Proxy-ConnectionはHTTP_*へ転送せず、HTTP_KEEP_ALIVE・HTTP_TE・HTTP_UPGRADE・HTTP_PROXY_CONNECTIONを設定しない。ヘッダー名による除外とし、この合意によって各ヘッダーの機能を新たに実装するものではない |
| CGI-44 | Connectionに列挙されたヘッダーの除外 | Connectionの値に列挙されたヘッダー名もHTTP_*への転送対象から除外する。名前はカンマで区切り、前後のSP/HTABを除去して大小文字を区別せず照合する。この除外で本文の受信処理や専用環境変数の規則は変更しない |
| CGI-45 | Proxyの除外 | 要求のProxyヘッダーはCGIへ転送せず、HTTP_PROXYを設定しない。Proxyヘッダーがあることだけを理由に要求を拒否する処理は追加しない |
| CGI-46 | 認証用ヘッダーの除外 | Authorization・Proxy-AuthorizationはCGIへ転送せず、HTTP_AUTHORIZATION・HTTP_PROXY_AUTHORIZATIONを設定しない。これらの存在だけを理由に要求を拒否する処理や、認証情報を解釈する処理は追加しない |
| CGI-47 | HTTP_*へ転送する名前の文字範囲 | HTTP_*へ転送するヘッダー名はASCII英数字と-だけで構成されるものに限定し、合意済みの除外規則を適用する。_やその他の記号を含む名前はCGIへの転送だけを省略し、HTTP要求としての受付は既存のtoken検証を維持する。同名ヘッダーの重複は別途確定する |
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
| CGI-89 | ファイル検査後の変化 | 検査後のファイル消失・権限変更等でも再検査・再起動によるリトライは追加しない。起動処理の失敗・Pythonの異常終了は既存のCGIエラー規則に従う。子のchdir等の失敗検出方法は9/10で確定する |
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
| CGI-119 | 使用不可の関数 | ユーザー確認により_exitとclock_gettimeはともに使用不可とする。子の終了・時間計測の方法はこの制約の下で残項目9/10として合意する |
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

HTTP要求のリクエスト行8 KiB・ヘッダー合計32 KiB(C-32、超過414/431)と、CGI出力ヘッダー8 KiB・stdout全体8 MiB(CGI-20、超過502)は別の上限である。入力ボディは既存のclient_max_body_sizeで制限する。

HTTPリクエスト行・HTTPヘッダーの改行はCRLF限定(C-31)。CGI-15のLF受理はCGI出力に限る。HTTPボディの改行はデータとして保持する。HTTP要求ヘッダーの折り返し拒否と前後SP/HTAB除去(C-33)もボディには適用せず、CGI出力の検証規則とは区別する。HTTP要求ヘッダー名のtoken検証・ASCII小文字保持(C-34)もCGI出力の受付やHTTP_*への変換規則を新たに確定するものではない。
要求のContent-Lengthは同値でも重複・カンマ入りを400にする(C-35)。これはCGI出力のContent-Length検証を確定するものではなく、CGI出力側の数値・桁あふれ・重複はCGI-67・69で確定済み。
長さ指定が両方ないPOSTも空ボディで受信完了する(C-36)。CGIへ進む場合はstdinに書き込まず、親側の書き込みパイプを閉じてEOFを渡す。CONTENT_LENGTHはCGI-31に従い空文字列で設定する。環境変数の細則はC12のCGI-27〜50・75〜83に従う。

CGI-03の「加工せず」は、HTTPのchunked復号後のボディをそのまま渡すという意味。CGIへの受け渡しのためにフォームの `name=value` 分解、URLデコード、multipartの分解をWebserv側で行わない。通常のアップロード機能(C6)の形式・保存処理は別途決める。

CGI-01は対応言語をPythonに限定する合意。Configで受理する拡張子の文法と1組限定はC-19、拡張子とインタプリタの文字列2つによる保持はC-42で確定済み。[Config合意事項](05_config_agreement.md)を参照する。

CGI-23では `/cgi/test.py?a+b` 自体を拒否せず、QUERY_STRINGを `a+b` とする。RFC 3875 §4.4にあるindexed queryの引数展開を省く設計判断であり、課題PDFにこの展開機能が個別に指定されているわけではない。CGI-24も出力の検査を省く意味ではない。課題PDFは不正なCGI出力の補正を明記しておらず、最小構成では不正と判定した時点で502にする。RFC §6.3が重複を禁じるContent-Type・Location・Statusと、その他の応答ヘッダーは区別して検証規則を決める。

### 環境変数の合意済みの規則

下記11変数の値と設定・省略の規則は合意済み。CGI用環境はCGI-50に従い、これらと転送対象のHTTP_*だけから構築し、親の環境変数はコピーしない。SERVER_SOFTWAREはCGI-38で対象外とし、CGIへ渡す環境変数に含めない。HTTP_*の名前・値の変換はCGI-39、Content-Type・Content-LengthのHTTP_*からの除外はCGI-40、Transfer-Encodingの除外はCGI-41、Connection自体の除外はCGI-42、その他4つの通信制御用ヘッダーの除外はCGI-43、Connectionに列挙されたヘッダーの除外はCGI-44、Proxyの除外はCGI-45、認証用ヘッダーの除外はCGI-46、転送する名前の文字範囲はCGI-47、転送対象の同名ヘッダーの重複拒否はCGI-48、HTTP_*の空値省略はCGI-49で確定済み。入力ヘッダーの残る規則はCGI-75〜83で確定済み。環境変数の受け渡しAPIは引き続き別途確定する。

| 変数 | 値の出所 | 空値/省略 | 合意ID |
|---|---|---|---|
| REQUEST_METHOD | 解析済みの要求メソッド。GET要求なら`GET`、POST要求なら`POST` | 必ず設定する。空文字列・省略にはしない | CGI-27 |
| QUERY_STRING | 最初の`?`より後ろの未デコードのクエリ。`?`自体は含めず、`+`や`%20`をそのまま保持する | 必ず設定する。クエリなし・末尾が`?`だけの場合は空文字列とし、省略しない | CGI-28 |
| PATH_INFO | C-24・C-25のデコード・正規化後のURLパスからCGI-25で切り分けた後続部分。スクリプト直後の`/`だけが残る場合は`/` | 必ず設定する。追加パスがなければ空文字列とし、省略しない | CGI-29 |
| SCRIPT_NAME | デコード・正規化後のスクリプトを示すURLパス。PATH_INFO・クエリ・ディスク上のパスは含めない。index経由ではURL上のディレクトリパスに選ばれたindexのファイル名を付ける | 必ず設定する。この対応範囲ではスクリプトを示す先頭`/`のパスとなり、空文字列・省略にはしない | CGI-30 |
| CONTENT_LENGTH | CGIへ渡す復号後の本文の実際のバイト数。1バイト以上なら10進数の文字列 | 必ず設定する。0バイトなら空文字列とし、省略しない。長さ指定なし・Content-Length: 0・空のchunked本文も同じ扱い | CGI-31 |
| CONTENT_TYPE | 要求のContent-Typeヘッダーの値。C-33による前後SP/HTAB除去後の値を、boundary等のパラメーターも含めて渡す | ヘッダーがあれば本文0バイトでも設定する。ヘッダーがなければ省略し、推測・既定値の補完はしない | CGI-32 |
| SERVER_PROTOCOL | 解析済みの要求バージョン。HTTP/1.0要求なら`HTTP/1.0`、HTTP/1.1要求なら`HTTP/1.1`。応答側の固定値で上書きしない | 必ず設定する。空文字列・省略にはしない | CGI-33 |
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

[RFC 3875 §4.1.8](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.8)は要求を送ったクライアントのネットワークアドレスの設定を必須としている。チームではIPv4の接続元を使い、ローカル接続なら`127.0.0.1`等を渡す。値は接続情報から取得し、X-Forwarded-For等を参照しない。この決定はREMOTE_ADDRの値の出所と形式についてであり、HTTP_*として渡すヘッダーの範囲を新たに確定するものではない。アドレスの文字列化手段と受け渡しAPIは、課題の許可関数に従って別途確定する。

[RFC 3875 §4.1.14](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.14)は要求先のホスト名またはネットワークアドレスを必ず設定することを要求している。チームでは`Host: localhost:8080`なら`localhost`、`Host: example.com`なら`example.com`を渡す。HTTP/1.0でHostがない場合の接続情報は、末尾/補完のリダイレクトで確定したC-28と共通にできる。listenのワイルドカード値`0.0.0.0`ではなく、接続先の実際のサーバー側IPを使う。Hostの受付検証はC-28・C-30に従い、HTTP/1.1のHost欠落は従来どおり400とする。DNS照会・server_name設定・Hostによる仮想ホスト選択は追加しない。接続情報の取得失敗時の扱いと受け渡しAPIは引き続き別途確定する。

**確定(CGI-38)**: 最小構成ではSERVER_SOFTWAREを省略し、固定のサーバー名・バージョンをCGIへ渡す処理は実装しない。課題PDFはこの変数を個別に列挙していない。[RFC 3875 §4.1.17](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.17)では必須だが、チームではRFC全体への準拠を目標とせず、この項目を対応範囲から外す。

**確定(CGI-39)**: 転送対象の要求ヘッダー名をASCII大文字に変換し、`-`を`_`に置き換えて先頭に`HTTP_`を付ける。`User-Agent: curl/8.0`なら`HTTP_USER_AGENT=curl/8.0`、`X-Test: AbC`なら`HTTP_X_TEST=AbC`となる。値はC-33の前後SP/HTAB除去後のものを使い、大小文字を保持する。[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)がこの名前の変換規則を定めている。この合意だけで全要求ヘッダーの転送を決めるものではなく、転送対象・除外・重複・異なる名前が同じ環境変数名になる場合の扱いは引き続き別途確定する。

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

**確定(CGI-50)**: `execve`へ渡す`envp`は、合意済みの11種類の環境変数と転送対象のHTTP_*だけから構築する。CONTENT_TYPE等の省略規則も維持し、常に11個すべてを設定する意味ではない。親プロセスの環境をコピーしないため、起動元のPATH・HOME・PYTHONPATH・HTTP_PROXY等は渡さず、親にREQUEST_METHOD等があっても要求から生成した値を使う。PythonとスクリプトはCGI-26の絶対パスで指定するため、起動対象の検索用PATHは追加しない。環境変数を追加・上書きするConfig項目も設けない。[Linux execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)では、envpで新しいプログラムに渡す環境を指定できる。課題PDFは要求情報をCGIから利用できることを求めるが、親の環境の継承方法は指定しておらず、本決定は最小構成のためのチーム方針である。envpの組み立て・寿命・所有権と担当境界は別途確定する。

**確定(CGI-75〜83、残項目4/10)**: 要求ヘッダーのCGIへの受け渡しを次の規則で確定する。

- CGI-39〜49の名前・除外・重複・空値の規則を満たす要求ヘッダーを転送し、Accept等の個別の許可一覧は作らない。例えばHostはHTTP_HOSTへポート付きの値を渡し、SERVER_NAMEのホスト部分のみの規則とは区別する。
- Trailer・ExpectもHTTP_*から除外する。本文後のtrailerはHTTP層で読み取り・構文検査し、環境変数に追加しない。初期ヘッダーを上書きしない。HTTP/1.1のExpectは既存の417でCGIを起動せず、HTTP/1.0では値によらず無視する。
- 要求Connectionの値は、各行をカンマで区切り、要素の前後SP/HTABを除去してヘッダー名のtokenとして検証する。不正な非空要素は400、空要素は無視する。複数行の場合は全行の名前を除外対象に加える。CGI-44の専用変数・本文受信を変更しない方針を維持する。
- 要求ヘッダーの共通の値の文字検査として、HTAB以外の制御文字(0x00〜0x1F)とDEL(0x7F)を400にする。0x80〜0xFFは変換せず保持する。転送から除外するヘッダーもこの共通検査を受ける。ボディには適用しない。
- CGIへの要求のContent-Typeは最大1行とし、同値・空値を含む重複は400とする。単一の値は前後SP/HTAB除去後にCONTENT_TYPEへ渡す。空ならCONTENT_TYPEを空文字列で設定し、ヘッダーなしなら省略する。本文の形式やパラメーターの解釈はCGIへ任せ、入力のメディアタイプの構文・登録一覧の照合をCGIへの受け渡しのために追加しない。通常のアップロード側の検証は別の既存仕様に従う。
- 転送除外対象の重複を一律に拒否する分岐は追加しない。Host・Content-Length等の個別検証、Connectionの全行処理とCGI要求のContent-Typeの重複拒否は維持する。
- 要求のContent-Encodingは転送規則を満たせばHTTP_CONTENT_ENCODINGへ渡す。Webservがgzip等を展開する機能は追加せず、chunked等の転送符号化の復号だけを済ませた本文を渡す。
- CGI環境変数に渡す要求情報に実NULバイトが残れば起動前に400とし、C文字列として値が途中で切れることを防ぐ。クエリの文字列%00は未デコードのまま渡せる。URLパスの%00を400とするC-24と、本文を加工しないCGI-03は維持する。

転送範囲は[RFC 3875 §4.1.18](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.1.18)、内容符号化を展開しない方針は[§4.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-4.2)、値の文字検査は[RFC 9110 §5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.5)、Connectionの空要素は[§5.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.1)を参考にする。課題PDFの個別指定ではなく、Trailer・Expectの除外、trailerを環境変数に追加しないこと、CGI要求のContent-Typeの重複拒否等は最小構成のためのチーム方針である。

### CGIのファイル・パス検証の合意済みの動作

**確定(CGI-84〜90、残項目5/10)**: 実行するスクリプトまでを検証し、PATH_INFOに対応するファイルを探す処理は追加しない。

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

**確定(CGI-56〜61、残項目1/10)**: 出力の基本構文・値・Content-Type・Statusを次の規則で検証する。

- 各ヘッダー行を最初のコロンで分ける。コロンがない行や、ヘッダー終端の空行がないままEOFになった出力は502とする。行ごとにLF/CRLFを受理し、混在も許容する。単独CRは拒否する。ヘッダー終端後の本文にはこれらの判定を適用しない。
- 値の前後SP/HTABだけを除去し、内部の空白・タブと大小文字を保持する。HTAB以外の制御文字(0x00〜0x1F)とDEL(0x7F)は502とする。0x80〜0xFFは変換せず保持し、ヘッダーごとの追加の構文検証を行う。空値は省略と同じ扱いとし、重複検査は空値の省略より先に行う。
- Content-Typeのtype/subtypeは双方が空でないtokenとし、付いているパラメーターは名前と値の構文を検証する。値はtokenまたはquoted-stringとし、構文上許されるSP/HTAB、引用・エスケープも扱う。未知の種類やパラメーターも構文が正しければ受理し、登録一覧の照合や本文からの形式推測・文字コード変換は行わない。通常の文書応答で空のContent-Typeは必須ヘッダー欠落として502とする。
- StatusはASCII数字3桁と直後の区切りSP、説明文の形式を検証する。区切りをHTABで代用しない。説明文は空でもよいため、末尾空白除去前の値で区切りSPを確認し、その後に説明文の前後空白を除去する。`Status: 200 `は構文を満たし、`Status: 200`は区切りSP欠落として502となる。コードはユーザーの指定に従い100〜599を受理し、099・600・4桁等は502とする。説明文は検証後にHTTPステータス行へ使う。通常の文書応答でStatus省略・空値ならCGI-14の200を維持する。

構文・空値・Content-Type・Statusは[RFC 3875 §6.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3)、値の文字の扱いは[RFC 9110 §5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.5)、コード範囲は[§15](https://www.rfc-editor.org/rfc/rfc9110.html#section-15)を参考にする。課題PDFの個別指定ではなく、公式リファレンスとCGI-24に沿って採用する検証規則である。混在改行の許容と不正出力への502はチーム方針。**1xxはコードの範囲検査を通すが、1xxだけで終了するCGIはCGI-74に従い502とする。** 1xxは中間応答であり、HTTP/1.0要求への送出制限と最終応答が必要なHTTPの規則に従って決定した([RFC 9110 §15.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.2))。本決定は、CGIの1xxを最終応答としてそのまま送出する合意ではない。

### Location・リダイレクトの合意済みの動作

**確定(CGI-62〜66、残項目2/10)**: Locationがある応答は、本構成では次のリダイレクトの形式として扱う。

- 前後SP/HTAB除去後の非空Locationは、スキーム付きの絶対URIと任意のフラグメントとして一般構文を検証する。スキームはHTTP/HTTPSに限定せず、[RFC 3986](https://www.rfc-editor.org/rfc/rfc3986.html)の一般URI構文を基準とする。空白・不正な%表記等は502とする。値をデコード・パス正規化せず、DNS照会や宛先への接続確認も行わない。重複・空値はCGI-52・59の規則を維持する。
- 有効なLocationだけがあり、本文が空なら302へ変換する。例えば`Location: https://example.com/next`と終端空行だけの出力は、この形式となる。サーバー独自のCGI拡張ヘッダーは定義しないため、Location単独の形式に追加の有効なCGI出力ヘッダーを認める合意ではない。
- Location・Status・Content-Typeがある形式では、Statusを301・302・303・307・308に限定し、指定されたコードと本文を返す。本文は空でもよい。その他の応答ヘッダーの転送はCGI-69〜72に従う。
- Locationに本文だけ・Statusだけ・Content-Typeだけを伴う等、上の形式に合わなければ502とする。StatusやContent-Typeを補完しない。Locationのない通常の文書応答で受理するStatusの100〜599は維持する。
- スキームのない`/other`・`other`・`//example.com/`等の非空Locationは502とする。内部再処理や絶対URIへの補完、外部向け302への変換は行わない。

応答形式は[RFC 3875 §6.2.2〜6.2.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.2.2)、Locationの定義は[§6.3.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3.2)を参考にする。課題PDFの個別指定ではなく、対応する本文付きリダイレクトのコードを5種類に限定し、対象外・不正な形式は502とするチーム方針である。

### HTTP応答への変換の合意済みの動作

**確定(CGI-67〜74、残項目3/10)**: CGIの出力を検証してから、WebservがHTTP応答を組み立てる。

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

課題PDFは相対ファイルアクセスのために正しいディレクトリでCGIを実行することを要求し、[RFC 3875 §7.2](https://www.rfc-editor.org/rfc/rfc3875.html#section-7.2)もUNIXでスクリプトの置かれたディレクトリをcwdにすることを推奨している。実行パス・cwdの受け渡しAPIと担当境界、chdir失敗時の検出・終了方法は引き続き別途確定する。

### 後続パスの合意済みの動作

**確定(CGI-25)**: C-24・C-25に従ってデコード・正規化し、locationを選択した後、そのlocationのroot/aliasで対応するパスを先頭から辿る。設定拡張子で終わる実在する通常ファイルに最初に到達したところをスクリプトとし、それより後ろをPATH_INFOとして渡す。例えばtest.pyが通常ファイルなら、`/cgi/test.py/extra/a.py`でも実行対象はtest.py、PATH_INFOは`/extra/a.py`となる。tools.pyがディレクトリなら、名前の末尾が.pyでも実行せず、その配下のパスを辿る。後続部分はファイルとして探索しない。設定されたindexにもCGI-10の判定を適用する。

この特定手順はチームの採用仕様。RFC 3875 §3.2–3.3はパスをスクリプトと後続部分に分ける考え方を示し、具体的な対応付けはサーバーの設定・実装に委ねている。実行パス・cwdの動作はCGI-26で確定済み。PATH_INFOがない場合の空値/省略はCGI-29で確定済み。アクセス失敗の細則は引き続き別途確定する。

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

### 残る決定の進め方(10項目)

細かな決定を10件ずつ提示する方式ではなく、残る全体を以下の10項目にまとめ、1項目ずつ合意する。既存CGI-IDは詳細の記録に使い、進捗はこの10項目で管理する。

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

### 担当境界と受け渡しAPIの合意事項(残項目6/10)

**確定(CGI-91〜98、残項目6/10)**: 以下の担当境界・型・APIで受け渡す。課題PDFとRFCはCGIへの入力・出力を定めるが、Webserv内部のクラス名・担当・APIは指定しない。

| 担当 | 処理 |
|---|---|
| rysato / HttpRequest・Router | 要求の構文・重複を検証し、location・スクリプト・cwdを解決してCGI環境の内容を生成する。完成済み本文とともにCgiRequestにまとめる。戻ってきたstdoutをCGIとして解析・検証しHttpResponseを生成する |
| tasugiya / EventLoop・CgiExecutor・CgiProcess | 接続のIPv4・実際のサーバーIPv4とポートを取得しConnectionInfoで渡す。CgiRequestを使って起動し、pollでパイプI/O・上限・期限・子の終了を管理する。成功時のstdoutか失敗時のコードをCgiResultで返す |

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

enum RouteResult { RESPONSE_READY, CGI_REQUIRED };

// Router(rysato)。CGI起動前の拒否もRESPONSE_READYでHTTP応答を返す。
RouteResult Router::route(const HttpRequest& request,
                         const ServerConfig& config,
                         const ConnectionInfo& connection,
                         HttpResponse& response, CgiRequest& cgi);
// 戻り値に対応する出力引数だけを有効とする。

// CgiExecutor(tasugiya)。0は起動手続き成功、非0は開始時のエラーコード。
// CGIの完了を待たずに返り、以後はEventLoopが進行する。
int CgiExecutor::start(const CgiRequest& request, CgiProcess& process);

// CgiProcess(tasugiya)。結果が未確定ならfalse。確定後に一度だけtrue。
bool CgiProcess::takeResult(CgiResult& result);

// Router(rysato)。実行成功後のstdoutを検証し、0または不正出力の502を返す。
// 0のときだけresponseが有効。成功後にEventLoopがapplyErrorPageを一度だけ適用する。
int Router::parseCgiOutput(const std::string& output, HttpResponse& response);
```

実行成功とは正常な子の終了とstdoutのEOFが確認できたことを指し、出力内容の妥当性はparseCgiOutputで別途判定する。失敗時は部分的なstdoutをHTTPへ送らない。生のwaitpidの終了状態はtasugiya側で解釈し、HTTP側へ渡す必須データを増やさない。失敗原因とコードの対応・error_page適用は10/10のCGI-128〜135に従い、結果を確定できる後始末の条件は8/10のCGI-107〜118で確定済み。

既存のHttpRequestの単一値getterだけでは重複・全行処理を実装できないため、`const std::vector<std::pair<std::string, std::string> >& getHeaders() const`も公開する。名前はASCII小文字、値は合意済みの前後SP/HTAB除去後で、全行を保持する。辞書への上書き・結合で重複を失わず、Connectionの全行処理・転送対象の重複検査に用いる。

HttpResponseには`void setStatus(int code, const std::string& reason)`を追加し、CGIの検証済み説明文(空も含む)をそのままステータス行に反映する。既存の`setStatus(int)`はサーバーが生成する応答の標準文言に使う。CGIの出力検証規則を変更する合意ではない。

CgiRequest/CgiResultの所有権・コピー/参照・envpへの変換と寿命は7/10のCGI-99〜106で確定済み。起動時の具体的な失敗検出は9/10に従う。この合意はCGIの受け渡しに必要な公開部分を定める。HTTP全体の残るAPI・内部実装は別途扱う。

根拠: [RFC 3875 §3.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-3.1)のサーバーの責任、[§4](https://www.rfc-editor.org/rfc/rfc3875.html#section-4)の入力、[§6.1](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.1)の出力処理、および[Linux execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)のargv/envpを踏まえた担当・APIのチーム合意。

### データ・環境変数の所有権と寿命の合意事項(残項目7/10)

**確定(CGI-99〜106、残項目7/10)**: 6/10のAPIを維持して以下の所有権・寿命で管理する。課題PDFは接続・メモリの後始末を要求するが、内部データをコピーするか参照するかは指定していない。

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

### 切断・終了時の後始末とfd継承の合意事項(残項目8/10)

**確定(CGI-107〜118、残項目8/10)**: 7/10の所有関係を維持して以下の後始末を行う。課題PDFのノンブロッキング・クライアント切断処理・単一poll・要求を無期限に待たせない要件に従う。SIGKILLの選択、回収方法、fdの継承防止方法はチーム方針である。

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
- 現在のpoll結果には閉じたfdの古い通知が残り得る。処理前に対応する管理オブジェクトと現fdが有効か確認し、閉じたものはスキップする。Client/CgiProcessの削除は現在のpoll結果を処理し終えた時点まで遅らせ、古い通知で解放済みオブジェクトや再利用されたfdを操作しない。

**子に残すfd**

listenソケット・acceptしたソケット・全CGIパイプに、生成時にfcntl(fd, F_SETFD, FD_CLOEXEC)を設定する。Pythonのexecve成功時に不要なfdが閉じるため、他Clientの接続や他CGIのパイプをPythonへ残さない。現在のCGIのstdin/stdoutはdup2で0/1へ接続し、元のパイプ端・使わない端を閉じる。0/1のFD_CLOEXECは解除する。同じfdへのdup2でも確実に解除し、継承するstderrの2もFD_CLOEXECを解除する。子の起動失敗時は9/10の方法で終了してfdを解放する。

このF_SETFDはLinux前提の方針。課題のfcntlの追加制限は「For MacOS only」の節であり、Linuxでの本方針に適用しない。起動時・accept時・CGI起動時の設定失敗は成功扱いにせず、作成済み資源を後始末する。失敗のコードはCGI-129・134に従う。全fdの一覧をstartへ渡す追加APIや/proc走査は設けない。

親のSIGPIPE無視は維持し、子はexecve前にSIGPIPEをSIG_DFLへ戻す。回収のためのSIGCHLD無視は行わない。シグナル設定・dup2・chdir・execve等の子側失敗の検出と終了方法は9/10で確定する。対象はWebservが直接起動したCGIの子で、孫プロセスの列挙・プロセスグループの管理は追加しない。

根拠: [Linux wait(2)](https://man7.org/linux/man-pages/man2/waitpid.2.html)のWNOHANGと子の回収、[kill(2)](https://man7.org/linux/man-pages/man2/kill.2.html)、[pipe(7)](https://man7.org/linux/man-pages/man7/pipe.7.html)のEOFと不要なfd、[poll(2)](https://man7.org/linux/man-pages/man2/poll.2.html)のPOLLHUPと残データ、[close(2)](https://man7.org/linux/man-pages/man2/close.2.html)の再試行の扱い、[F_GETFD/F_SETFD](https://man7.org/linux/man-pages/man2/F_GETFD.2const.html)と[dup(2)](https://man7.org/linux/man-pages/man2/dup.2.html)のFD_CLOEXEC、および[signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)のSIGKILLと無視されたシグナルの継承。

### 子の起動失敗・終了方法と時間計測の現時点の採用案(残項目9/10)

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
- Linuxの/procが利用できる実行環境を前提にする。壁時計ではなく起動後の経過時間を用い、サスペンド時間も含む。ファイルの再読みによるI/Oは増えるが、禁止された時刻関数を使わずに10秒を測る方法とする。通常接続の60秒の起点やDate/ログの暦日時は別議題とし、ここでそれらの仕様を確定した扱いにしない。

根拠: [Linux proc_uptime(5)](https://man7.org/linux/man-pages/man5/proc_uptime.5.html)の2つの数値とサスペンド時間、[カーネルのuptime.c](https://github.com/torvalds/linux/blob/master/fs/proc/uptime.c)の起動後時間と小数2桁の出力、[generic.cの通常ファイル登録](https://github.com/torvalds/linux/blob/master/fs/proc/generic.c)。明示的に使うopen/read/close・fork/dup2/chdir/execve/fcntl/signal/waitpidは課題PDFの許可一覧内。mainのreturnは言語の通常の終了方法であり、明示的なexitの追加使用ではない。

### エラーの受け渡しと結合時の完了条件の合意事項(残項目10/10)

**確定(CGI-128〜135)**: 最後の10/10を合意済み。これで10項目の合意は完了。9/10の代替案の実環境検証と実装・結合試験はこれから行う。

課題PDFは未指定時のデフォルトエラーページを要求するが、CGIの正常なエラー応答を置き換えるか、内部のAPI・エラーの受け渡し方法までは指定しない。[RFC 3875 §6.3.3](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.3.3)はCGIのエラーステータスとスクリプト自体の失敗を区別し、[§6.4](https://www.rfc-editor.org/rfc/rfc3875.html#section-6.4)は本文を変更せず返すことを推奨する。本方針では確定済みConfig C-20の400〜599のエラーページ方針をCGIにも適用する。これはチームの選択であり、RFCや課題がCGI本文の置換を要求しているという意味ではない。

1. **受け渡しの順序**: rysato側のrouteが即時応答を返した場合はCGIを起動しない。tasugiya側のstart失敗はそのコードでmakeErrorを生成する。非同期のCgiResultはerrorStatusが非0なら部分stdoutを解析せずmakeErrorへ渡す。0ならrysato側のparseCgiOutputで全出力を検証し、解析失敗は502、成功はCGIの応答として扱う。失敗時のresponse出力引数は使用しない。
2. **失敗コード**: スクリプト未発見404・読み取り不可403・パス長414等の既存分類を維持する。親側のpipe/fork/fcntl・計時・管理の失敗は500、子側の準備/exec失敗・非0終了/シグナル終了・パイプI/O失敗・出力上限超過・不正出力は502、10秒経過は504。複数の原因が発生しても最初に確定したコードを維持する。正常に返されたStatus: 404等はCgiResultの実行失敗ではない。
3. **error_pageの適用**: 出力検証後の最終コードが400〜599なら、CGIが正常に返したエラーも含め、指定ファイルの本文へ置き換える。未指定・読み取り失敗なら内蔵HTMLにする。元のコードを維持し、有効なCGIのStatus説明文は空文字列も保持する。200〜399には適用しない。1xxは既決定のCGI-74により先に502となる。不正出力をerror_pageで覆って元のCGIコードのまま受理しない。
4. **置換する本文とヘッダー**: エラーページはHTMLとして扱い、Content-Typeはtext/html、Content-Lengthは置換後のバイト数で生成する。元本文の情報を残さないため、CGI由来のContent-*、ETag、Last-Modified、Digest、Repr-Digestを大小文字を区別せず除去する。この一律除去は最小構成のチーム方針。その他の既存転送対象ヘッダー(Allow、WWW-Authenticate、Set-Cookie等)は保持する。200〜399の応答にはこの除去を行わない。
5. **エラーページの読み取り**: 設定済みのファイルパスをそのまま使い、実行時に通常ファイルであることを確認して読む。空ファイルも読み取り成功なら空本文として採用する。失敗時は元コードの内蔵HTMLへ一度だけフォールバックし、再ルーティング・CGI実行・再帰的なエラー生成を行わない。CGIの8 MiB制限は元のstdoutに適用し、エラーページに新しいCGI専用上限は追加しない。
6. **内部の共通処理**: HttpResponseにvoid applyErrorPage(const ServerConfig& conf)を追加し、400〜599の場合だけ上記の本文置換を行う。makeErrorは同じ処理を内部で利用する。EventLoopはparseCgiOutputが成功した応答に一度だけapplyErrorPageを呼び、makeErrorの結果には再適用しない。ファイル読み取りと内蔵HTMLへのフォールバックをCGI専用に重複実装しない。
7. **応答を返せない場合**: クライアント切断時は停止・回収だけを行い、応答を生成しない。送信失敗後に別のエラー応答を追加送信しない。listen側のCLOEXEC設定失敗は起動失敗、acceptしたfdの設定失敗はその接続を閉じて監視へ登録しない。CGIパイプの設定失敗は上記の親500/子502の分類を使う。
8. **結合の完了条件**: 04のCGI受入項目を実装後に満たすことを条件とする。正常なGET/POST・chunked入力、入出力の並行処理、出力検証・エラーコード・上限・期限、切断/半閉鎖/SIGINT、fd継承・子の回収・寿命、CGI実行中の静的配信を確認する。加えて正常なCGIの404/599の置換、200の非置換、エラーページ未指定/読取失敗/空ファイル、元本文の符号化等の除去とAllow等の保持、不正な404出力が502になることを確認する。9/10の代替案は実環境で検証し、必要なら方法を見直す。10項目の合意完了と、未実装の結合試験合格は別に扱う。

### 合意済みの外部に見える仕様

以下は合意済みの仕様の確認表。実装・試験の完了を意味しない。

| 合意 | 論点 | 推奨案・確定する内容 |
|---|---|---|
| [x] | CGIファイルパスの結合・検証 | 残項目5/10としてCGI-84〜90で確定済み。root/aliasの結合、スクリプトまでの検査、種別・R_OK、Pythonの要求時事前検査とリトライの省略、stat/access失敗の分類を採用する。C-24〜26のURL処理・symlinkの運用前提は維持する |
| [x] | 実行パス・cwdの受け渡し | CGI-26の動作とCGI-91〜98の担当・APIで確定済み。rysatoが解決したスクリプト絶対パスとcwdをCgiRequestでtasugiyaへ渡す |
| [x] | HTTPヘッダーの転送 | CGI-39〜49と残項目4/10のCGI-75〜83で動作を確定済み。転送対象、追加の除外、trailer、Connectionの複数行、共通の値の文字検査、Content-Type、除外対象の重複、内容符号化・NULの規則に従う。HTTPパーサーの全行保持・受け渡しAPIは6/10、所有権・寿命は7/10で確定済み |
| [x] | 出力の基本構文・値、Content-Type・Statusの検証 | 残項目1/10としてCGI-56〜61で確定済み。Statusコードは100〜599。Location、Content-Length、その他のヘッダーとHTTP応答への変換は残項目2/10・3/10で確定済み |
| [x] | リダイレクトの詳細 | 残項目2/10としてCGI-62〜66で確定済み。Locationの一般URI構文、単独時の302、Location・Status・Content-Typeの形式、対応する5種類のコード、不完全な形式と内部・相対Locationの502を採用する |
| [x] | その他の応答ヘッダー・HTTP応答変換 | 残項目3/10としてCGI-67〜74で確定済み。長さの数値・実長照合・生成、全ヘッダーの重複拒否、接続用ヘッダーの除外、非空Transfer-Encodingの502、その他のヘッダー転送、204・205・304の本文禁止と1xxだけで終了するCGIの502を採用する |
| [x] | stderr | CGI-51で確定済み。Webservのstderrを引き継ぎ、stdoutへ混ぜず専用のパイプ・監視・収集処理を追加しない。stderrへの出力だけでは502にしない |

内部リダイレクトを対象外とする決定により、Requirementsと本書のCGI方針は「RFC 3875を参考に、対応範囲を限定」とする。課題PDFに個別の対応範囲が明記されていない機能を、課題から免除されたと断定しない。

### 合意済みの分担・受け渡しと未実施の結合テスト

大枠の分担はtasugiya=ネットワーク/イベントループ、rysato=HTTP/Config、CGIの結合は共同。下表の処理の担当は6/10で確定済み。所有権・寿命・後始末・エラー適用境界も合意済み。以下のテストの未チェックは、実施・合格が未完了であることを示す。

| 担当 | 責任 |
|---|---|
| tasugiya | pipe/fork/execve、パイプI/O、pid・fd・実行状態の所有、時間管理、子の終了・回収、切断時の後始末 |
| rysato | 実行対象・パスの解決、環境変数の内容、CGI出力の解析、HTTP応答の生成 |
| 共同 | 以下の受け渡し、確定済みエラー・上限の実装境界、結合テスト |

- [x] **APIと戻り値**: CGI-91〜98で確定済み。CgiRequest / CgiResult / ConnectionInfo、RouterのrouteとparseCgiOutput、CgiExecutorのstart、CgiProcessのtakeResultを使う。argvの可変一覧・生の終了状態を別途受け渡さない。
- [x] **データの寿命**: CGI-99〜106で確定済み。本文はCgiProcessへのコピー、起動情報はfork前の一時データ、stdoutはCgiResultへのswapとする。EventLoopがClient/CGIを所有し、Client破棄前に非所有ポインターを外す。
- [x] **切断・終了時の処理**: CGI-107〜118で確定済み。監視解除・一度だけのclose・必要なSIGKILL・WNOHANG回収・結果受け渡し後の削除とする。fdはCLOEXECで継承を制御し、親子のSIGPIPE設定・半閉鎖・古いpoll通知・SIGINT終了も合意済み。
- [x] **子の起動失敗と時間計測(現時点の採用案)**: CGI-119〜127を採用する。mainへのreturnと子専用の明示的解放による終了、/proc/uptimeによる計時は、実装・実行環境・動作確認で見直し得る代替案。失敗時の親用処理の回避・終了処理の副作用・計時精度と負荷を検証して判断する。
- [x] **エラーを返す境界**: CGI-128〜134で確定済み。実行失敗時は部分stdoutを使わずmakeError、成功時はparseCgiOutputで検証後にapplyErrorPageとする。最初の失敗コードを保持し、正常なCGIの400〜599も既存Configの本文置換方針に従う。

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
| C4–C5, C7 | S〜M | 短時間。ただしC7は許可関数の確認待ち |
| C6(アップロード) | L | RFC読み込み+設計判断(方式・サニタイズ・メモリ) |
| C8–C10(CGI) | XL | 共同。tasugiya=C8、rysato=C9を主に調査し、C10の結合設計を合同で行う |
| C11 | M | 後回し可 |
