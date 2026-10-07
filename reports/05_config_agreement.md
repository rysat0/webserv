# 05 Configの設計と合意事項

> 記録日: 2026-10-04。Configの6項目合意完了: 2026-10-05。資料整理: 2026-10-07。対象: webservの設定ファイル。作成時のブランチ: `research`。

NGINXを基本に、課題要件を満たす範囲でConfigの文法・設定項目・検証を小さくするための資料。本書のHTTP/1.1要求規則はHTTP-04に従いHTTP/1.2〜1.9にも適用する。要求先の追加の文字検査はHTTP-05を参照する。Configのパス規則・限定文法と、CGI関連の設定方針を確定事項として記録する。

<a id="config-decisions"></a>

## Configの確定事項

| ID | 決定 | 具体例・動作 |
|---|---|---|
| C-01 | **rootとaliasを区別する** | rootはディレクトリにURI全体を追加する。aliasはlocationのprefixを指定ディレクトリへ置き換える。課題のprefix置換例はaliasで実現する |
| C-02 | **error_pageはファイルを直接指定する** | NGINXのURI指定・内部リダイレクトは行わない。400〜599では指定ファイルを本文として読み、未指定・読み取り失敗時は元コードの内蔵エラーページを返す。100〜399には適用しない(C-20) |
| C-03 | **相対パスは設定ファイルのディレクトリ基準** | `conf/default.conf` 内の `root ../www/site1;` はプロジェクトの `www/site1` を指す |
| C-04 | **文法を5点に絞る** | ファイル直下にserver、server内に設定とlocationを置く。設定は`;`、ブロックは`{}`、空白・タブ・改行を区切りとし、`#`から行末をコメントとする。引用符・エスケープ・変数・include・正規表現・locationの入れ子・reloadとhttp/eventsの外枠を省く。未対応の書き方・文法間違いは理由を表示して起動を中止する |
| C-05 | **省略時は原則NGINXの既定値を適用する** | 対応する設定項目にNGINXの既定値があれば採用し、省略自体をエラーにしない。明示offを強制しない。独自項目の省略値はNGINXの値と区別して決める。継承なし・相対パスの基準は維持する。listenのみC-39の固定値を優先する |
| C-06 | **allow_methods省略時はGETのみ許可** | POST・DELETEは必要なlocationで明示する。明示した場合はそのリストを使う。CGI用locationではGET/POSTの範囲に限定し、DELETEは405とする既決定を維持する |
| C-07 | **upload_store / cgi_extension省略時は無効** | Webservによるアップロード保存・CGI実行をそれぞれ無効にする。保存先・拡張子・Pythonのパスを自動補完しない。明示offは必須にしない |
| C-08 | **同じlocationでroot / aliasを併記しない** | 両方が明示されていたら、記述順によらず設定エラーとして起動を中止する。aliasのみ指定した場合は既定のrootを補完しない。配信locationで両方省略した場合は既定のrootを適用する |
| C-09 | **aliasはディレクトリ対応・両側の末尾 `/` 必須** | aliasを使うlocationのprefixとaliasの値をともに `/` で終える。欠けていれば自動補完せず設定エラーで起動中止。単一ファイルへのaliasは対象外。rootを使うlocationにはこの末尾制限を追加しない |
| C-10 | **明示するlistenはIPv4:portのみ** | IPv4はドット区切りの4個の十進数(各0〜255)、ポートは十進数の1〜65535。0.0.0.0も可。ポートのみ・ホスト名・IPv6・追加オプション・不正値は設定エラーで起動中止。省略時の既定値は維持する。指定回数はC-11、server間の重複と競合はC-12に従う |
| C-11 | **listenの明示は1serverに最大1回** | 同じ値でも異なる値でも2回以上の指定は設定エラーで起動中止。省略時は既定値で1組を補完する。複数ポートはserverブロックを分ける。server間の重複と競合はC-12に従う |
| C-12 | **listenの重複・競合を拒否し、名前ベースの仮想ホストを省く** | server間の同一IPv4:portと同じポートでの0.0.0.0/特定IPの併用は、既定値補完後に設定エラーとして起動中止。異なる特定IPなら同じポートも設定可。server_name・Hostによるサイト選択は実装せず、HTTP/1.1のHost検証は維持する |
| C-13 | **ボディ上限は正の十進整数・バイト単位のみ** | client_max_body_size省略時は1 MiB。明示値の0・符号・小数・単位接尾辞・保持する整数型の範囲超過は設定エラーで起動中止。要求ボディは復号後の全体で判定し、上限と同じサイズは受理、超過は413 |
| C-14 | **設定項目の重複は起動エラー** | 通常の設定項目は同じブロック内に最大1回とし、同じ値でも重複を拒否する。error_pageは同じserver内で異なるコードなら複数可、locationは同じserver内で異なるprefixなら複数可。同一コード・同一prefixの重複は起動エラー |
| C-15 | **indexはファイル名1個・未発見時はautoindexへ** | 複数候補・`/`を含むパス・`.` / `..` は起動エラー。省略時index.html。存在は起動条件にしない。ディレクトリへのGET要求でindexがなければautoindexがonなら一覧、offなら403。ディレクトリ自体がなければ404。読み取り不可を一覧表示への切り替え条件にしない。選ばれたindex.pyにも既存のCGI判定を適用する |
| C-16 | **autoindexは小文字on/off・HTML固定** | 引数は小文字on/offの1個のみ、省略時off。それ以外の値・引数数は起動エラー。表示形式・サイズ表記・時刻表示等の追加Configは作らない。index優先を維持し、offでも個別ファイルへのアクセスは妨げない |
| C-17 | **allow_methodsは大文字GET/POST/DELETEの許可リスト** | 空白区切りで重複なく1個以上。空・重複・小文字・カンマ区切り・その他の値は起動エラー。順序に意味はなく、明示リストにGET等を自動追加しない。省略時GETのみとCGI用locationへのDELETE 405は維持する |
| C-18 | **upload_storeはoffまたは既存ディレクトリ1個** | 独自名upload_storeを採用。引数は小文字offまたは保存先パス1個、省略時は無効。絶対パスと設定ファイル基準の相対パスを受理。保存先は起動時に存在するディレクトリを要求し、自動作成しない。GET公開先はroot/aliasで別途指定する |
| C-19 | **cgi_extensionはoffまたは拡張子とPython絶対パス1組** | 小文字offの1引数、または拡張子＋Python絶対パスの2引数。拡張子は`.`＋ASCII英数字1文字以上で大小を区別。不正な拡張子・不足/余分な引数・複数組・相対パスは起動エラー。`.cgi`等でも実行言語はPythonのみ |
| C-20 | **error_pageは100〜599のコード1個＋ファイルパス1個** | server内に3桁の十進コードと絶対/設定ファイル基準の相対パスを指定。複数コード・ステータス変更・不正値/引数数は起動エラー。本文差し替えは400〜599のみ。100〜399の指定は受理するが応答には適用しない。適用対象で未指定・読み取り失敗時は元コードの内蔵ページ。本文禁止規則は維持 |
| C-21 | **returnは301/302＋固定HTTP(S)絶対URL** | location内にコードとURLの2引数。URLはhttp://またはhttps://で始まりホスト部分を持つ。相対URL・他コード・不正な値/引数数は起動エラー。変数展開・元URI/queryの自動追加・本文指定は省き、Locationで移動先を返す |
| C-22 | **returnがあるlocationはリダイレクト専用** | allow_methodsは併記可・省略時GETのみ。root / alias / index / autoindex / upload_store / cgi_extensionの明示はoffでも起動エラー。配信設定の既定値補完・配信パス検証は行わない。許可メソッドを先に確認し、禁止された対応メソッドには405とAllowを返す |
| C-23 | **locationは文字列の最長前方一致** | URLパスに一致する最も長いprefixを選び、設定順には依存しない。パス要素の境界は条件にせず、/aは/abcにも一致する。/a/は/a/file.txtに一致するが/a・/abcには一致しない。処理順序はC-24、正規化の具体的規則はC-25に従う |
| C-24 | **query分離後、パスを1回だけデコードする** | 最初の?で分離→パスの%XXを1回復号→正規化→location選択。queryは未デコードのままCGIへ。パスの%2F/%2fは/、+はそのまま。パスの不正な%表記・%00は400。正規化の具体的規則はC-25に従う |
| C-25 | **パスの`.`・`..`・連続スラッシュを正規化** | デコード後、location選択前に連続/を1個にし、.を除去、..で前の要素へ戻る。URLの先頭/より上へ戻る要求は400。末尾のディレクトリを示す/を保ち、file..txt・.hiddenは変更しない。symlinkの運用前提と制限はC-26に従う |
| C-26 | **symlinkを置かない運用前提・検出機能は省く** | 公開・CGI・アップロード領域は二人が管理する。symlinkの検出・拒否機能やConfig項目は追加しない。置かれた場合はOSが辿り得るため、公開領域外へのアクセスを完全に遮断する仕様とはしない。URLのデコード・正規化は維持する |
| C-27 | **実ディレクトリへのGETは末尾/を301で補完** | location選択・許可メソッド確認後、配信先が実ディレクトリでパスに末尾/がなければ301。queryを保持し、/付き要求でindex/autoindexへ進む。POST・DELETEには適用せず、Configのreturn指定は優先する。locationの一致規則は変更しない |
| C-28 | **自動補完URLはHost優先・なければ接続のローカルIP/port** | http絶対URLを返す。有効なHostは明示ポートごと使用し、ポートなしなら追加しない。HTTP/1.0のHost欠落時だけacceptした接続のサーバー側IP/portを使用。両版ともHostの重複・不正は400、1.1は欠落も400。tasugiyaが接続情報、rysatoがHost検証・URL生成を担当。転送ヘッダーは使わない |
| C-29 | **自動補完のパスを再エンコードしqueryは保持** | デコード・正規化済みのパスに/を付け、ASCII英数字・-._~・区切り/以外をバイトごとに大文字%XXへ変換。queryは元のまま付ける。内部パスから一度生成し、Configのreturnは変換しない |
| C-30 | **HostはIPv4またはASCIIホスト名＋任意port** | 前後SP/HTABを除去。名前は英数字と-の要素を.で区切り、各要素の先頭/末尾は英数字、末尾.は1個まで。IPv4は各0〜255、portは明示時1〜65535。大小両方可。空値・重複・不正値・IPv6・%表記は400。DNS照会なし。Hostの省略規則はC-28を維持 |
| C-31 | **HTTP要求行・ヘッダーはCRLF限定** | HTTP/1.0・1.1とも単独LF・不正な単独CRは400。断片末尾のCRは続きを待ち、次がLFなら正常。ボディの改行には適用しない。CGI出力のLF/CRLF受理とHTTP応答のCRLF生成は維持する |
| C-32 | **要求行8 KiB・ヘッダー合計32 KiB** | 受信バイト数でCRLF込み、ヘッダーは終端空行込み・要求行とボディを除外。上限と同値は受理、超過を受信中に検出して414/431。コード内定数でConfigを増やさず、ヘッダー単位/個数の別上限も設けない。ボディとCGI出力の上限は維持 |
| C-33 | **HTTP要求の折り返し拒否と空白処理** | 両バージョンともSP/HTABで始まる非空ヘッダー行とコロン直前の空白・タブは400。値の前後のSP/HTABだけを除去し、空のCRLF行でヘッダー終了。ボディとCGI出力には適用しない |
| C-34 | **HTTP要求ヘッダー名はtoken・小文字で保持** | HTTP/1.0・1.1ともASCII英数字とRFCの記号からなる1文字以上の名前を受理。空・不正文字・コロンなしは400。名前はASCII小文字に揃え、値は一律に小文字化しない |
| C-35 | **要求のContent-Lengthは最大1行** | HTTP/1.0・1.1とも同値でも重複は400。名前の大小文字違いも重複として扱う。カンマ入りも400とし、結合・補正しない。他のヘッダーやCGI出力には一律適用しない |
| C-36 | **長さ指定のない要求は空ボディ** | HTTP/1.0・1.1ともContent-LengthとTransfer-Encodingが両方なければPOSTも0バイト。ヘッダー完了で受信を終え、省略だけで411にしない。不正な指定は既存規則で拒否する |
| C-37 | **Config文法の細則と空location** | serverを1個以上、各serverにlocationを1個以上必須とする。ブロック後の`;`は禁止、設定名は大小区別、同じ階層内の記述順に依存しない。特殊location・if/rewrite/try_filesを拒否。空locationは既定値で受理する |
| C-38 | **location prefixの受付範囲** | /始まり、ASCII英数字と/-._~のみ。連続//と単独の./..要素は起動エラー。設定値は補正せず、大小文字を区別する。location /は任意、マッチなし404。要求URLの文字制限には転用しない |
| C-39 | **listen省略時は0.0.0.0:8000固定** | 起動権限の判定を省き、listenだけNGINX既定値から変更する。明示値を優先し、補完後の競合は起動エラー。bind失敗時に別ポートへ切り替えない |
| C-40 | **起動時のファイル検証範囲** | root/alias/upload先は存在するディレクトリ、Pythonは通常ファイルとX_OKを確認。既定rootにも適用。ディレクトリの追加権限検査・自動作成・配下全件検査を省く。index/error_pageは要求時に扱い、redirect専用locationの配信パスは検証しない |
| C-41 | **locationの設定の組み合わせを起動時検証** | 配信POSTにはuploadかCGIの有効化が必要。両方有効・CGI有効でDELETE許可は起動エラー。off/省略は無効、returnはPOST処理先検証の対象外。機能有効でもPOSTは自動許可しない |
| C-42 | **Configの型・API・所有関係** | parse(path)が検証済みConfigを値で返す。失敗はruntime_errorをmainで処理。型はstring/int/size_t/vector/set/mapとenum。CGIは文字列2つの1組。getterのみ公開し、mainのconst Configをtasugiyaがconst参照する |

C-01はNGINXのroot / aliasと同じ意味。C-02は実装を簡略化するための意図的な差異。C-03はこのプロジェクトで採用するパス解決の基準である。

## CGIに関する追加の確定事項

- CGI-26: 子プロセスでスクリプトの置かれたディレクトリへchdirし、Pythonとスクリプトをともに絶対パスで指定して起動する。cwdはPATH_INFOから決めず、追加のConfig項目は設けない。

- 必須部分はPythonのみ。`cgi_extension .py /絶対パス/python3` で指定したPythonを `execve` で起動し、選択されたlocationのCGI設定と拡張子で対象を判定する。
- CGI用locationはGET/POSTに限定する。DELETEは405とし、スクリプトの削除へ進めない。
- 設定されたindexとして選ばれた `index.py` にもCGI判定を適用する。基本的な後続パス(`/cgi/test.py/extra`)をPATH_INFOとして渡す。
- アップロードとCGIはlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にアップロードさせない。
- CGIの10秒制限、ヘッダー8 KiB・stdout全体8 MiBはコード内定数とし、Config項目を増やさない。CGI同時実行数の独自上限・件数による503・待ち行列は設けない。

CGIの決定表は [Requirements](../Requirements.md#cgi-decisions)、説明・共有APIは [03のC12](03_config_cgi_upload.md#cgi-details) を参照する。

## 資料の役割

本書をConfigの決定表C-01〜42・詳細規則・設定例・公開APIの正本とする。課題要件とHTTP・CGIの決定表は [Requirements](../Requirements.md)、CGIの説明・共有APIは [03](03_config_cgi_upload.md#cgi-details)、今後の設計判断と検証は [04](04_testing_and_workplan.md#remaining-design) を参照する。

## 設定の合意内容

以下はConfigの確定事項をまとめたもの。限定文法の5点はC-04で確定済み。省略時の既定値はC-05〜07、listenの例外はC-39で確定済み。設定項目の指定回数はC-11・C-14で確定済み。文法の細則はC-37、location設定値はC-38で確定済み。Configの6項目はC-37〜C-42で合意済み。HTTP/Router/CGI等に属する残る論点は、Configの確定事項とは区別する。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- **確定**: 省略された設定項目には下表の既定値(NGINXを基本としlistenのみC-39の例外)を適用し、明示offを強制しない。明示された値を優先する。独自項目の既定値はチームで定め、allow_methodsはGETのみ(C-06)、upload_store / cgi_extensionは無効(C-07)とする。構文不正・不正な値をデフォルトで置き換えて受理する意味ではない。
- **確定**: 400〜599の本文置換時に、`error_page` 未指定・読み取り失敗なら内蔵エラーページを使う。100〜399には適用しない(C-20)。
- **確定(C-22)**: returnがあるlocationはリダイレクト専用とする。allow_methodsは併記でき、省略時はGETのみ。root / alias / index / autoindex / upload_store / cgi_extensionの明示は、offであっても設定エラーとして起動を中止する。配信項目のデフォルト補完・配信パスの検証は行わない。この併記禁止は本課題用の独自制限である。
- **確定(C-11)**: listenの明示は1つのserverにつき最大1回。同じ値でも異なる値でも2回以上の指定は設定エラーで起動中止。省略時は既定値で1組を補完し、複数ポートはserverブロックを分けて対応する。**確定(C-12)**: Hostによる仮想ホスト選択は行わない。
- **確定**: NGINXの `root` と `alias` は意味を分けて採用する。**確定(C-08)**: 同じlocationでrootとaliasを併記した場合は、記述順によらず設定エラーとして起動を中止する。aliasのみ指定した場合は既定のrootを補完しない。

### 省略時の既定値

C-05により、既定値のある設定を省略したことだけでは起動エラーにしない。以下は本課題で扱う設定項目に限った表であり、未対応のNGINXディレクティブや継承機構を追加する意味ではない。

- **確定(C-39)**: listenを省略したserverは、起動権限に関係なく`0.0.0.0:8000`で待ち受ける。権限判定は実装しない。C-05のNGINX既定値採用方針のうちlistenだけをこの固定値に変更し、他の項目の既定値は維持する。明示したlistenはC-10・C-11に従ってそのIP・ポートを使用し、80番も指定可能とする。既定値補完後の重複・競合はC-12に従って起動エラーとし、複数serverでlistenを省略して同一アドレスになった場合も拒否する。bind失敗時は起動を中止し、別IP・別ポートへ自動変更しない。

| 項目 | 省略時の値・動作 | 位置付け |
|---|---|---|
| `listen` | `0.0.0.0:8000`(C-39) | 省略可。起動権限の判定なし。NGINXとの意図的な差異。bind失敗時は起動エラーとし、別ポートへ切り替えない |
| `client_max_body_size` | 1 MiB = 1,048,576バイト | NGINXの`1m`を既定値として採用。明示する場合は正の十進整数・バイト単位のみとし、単位接尾辞は受理しない(C-13) |
| `root` | `html` | 配信locationでroot/aliasの両方がなければ適用。aliasが明示されている場合は補完しない |
| `alias` | 指定なし | 勝手にaliasの対象を作らない |
| `index` | `index.html` | `index.py`は自動選択しない。CGIのindexを使う場合は明示する |
| `autoindex` | `off` | 明示offと省略は同じ動作 |
| `error_page` | 個別ファイル指定なし。400〜599では内蔵エラーページを使う | C-02・C-20。100〜399には適用しない |
| `return` | リダイレクトなし | 通常の配信locationとして扱う(C-22) |
| `upload_store`(独自) | **確定**: 無効 | C-07。Webservによるアップロード保存を行わず、保存先を自動補完しない |
| `cgi_extension`(独自) | **確定**: 無効 | C-07。CGIを実行せず、拡張子やPythonのパスを自動補完しない |
| `allow_methods`(独自) | **確定**: GETのみ許可 | C-06のチーム独自の既定値。POST・DELETEは必要なlocationで明示する。CGI用locationのGET/POST限定・DELETEへの405は維持する |

既定の`root html;`にもC-03を適用する。例えば`conf/default.conf`では`conf/html`を指す。NGINXのインストール先等のパスをそのまま使うわけではない。既定値の補完と上位設定の継承は別であり、継承なしの方針は維持する。必須のブロック構造や引数不足・重複・不正値の検証も、設定項目の省略とは区別する。

根拠: [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)。

### 文法

基本文法はC-04、以下の文法細則はC-37で確定済み。locationのprefixはC-38に従う。起動時のファイル検証はC-40に従い、設定の組み合わせはC-41に従い、型/APIはC-42に従う。

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブ(任意)とlocationブロック(1個以上、順不同) }
location    := location URL-prefix { location用ディレクティブ(0個以上) }
単純設定    := 名前 引数... ;
```

- **確定**: 最上位には `server` だけを書き、server内に設定とlocationを置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。**確定(C-37)**: ファイルにはserverを1個以上、各serverにはlocationを1個以上置く。空ファイル・コメントだけのファイル・locationがないserverは設定エラーで起動を中止する。
- **確定**: 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。単純設定は`;`で終える。改行そのものは設定の終わりではなく、`;`を省略できない。ブロックは`{}`で囲む。**確定(C-37)**: ブロック末尾の `}` に `;` は付けず、`};` は設定エラーとする。
- **確定**: `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を使った未対応の書き方は設定エラーとする。
- **確定**: メソッドはC-17、`on` / `off`はC-16・C-18・C-19の大小文字規則に従う。**確定(C-37)**: server/locationを含む設定名は大文字小文字を区別する。`root`を受理し、`ROOT` / `Root`は未知の設定として拒否する。同じ階層内ではディレクティブ・ブロックの記述順に依存しない。server内でlisten等をlocationの後に記述してもよい。重複はC-14に従って拒否し、後勝ちで上書きしない。
- **確定**: 未対応の書き方・文法間違いは理由を表示して起動を中止する。未知の名前や未閉じブロックを無視・補正して起動しない。各項目の階層・引数数・重複禁止規則は下表の確定事項に従い、実装時も確定済みの規則を適用する。
- **確定**: `include`、正規表現location、設定reloadは実装対象外。設定変更後は再起動する。**確定(C-37)**: 通常の `location URL-prefix { ... }` だけを受理し、`=` / `^~` / `~` / `~*` の修飾子、名前付きlocation、`if` / `rewrite` / `try_files` は設定エラーとして拒否する。
- **確定(C-37)**: `location / { }` のような空locationは文法上受理し、既決定の省略時規則を適用する(root html、index index.html、GETのみ、autoindex無効、upload/CGI無効)。空locationの受理は、root等の実在・権限検査を省く決定ではない。これらの起動時検証はC-40に従う。

**確定(C-14)**: 同じserver内のlisten / client_max_body_size、同じlocation内のroot / alias / index / autoindex / allow_methods / upload_store / cgi_extension / returnはそれぞれ最大1回。値が同じでも重複は設定エラーで起動を中止し、上書き・結合しない。rootとaliasの併記禁止(C-08)も維持する。error_pageは同じserver内でコードが異なれば複数可、locationは同じserver内でprefixが異なれば複数可。別ブロックで同じ設定項目を使うことや、1行のallow_methodsに複数メソッドを書くこととは区別する。1行の引数数・値の検証は、個別に確定した範囲に従う。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | **確定(C-11)**: 省略可・明示は最大1回。2回以上は起動エラー | **確定(C-10)**: 明示時はIPv4:portのみ。IPv4はドットで区切る4個の十進数(各0〜255)、ポートは十進数の1〜65535。`0.0.0.0`も可。ポートのみ、ホスト名、IPv6、追加オプション、不正な書式・範囲外の値は設定エラーで起動中止 |
| `client_max_body_size` | `client_max_body_size 1048576;` | 省略可・明示は最大1回(確定、C-14) | **確定(C-13)**: 正の十進整数・バイト単位のみ。`0`、符号、小数、`1m`等の単位接尾辞、std::size_tの範囲を超える値は設定エラーで起動中止。復号後ボディに適用し、上限と同じサイズは受理、超過は413 |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可(確定、C-14) | **確定(C-20)**: server内に3桁の十進数100〜599のコード1個とファイルパス1個。引数不足・余分な引数・範囲外・複数コード・ステータス変更指定は起動エラー。**確定(C-14)**: 同じserver内の同一コードの重複は起動エラー。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | **確定(C-37)**: 各serverに1個以上必須。複数可(C-14) | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させたら起動エラー(確定、C-14) |

- **確定(C-12)**: server間で同じIPv4:portが重複したら設定エラーで起動を中止する。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。listenを省略したserverも既定値を補完してから同じ規則で検証する。異なる特定IPv4アドレス同士は同じポートを指定できるが、実際のbind成功は起動時に確認する。
- **確定(C-20)**: error_pageはserver内に「100〜599の3桁の十進コード1個＋ファイルパス1個」を指定する。絶対パスと設定ファイル基準の相対パスを受理する。複数コードを1行に書く形式や `=200` 等のステータス変更は扱わない。同一server内の同一コード重複はC-14に従い起動エラー。設定の受付範囲を広げる決定であり、全コードの発生処理を追加する意味ではない。既存のHTTP規則が優先され、1xx・204・304等の本文を禁止する応答には、指定ファイルや内蔵ページの本文を付けない。未対応の応答コードを生成する機能も、この設定によって有効にはならない。**確定(C-20追記)**: 設定としては100〜599を受理するが、error_pageによる本文差し替えは400〜599の応答に限定する。100〜399の指定は応答に適用せず、通常の成功応答・リダイレクト・CGIの成功応答の本文を変更しない。100〜399の指定ファイルを読み込んだり、その未指定・読み取り失敗を理由に内蔵ページへ差し替えたりしない。

- error_pageの適用対象である400〜599の応答では、`error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- **確定(C-12)**: `server_name` とHostによる仮想ホスト選択は実装しない。接続を受け付けたlisten先のServerConfigを使用する。未対応のserver_name指定はC-04に従い設定エラーとする。HTTP/1.1要求のHost必須・欠落/重複/不正時400という検証は維持する。

### locationの設定

**確定(C-22)**: locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 省略可・既定GETのみ。大文字GET/POST/DELETEを重複なく1個以上(C-17)、明示は最大1回(C-14) | 同左 |
| `root` | `root ../www/site1;` | 省略可・明示は最大1回(C-14)。root/aliasとも未指定なら`html` | 指定不可(C-22) |
| `alias` | `alias ../uploads/;` | 省略可・明示は最大1回(C-14)。明示rootとの併記は禁止(確定、C-08) | 指定不可(C-22) |
| `index` | `index index.html;` | 省略可・既定`index.html`。ファイル名1個のみ(C-15)、明示は最大1回(C-14) | 指定不可(C-22) |
| `autoindex` | `autoindex off;` | 省略可・既定off。小文字on/offの1引数のみ(C-16)、明示は最大1回(C-14) | 指定不可(C-22) |
| `upload_store`（独自、C-18） | `upload_store off;` または `upload_store ../uploads;`(引数1個) | **確定**: 省略時は無効。明示は最大1回(C-14) | 指定不可(C-22) |
| `cgi_extension`（独自、C-19） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;`(1組のみ) | **確定**: 省略時は無効。明示は最大1回(C-14) | 指定不可(C-22) |
| `return`(C-21) | `return 301 http://127.0.0.1:8080/;`(2引数) | 指定不可 | 必須・1回(C-14・C-22) |

- **確定(C-17)**: `allow_methods` は大文字のGET / POST / DELETEから、空白区切りで重複なく1個以上指定する。空リスト・重複・小文字・カンマ区切り・その他の値(HEAD / PUT等)は設定エラーで起動中止。記述順に意味はなく、明示リストにGET等を自動追加しない。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止された対応メソッドは405とAllowヘッダーで応答する。HEAD等を後から採用する場合はHTTPの対応範囲とこのConfig規則を合わせて変更する。CGI用locationへのDELETEは引き続き405とする。
- **確定**: `allow_methods` を省略したlocationはGETのみ許可する。POSTを受けるCGI・アップロード用locationには、例えば `allow_methods GET POST;` を明示する。通常locationでDELETEも提供するなら `allow_methods GET POST DELETE;` 等を明示する。許可メソッドの設定だけでCGIやアップロード保存が有効になるわけではない。
- **確定(C-15)**: `index` は対象ディレクトリ内のファイル名1個のみ。複数候補、`/`を含むパス、`.` / `..` は設定エラーで起動を中止する。省略時はindex.html。indexファイルの存在自体は起動条件にしない。存在するディレクトリへのGET要求で指定ファイルがなければautoindexへ進み、onなら一覧、offなら403。ディレクトリ自体がなければ404。indexが存在するが読み取れない場合は、一覧表示へ切り替えない。HTTP 4/10で失敗応答を具体化済み(HTTP-20・21)。詳細は [Requirements](../Requirements.md#http-static-files) を参照。末尾 `/` のない実ディレクトリへのGETは、C-27の301を先に返す。
- **確定**: 設定されたindexとして `index.py` が選ばれた場合も、locationのCGI設定と拡張子で判定して実行する。index省略時は`index.html`を使い、`index.py`の自動補完は行わない。
- **確定(C-16)**: `autoindex` の引数は小文字の `on` または `off` を1個だけ受理し、省略時はoff。引数なし・複数引数・`ON` / `true` / `1` 等は設定エラーで起動中止。出力形式はHTMLに固定し、autoindex_format / autoindex_exact_size / autoindex_localtime等の追加設定項目は実装しない。indexを優先するC-15の流れを維持する。offは一覧表示のみを無効にし、個別ファイルへのアクセスを禁止する設定ではない。一覧の並び順・表示項目・取得失敗・上限はHTTP 5/10で合意済み。詳細は [Requirements](../Requirements.md#http-autoindex) を参照。
- **確定(C-18)**: 独自設定名は `upload_store` とし、引数は小文字 `off` または保存先ディレクトリのパス1個。引数不足・複数引数は設定エラーで起動中止。省略またはoffはWebservによるアップロード保存を無効にする。絶対パス・設定ファイル基準の相対パスを受理する。パス指定時は起動時に存在・ディレクトリ種別を検証し、不在・通常ファイルなら起動エラー。保存先や親ディレクトリを自動作成しない。GETで公開するにはroot/aliasを別途設定し、upload_storeから公開先を自動設定しない。実際の保存時の失敗処理、ファイル名・競合規則はHTTP 6/10で合意済み。詳細は [Requirements](../Requirements.md#http-upload) を参照。途中ファイル削除と失敗時の扱いはHTTP-49・50で合意済みで、std::removeを使う。
- **確定**: `cgi_extension` で拡張子とPythonの絶対パスを指定する。必須部分はPythonのみ。locationのCGI設定と拡張子で実行対象を判定し、通常ファイルのスクリプトを `execve` で起動したPythonに渡す。FastCGI設定は扱わない。
- **確定(C-19)**: `cgi_extension` は小文字 `off` の1引数、または拡張子とPythonの絶対パスの2引数(1組)のみ受理する。拡張子は `.py` に固定せず、`.` と1文字以上のASCII英数字(A-Z / a-z / 0-9)からなる値とし、大文字小文字を区別する。引数不足・余分な引数・複数組・不正な拡張子・Pythonの相対パスは設定エラーで起動中止。`*.py`等のパターンや正規表現は使わない。拡張子は実行対象の目印であり、`.cgi`等でも指定したPythonで実行する。省略時は無効(C-07)、同じlocationでの複数回指定は拒否(C-14)。Pythonの起動時検証はC-40に従う。
- **確定**: アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にはアップロードさせない。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- **確定(C-41)**: 同じlocation内の設定を、既定値補完後の有効/無効と許可メソッドで検証する。配信locationでPOSTを許可するならupload_storeまたはcgi_extensionのどちらか1つが有効であることを要求し、両方無効なら起動エラーとする。同じlocationでupload_storeとcgi_extensionが両方有効なら、POST許可の有無によらず起動エラー。CGI有効locationのallow_methodsにDELETEが含まれる場合も起動エラーとする。省略/offは無効として扱い、設定行の存在だけで有効とは判定しない。returnのリダイレクト専用locationはPOST処理先の検証対象外とし、C-22の併記禁止規則を維持する。CGIやuploadを有効にしてもPOSTは自動許可せず、GETのみでCGIを使う設定やupload有効かつPOST不許可の設定も受理する。起動後のCGI用locationへのDELETE要求は引き続き405とし、CGIファイルの削除へ進めない。CGIの拡張子判定、uploadとCGIのディスク上の保存先を分ける既決定も維持する。
- **確定**: CGI用locationはGET / POSTに限定し、DELETEは405にする。通常locationでDELETEを提供し、CGIファイルの削除へ進めない。
- **確定**: `/cgi/test.py/extra` 等の基本的な後続パスはPATH_INFOとして渡す。URLの分離・デコード・正規化はC-24・C-25に従う。symlinkの運用前提と制限はC-26に従う。スクリプト特定手順はCGI-25、CGIのファイルパスの結合・検証はCGI-84〜90で確定済み。
- **確定**: CGIは起動から10秒(出力による延長なし)、ヘッダー8 KiB、ヘッダーを含むstdout全体8 MiB。これらはコード内定数とし、追加のConfig項目は設けない。独自のCGI同時実行数上限も設けない。入力上限には既存の `client_max_body_size` を使う。
- **確定(C-21)**: `return` はlocation内に「301または302＋固定のHTTP(S)絶対URL」の2引数で指定する。URLは `http://` または `https://` で始まり、ホスト部分を持つものとする。相対URL、他のコード、引数不足・余分な引数、不正なURLは設定エラーで起動中止。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。設定したURLをLocationヘッダーでクライアントに返し、Webserv自身は移動先を取得しない。同じlocation内での複数回指定はC-14に従い拒否する。
- **確定(C-22)**: 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。禁止された対応メソッドは405とAllowヘッダーで応答し、リダイレクトしない。例えばreturnのみのlocationはGETなら指定した301/302、POST・DELETEなら405。これは本課題用の処理順序であり、未対応メソッドや不正なHTTP要求の扱いを新たに定めるものではない。

### locationの選択とパス

- **確定(C-36)**: HTTP/1.0・HTTP/1.1要求でContent-LengthもTransfer-Encodingもない場合は、POSTを含めてボディ長0として扱う。他のヘッダー検証でエラーがなければヘッダー終端で要求の受信を完了し、ボディや接続切断を待たない。長さ指定の省略だけを理由に411を返さない。Content-Length: 0も空ボディとし、正常な正のContent-Lengthは指定バイト数、HTTP/1.1の正常なchunkedは終端まで受信する。存在する不正な長さ指定やHTTP/1.0のTransfer-Encodingを省略扱いにはせず、既存のエラー規則を適用する。要求完成後の余剰データはボディや次の要求として処理しない既存方針を維持する。空ボディで受信完了しても処理成功を保証せず、メソッド許可や処理側の検証を行う。CGIへ進む場合はstdinにデータを書かず親側の書き込みパイプを閉じてEOFを渡す。
- **確定(C-35)**: HTTP/1.0・HTTP/1.1要求のContent-Lengthは最大1行とする。同じ値でも2行以上は400とし、名前の大小文字だけが異なる場合もC-34に従って重複と判定する。値にカンマを含む場合は、同値の一覧を含めて400とする。複数行の結合、先勝ち・後勝ち、同値の単一化による補正は行わない。不正な数値・桁あふれの400と、Transfer-Encoding併記時の400は既存方針を維持する。検出したエラーはボディの完了を待たず既存のエラー応答経路へ渡し、応答送信完了後に切断する。この決定は要求のContent-Lengthについてであり、他のヘッダーやCGI出力に一律適用しない。Content-LengthとTransfer-Encodingが両方ない場合はC-36に従う。
- **確定(C-34)**: HTTP/1.0・HTTP/1.1要求のヘッダー名はRFC 9110のtoken(ASCII英数字と ! # $ % & ' * + - . ^ _ ` | ~ のいずれか1文字以上)として検証し、ASCII大文字A〜Zを小文字a〜zに変換して保持する。名前の比較は大小文字を区別しない。空の名前・許可外の文字・名前と値を区切るコロンの欠落は補正せず400とする。ヘッダー行は最初のコロンで名前と値に分け、値内のコロンはそのまま扱う。値はC-33の前後SP/HTAB除去を行うが、一律の小文字化はしない。個別ヘッダーの値の検証は既存方針に従う。この決定はHTTP要求のヘッダー名についてのもので、重複はHostのC-28・C-30とContent-LengthのC-35に従い、その他の重複と個別値の検証規則はHTTP-06〜11に従う。CGI出力・HTTP_*転送とCGIの公開APIはCGI-39〜98で合意済み。HTTP全体の公開API/保持はHTTP-59〜64で合意済み。
- **確定(C-33)**: HTTP/1.0・HTTP/1.1要求のヘッダーは折り返し(obs-fold)を受け付けず、SP/HTABで始まる空でないヘッダー行は400とする。最初のヘッダー行の字下げやSP/HTABだけの行も同様に拒否する。フィールド名とコロンの間の空白・タブは400とし、補正しない。コロン後の値は前後のSP/HTABだけを除去し、値の内部の空白・タブはこの処理では変更しない。`Host:localhost` と `Host: localhost` は同じ値として扱う。ヘッダー終端は完全に空のCRLF行とする。この処理はHTTP要求ヘッダーに限り、ボディには適用しない。フィールドごとの値の検証(HostのC-30等)は別途行い、CGI出力の検証規則は変更しない。
- **確定(C-32)**: HTTP/1.0・HTTP/1.1要求のリクエスト行は末尾CRLF込みで8 KiB(8,192バイト)、その直後のヘッダー部分全体は各行のCRLFと終端の空行込みで32 KiB(32,768バイト)を上限とする。ヘッダー合計にリクエスト行・ボディを含めない。URLデコード等の前の受信バイト数で数え、上限と同じサイズは受理、超過時はそれぞれ414 / 431で拒否する。受信中に検査し、超過が確定したら行末・ヘッダー終端・ボディの到着を待たずエラー応答へ進み、既存方針どおり応答送信完了後に切断する。同じrecvで届いたボディをヘッダーサイズへ加算しない。ヘッダー1行ごとの別上限と個数上限は追加しない。2つの上限はコード内定数とし、Config項目を増やさない。ボディ上限は既存client_max_body_sizeと413、CGI出力上限はCGI-20の規則を維持する。
- **確定(C-31)**: HTTP/1.0・HTTP/1.1要求のリクエスト行とHTTPヘッダーの改行はCRLFだけを受理し、単独LF・不正な単独CRは400とする。受信断片の末尾がCRだけなら、その時点では不正とせず次のバイトを待つ。次がLFなら正常なCRLFとして扱い、LF以外なら不正と判定する。切断やタイムアウトで未完了になった場合は既存の接続処理に従う。この改行制限はボディのデータには適用せず、ボディ中のCR/LFを変更・拒否しない。CGI出力ヘッダーのLF/CRLF両方受理と、HTTP応答ヘッダーのCRLF生成はCGI-15の既決定を維持する。
- **確定(C-30)**: Host値は前後のSP/HTABを取り除いてから、IPv4またはASCIIホスト名と任意の `:port` として検証する。ホスト名はASCII英数字と `-` からなる空でない要素を `.` で区切り、各要素の先頭・末尾は英数字とする。localhostのような単一要素、大文字・小文字、末尾の `.` 1個を受理する。IPv4はドット区切りの4個の十進数で各0〜255。ポートは省略可、明示するなら十進数の1〜65535で空値は拒否する。Host全体の空値、途中の空白・タブ、重複、不正な値、スキーム・パス・ユーザー情報付きの値は400。IPv6の角括弧表記とHost内のパーセント表記は受付対象外として400にする。DNSへの問い合わせや名前の実在確認は行わず、Hostによる仮想ホスト選択も行わない。HTTP/1.0のHost省略可、HTTP/1.1の欠落400、自動補完URLでの使用方法はC-28を維持する。この受付範囲は本課題用の限定仕様であり、RFCのHost構文全体を実装するものではない。
- **確定(C-29)**: C-27の自動補完のLocationを生成するときは、デコード・正規化済みのパスに末尾 `/` を付け、ASCII英数字(A-Z / a-z / 0-9)・`- . _ ~` と区切りの `/` だけをそのまま出力する。それ以外はバイトごとに大文字16進の `%XX` へ変換する。例: 内部パス `/hello world` → `/hello%20world/`、`/a+b` → `/a%2Bb/`、`/name?part` → `/name%3Fpart/`、`/literal%20` → `/literal%2520/`。URLとして既にエンコード済みの文字列へ重ねて適用せず、内部のデコード済みパスから一度生成する。queryは元の表記を保持して後ろに付け、デコードも再エンコードも行わない。例えばHost localhost:8080への `/hello%20world?name=a+b` は `http://localhost:8080/hello%20world/?name=a+b` へ案内する。Configのreturnの固定URLはこの変換の対象にしない。
- **確定(C-28)**: C-27の自動補完のLocationは `http://` で始まる絶対URLとする。有効なHostがあればホスト名と明示ポートを使い、HostにポートがなければURLにも追加しない(HTTPの既定80番)。HTTP/1.0でHostがない場合だけ、acceptした接続のサーバー側IP・ポートをgetsocknameで取得して使う。listenのワイルドカード値0.0.0.0やクライアント側IP・ポートを代用しない。HTTP/1.0・1.1ともHostの重複・不正は400、HTTP/1.1のHost欠落も400。tasugiyaが接続のサーバー側IP・ポートを取得して渡し、rysatoがHostの検証とURL組み立てを担当する。Forwarded / X-Forwarded-*を参照せず、ConfigのreturnはC-21の固定URLを使う。パスの再エンコードはC-29に従う。Hostの受付範囲はC-30に従う。接続情報の受け渡しAPIはCGI-93のConnectionInfoで合意済み。取得失敗時の扱いは、残るHTTP/ネットワーク境界の設計で明確にする。
- **確定(C-27)**: GETでlocationを選択し許可メソッドを確認した後、選択した配信locationのroot/aliasで解決した対象が実ディレクトリで、正規化後のURLパスが `/` で終わらなければ、末尾 `/` 付きURLへ301を返す。queryは保持し、例えば `/docs?lang=ja` の移動先のパスとqueryは `/docs/?lang=ja` とする。末尾 `/` 付きの要求でC-15のindex→未発見ならautoindexの処理へ進む。POST・DELETEにはこの自動補完を適用せず、各メソッドの処理規則に従う。returnがあるlocationはC-21・C-22の指定を使う。location選択は変更せず、`location /docs/` を `/docs` に一致させる特別扱いは追加しない。Locationのホスト名・ポートの出所はC-28に従う。パスの再エンコードとqueryの保持はC-29に従う。
- **確定(C-26)**: 公開・CGI・アップロード領域は二人が管理し、配下にsymlinkを置かない運用前提とする。Webservにはsymlinkの検出・拒否機能を実装せず、関連するConfig項目も増やさない。symlinkが置かれた場合はOSがリンク先を辿り得るため、公開領域外へのアクセスを完全に遮断する仕様とはしない。URL由来のパスにはC-24・C-25のデコード・正規化を適用し、その処理だけでsymlink経由の領域外アクセスを防げるとは扱わない。CGIのファイル存在・種別・アクセス失敗とroot/aliasのパス結合はCGI-84〜90で確定済み。通常静的配信・一覧生成はHTTP-20〜32で合意済み。アップロード名・保存はHTTP-33〜44で合意済み。DELETE/途中ファイルの後始末はstd::removeによる削除を含めHTTP-45〜50で合意済み。
- **確定(C-25)**: C-24で1回デコードしたURLパスをlocation選択前に正規化する。連続する `/` は1個にまとめ、`/` で区切った要素全体が `.` なら除去、`..` なら直前の要素を1個取り除く。URLの先頭の `/` より上へ戻ろうとした時点で400とする。`file..txt`・`.hidden` 等は変更しない。末尾 `/` と、末尾 `/.`・`/..` が表すディレクトリの末尾 `/` を保つ。例: `/a//b.txt`・`/a/./b.txt`・`/a/x/../b.txt` は `/a/b.txt`、`/a/../b.txt` は `/b.txt`、`/a/.`・`/a/b/..` は `/a/`、`/a/..` は `/`。`/../secret.txt`・`/a/../../secret.txt` は400。`/a/%2e%2e/b.txt` はデコード後に `/b.txt` としてlocationを選ぶ。この処理はURL文字列の整理であり、symlinkの扱いと公開範囲の保証の制限はC-26に従う。ファイルパスの結合・検証の残る詳細は別途確定する。末尾 `/` のない実ディレクトリへのGETはC-27に従う。
- **確定(C-24)**: 最初のリテラルの `?` でパスとqueryを分離し、パスの `%XX` を1回だけデコードする。その後にパスを正規化してからlocationを選ぶ。queryは未デコードのままCGIのQUERY_STRINGへ渡す。パスでは `%20` は空白、`%2F` / `%2f` は区切りの `/` となり、`+` は空白へ変換しない。`%252e` は `%2e` までとし、HTTP/Router/CGIへの受け渡しで二重デコードしない。復号して生じた `?` をqueryの区切りとして再解釈しない。パスにある `%` / `%2` / `%GG` 等の不正なパーセント表記と `%00` (NUL) は400とする。パスの400規則はC-24を維持する。queryの構文検査はHTTP-05で合意済みで、デコードせず保持する。`.` / `..` / 連続スラッシュの正規化はC-25に従い、symlinkの運用前提と制限はC-26に従う。その他の不正パスとファイルパスの結合・検証の詳細は別途確定する。
- **確定(C-23)**: queryを除いたURLパスに対する文字列前方一致とし、一致する最も長いprefixを選ぶ。設定順には依存せず、パス要素の境界は一致条件にしない。`/a` は `/a`・`/a/file.txt`・`/abc` に一致する。ディレクトリ配下を指定するには `/a/` と書き、これは `/a/file.txt` には一致するが `/a`・`/abc` には一致しない。`/`・`/a`・`/a/b/` があれば、`/a/b/file.txt` には `/a/b/`、`/abc` には `/a` を選ぶ。URIの分離・デコード・処理順序はC-24で確定。正規化はC-25で確定。末尾スラッシュ補完リダイレクトはC-27に従う。
- **確定(C-38)**: Configのlocation prefixは先頭`/`を必須とし、ASCII英数字(A-Z / a-z / 0-9)と `/ - . _ ~` だけを受理する。連続する`//`、パス要素全体が`.`または`..`の指定は設定エラーで起動を中止する。queryや%エンコード、非ASCII文字など許可外の文字を受け付けず、設定値をデコード・正規化して救済しない。`/`・`/images/`・`/cgi-bin/`・`/v1.0/`・`/.hidden/`・`/file..txt`は受理する。prefixの比較はOSによらず大小文字を区別し、`/Images/`と`/images/`は別のprefixとする。`location /`は推奨するが必須ではなく、一致するlocationがなければ404。各serverにlocationを1個以上要求するC-37は維持する。alias使用時のprefix末尾`/`必須(C-09)も維持し、root使用時は末尾`/`を必須にしない。この文字制限はConfigのprefixだけに適用し、HTTP要求URLやファイル名の受付範囲には一律適用しない。要求側はC-24・C-25のデコード・正規化後にC-23の最長前方一致で照合する。
- **確定(C-38)**: locationの設定値の受付は上記に従う。要求URIの分離・デコード・正規化はHTTP/Router側の責務とし、処理順序はC-24に従う。正規化はC-25に従い、symlinkの運用前提と公開範囲の保証の制限はC-26に従う。ファイルパスの結合・検証の残る詳細は別途合意する。
- `root` は設定ディレクトリにURLパス全体を追加する。`alias` は一致したlocationのprefixを設定ディレクトリへ置き換える。
- **確定(C-09)**: aliasはディレクトリ対応だけを扱い、**locationのprefixとaliasの値の両方を `/` で終える**。どちらかの末尾 `/` が欠けていたら、自動補完せず設定エラーとして起動を中止する。正規表現や単一ファイルへのaliasは扱わない。この末尾制限はaliasを使うlocationに適用し、rootを使うlocationには追加しない。要求URLの末尾 `/` 補完リダイレクトはC-27に従う。

| 設定 | 要求パス | 探すファイル |
|---|---|---|
| `location /images/` + `root /data;` | `/images/a.png` | `/data/images/a.png` |
| `location /images/` + `alias /data/pictures/;` | `/images/a.png` | `/data/pictures/a.png` |
| `location /kapouet/` + `alias /tmp/www/;` | `/kapouet/pouic/toto/pouet` | `/tmp/www/pouic/toto/pouet` |

最後の例で課題PDFのprefix置換を満たす。`root` にaliasの意味を持たせない。`/images/` は `/images` に一致しない。C-27は選択済みの配信locationで実ディレクトリと判定できたGETだけに適用し、別のlocationを末尾 `/` の補完によって探索する処理は追加しない。

- **確定**: ファイルシステムの相対パスは**設定ファイルを置いたディレクトリ基準**。例えば `conf/default.conf` 内の `root ../www/site1;` は `www/site1` を指す。URL-prefix・returnのURL・indexのファイル名にはこの基準を適用しない。
- **確定(C-40)**: 起動時は、配信locationのroot/aliasと有効なupload_storeについて、パス解決・既定値補完後にstatで存在とディレクトリ種別を確認し、確認失敗・非ディレクトリなら設定エラーで起動を中止する。root省略時のhtmlにも適用し、conf/default.confならconf/htmlが必要になる。ディレクトリや親ディレクトリは自動作成しない。ディレクトリの読み取り・書き込み等の権限を追加のaccess検査で事前確認せず、実際の配信・一覧取得・保存時に失敗を処理する。有効なCGI設定のPythonはstatで存在する通常ファイルであること、access(path, X_OK)の事前確認が成功することを要求し、失敗したら起動エラーとする。起動時にPythonを試験実行しない。indexの存在は起動条件にせず、error_pageの不在・読み取り不可も起動エラーにしない。これらは要求時に既存のindex/autoindex・内蔵エラーページの規則で処理する。個々のHTML・画像・CGIスクリプトを起動時に全件検査しない。リダイレクト専用locationはC-22に従い配信設定を補完・検証しない。起動時検査は実行時の成功を保証するものではなく、ファイル消失・権限変更やexecve等の失敗は引き続き要求時に処理する。symlinkの運用前提と検出機能を省くC-26は維持する。
- 起動後のファイル消失や権限変更は各要求のエラーとして処理する。起動時の検証だけで実行時の確認を省略しない。

### 設定例

以下は、Config・CGIの確定事項に沿った例。実装用設定と対応するデモファイルは別途用意する。以下の例は既定値も一部明示しているが、NGINX既定値のある項目の記述を必須とする意味ではない。

`conf/default.conf` に置く想定。ディレクトリ・HTML・CGI等のデモファイルは別途用意する。この例は本課題用であり、そのままNGINXに読み込ませる設定ではない。

```nginx
server {
    listen 127.0.0.1:8080;
    client_max_body_size 1048576;
    error_page 404 ../www/errors/404.html;

    location / {
        allow_methods GET;
        root ../www/site1;
        index index.html;
        autoindex off;
        upload_store off;
        cgi_extension off;
    }

    location /files/ {
        allow_methods GET POST DELETE;
        alias ../uploads/;
        index index.html;
        autoindex on;
        upload_store ../uploads;
        cgi_extension off;
    }

    location /cgi/ {
        allow_methods GET POST;
        alias ../cgi-bin/;
        index index.py;
        autoindex off;
        upload_store off;
        cgi_extension .py /usr/bin/python3;
    }

    location /old/ {
        allow_methods GET;
        return 301 http://127.0.0.1:8080/;
    }
}

server {
    listen 127.0.0.1:8081;
    client_max_body_size 1048576;

    location / {
        allow_methods GET;
        root ../www/site2;
        index index.html;
        autoindex off;
        upload_store off;
        cgi_extension off;
    }
}
```

<a id="config-api"></a>

### tasugiyaとrysatoの間の契約と検証

**確定(C-42)**: ConfigはConfig / ServerConfig / LocationConfigの3つの読み取り専用データクラスで保持する。rysatoのConfigParserが設定ファイルを読み、構文解析・既定値補完・設定ファイル基準のパス解決・C-40/C-41等の検証まで完了してから、Configを値で返す。tasugiyaは完成した値を使用し、設定文字列の再解析や既定値の再計算を行わない。

| 保持先 | データ | 型・表現 |
|---|---|---|
| Config | servers | std::vector<ServerConfig> |
| ServerConfig | host / port / clientMaxBodySize | std::string / int / std::size_t |
| ServerConfig | errorPages / locations | std::map<int, std::string> / std::vector<LocationConfig> |
| LocationConfig | path / allowedMethods | std::string / std::set<std::string> |
| LocationConfig | kind / pathMode | enum Kind { SERVE, REDIRECT } / enum PathMode { ROOT, ALIAS } |
| LocationConfig | basePath / index / autoindex | std::string / std::string / bool |
| LocationConfig | uploadPath | std::string。空文字列は無効 |
| LocationConfig | cgiExtension / cgiInterpreter | std::stringを2つ。1組だけ保持し、無効なら両方空。mapは使用しない |
| LocationConfig | redirectCode / redirectUrl | int / std::string。REDIRECTのとき使用 |

- ポートとステータスコードはintに変換する際に各既決定の範囲を検証する。ボディ上限はstd::size_tで保持し、範囲を超える値は設定エラーとする(C-13)。文字列から数値への変換で桁あふれを起こしてから判定しない。
- 設定値はconstのgetterで公開し、公開setterは設けない。文字列・コンテナはconst参照、int / std::size_t / bool / enumは値で取得する。getterの名前はgetServers / getHost / getPort / getClientMaxBodySize / getErrorPages / getLocations / getPath / getAllowedMethods / getKind / getPathMode / getBasePath / getIndex / getAutoindex / getUploadPath / getCgiExtension / getCgiInterpreter / getRedirectCode / getRedirectUrlとする。
- kindを確認して該当する設定だけを使う。REDIRECTでは配信項目を使用せず、配信既定値の補完・配信パス検証もしない(C-22)。解析中は明示の有無を保持して重複や禁止された併記を検出し、検証後の設定だけを公開する。
- 設定ファイルの読み込み・構文解析・検証の失敗はstd::runtime_errorで理由を伝える。mainが捕捉して理由を表示し、listenを始めず異常終了する。ConfigParser内部でプロセスを終了させず、不完全なConfigを返さない。これは起動時のみ例外を許可する既決定に従う。
- mainがconst Configを保持し、EventLoopはconst Config&、ServerSocket/Clientは対応するconst ServerConfig&またはconst ServerConfig*を参照する。設定の所有権は移さず、参照先をdeleteしない。Configとそのvectorの要素は起動後に変更せず、EventLoop・Client等を先に終了/破棄し、Configを最後まで生存させる。
- tasugiyaはServerConfigごとにlistenし、accept元に対応するServerConfigをClientに紐付ける。body上限は最初のappendDataより前にHttpRequestへ渡す。bind等の起動失敗では作成済みfdも閉じる。
- エラーメッセージには設定項目名と、分かる場合はserverのlisten先またはlocationを含める。行番号は実装しない既決定を維持する。

公開APIの中心は次のとおり。メンバー構築用のprivate関数等は実装詳細とする。ConfigParserは1クラスで、字句解析と構文解析をprivate関数に分ける既決定を維持する。

```cpp
class ConfigParser {
public:
    Config parse(const std::string& configPath);
};

// Configのgetter
// const std::vector<ServerConfig>& getServers() const;
// ServerConfigのgetterの例
// const std::string& getHost() const;
// int getPort() const;
// std::size_t getClientMaxBodySize() const;
// const std::vector<LocationConfig>& getLocations() const;
```

```cpp
// main内の起動処理。呼び出し元でstd::runtime_errorを捕捉する。
ConfigParser parser;
const Config config = parser.parse(configPath);
EventLoop* loop = new EventLoop(config);
loop->run();
if (CgiExecutor::isChildProcess()) {
    loop->cleanupInChild();
    delete loop;
    return 127;
}
delete loop;
// 親はrun内で停止・回収を完了してからdeleteする。
// configは参照元の破棄後まで生存する。
// 子の終了・解放経路はCGI-120〜127の現時点の採用案。
```

二人で共有する確認例は、既定値補完後の型・値、C-40/C-41の検証、不正設定でruntime_errorとなりlistenへ進まないこと、upload/CGI無効時の空文字列、getterが書き換え可能な参照を公開しないこと、Configの寿命が参照元より長いこと。HTTP/Routerの残るAPIはC-42とは別議題。CGIの受け渡しAPIはCGI-91〜98、エラー適用境界はCGI-128〜135で合意済み。

### 合意状況と残る確認事項

ConfigのC-01〜42は合意済み。最後の6項目は文法(C-37)、location設定値(C-38)、listen省略値(C-39)、起動時検証(C-40)、設定の組み合わせ(C-41)、型とAPI(C-42)。実装・試験の合格を示すものではない。

通常静的配信・一覧生成・アップロードはHTTP-20〜44で合意済み。途中ファイル削除・DELETEはstd::removeによる削除を含めHTTP-45〜50で合意済み。応答生成はHTTP-51〜58で合意済み(Date省略は暫定案)。HTTPの公開API/データの保持はHTTP-59〜64で合意済み。ネットワーク境界・計時・通常ファイルI/OはHTTP-65〜74で合意済み(採用案を含む)。正本は [Requirements](../Requirements.md#http-network-boundary)、Dateの暫定案・接続情報取得失敗時の扱い・採用案の検証は [04](04_testing_and_workplan.md#remaining-design) で管理する。CGIの受け渡し・出力検証は合意済みであり、ConfigParserへ移さない。正常なCGIの400〜599にもC-20の本文置換を適用する(CGI-130〜133)。CGIの起動・計時方法は現時点の採用案で、実際の動作で見直し得る。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト

- [RFC 9112 §5.1–5.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-5.1): コロン直前の空白拒否、値の前後OWSの除去、obs-foldの扱い。C-33では折り返しを補正せず400にする。
- [RFC 9110 §5.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1)、[§5.6.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.2): ヘッダー名の大小無視とtokenの文字集合。C-34では名前をASCII小文字に揃えて保持する。
- [RFC 9110 §8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6): 同値を繰り返したContent-Lengthは拒否または単一値への補正が可能。C-35では拒否を選択する。
- [RFC 9112 §6.3](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3)、[RFC 1945 §7.2](https://www.rfc-editor.org/rfc/rfc1945.html#section-7.2): 要求ボディの長さ判定。C-36では長さ指定が両方ない要求を空ボディとして扱う。
