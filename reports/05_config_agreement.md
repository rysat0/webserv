# 05 Configの設計案と合意事項

> 記録日: 2026-10-04。対象: webservの設定ファイル。作成時のブランチ: `research`。

NGINXを基本に、課題要件を満たす範囲でConfigの文法・設定項目・検証を小さくするための資料。Configのパス規則・限定文法と、CGI関連の設定方針を確定事項として記録する。

## Configの確定事項

| ID | 決定 | 具体例・動作 |
|---|---|---|
| C-01 | **rootとaliasを区別する** | rootはディレクトリにURI全体を追加する。aliasはlocationのprefixを指定ディレクトリへ置き換える。課題のprefix置換例はaliasで実現する |
| C-02 | **error_pageはファイルを直接指定する** | NGINXのURI指定・内部リダイレクトは行わない。指定ファイルを本文として読み、未指定・読み取り失敗時は内蔵エラーページを返す |
| C-03 | **相対パスは設定ファイルのディレクトリ基準** | `conf/default.conf` 内の `root ../www/site1;` はプロジェクトの `www/site1` を指す |
| C-04 | **文法を5点に絞る** | ファイル直下にserver、server内に設定とlocationを置く。設定は`;`、ブロックは`{}`、空白・タブ・改行を区切りとし、`#`から行末をコメントとする。引用符・エスケープ・変数・include・正規表現・locationの入れ子・reloadとhttp/eventsの外枠を省く。未対応の書き方・文法間違いは理由を表示して起動を中止する |
| C-05 | **省略時はNGINXの既定値を適用する** | 対応する設定項目にNGINXの既定値があれば採用し、省略自体をエラーにしない。明示offを強制しない。独自項目の省略値はNGINXの値と区別して決める。継承なし・相対パスの基準は維持する |
| C-06 | **allow_methods省略時はGETのみ許可** | POST・DELETEは必要なlocationで明示する。明示した場合はそのリストを使う。CGI用locationではGET/POSTの範囲に限定し、DELETEは405とする既決定を維持する |

C-01はNGINXのroot / aliasと同じ意味。C-02は実装を簡略化するための意図的な差異。C-03はこのプロジェクトで採用するパス解決の基準である。

## CGIに関する追加の確定事項

- 必須部分はPythonのみ。`cgi_extension .py /絶対パス/python3` で指定したPythonを `execve` で起動し、選択されたlocationのCGI設定と拡張子で対象を判定する。
- CGI用locationはGET/POSTに限定する。DELETEは405とし、スクリプトの削除へ進めない。
- 設定されたindexとして選ばれた `index.py` にもCGI判定を適用する。基本的な後続パス(`/cgi/test.py/extra`)をPATH_INFOとして渡す。
- アップロードとCGIはlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にアップロードさせない。
- CGIの10秒制限、ヘッダー8 KiB・stdout全体8 MiBはコード内定数とし、Config項目を増やさない。CGI同時実行数の独自上限・件数による503・待ち行列は設けない。

CGIの出力・エラー処理を含む全決定は [Requirements.md](../Requirements.md) と [03のC12](03_config_cgi_upload.md) に記載する。

## 既存資料との関係

- [Requirements.md](../Requirements.md)の既決定である「NGINX風の名前と独自名の併用」「継承なし」「省略時はNGINXの既定値を適用」「最長前方一致」「必須部分のCGIはPythonのみ」を前提とする。
- [03 設定・CGI・アップロード](03_config_cgi_upload.md)は調査論点とCGIの確定事項・残る合意を記録する。本書はConfigの具体案と合意状況を記録する。
- RequirementsのConfig節と本書の詳細案・合意状況を揃える。確定事項以外の文法・検証等まで一括して承認済みとは扱わない。

## 設定の詳細案

以下は確定事項と詳細案をまとめたもの。限定文法の5点はC-04で確定済み。省略時の既定値はC-05に従う。独自項目の省略値、指定回数、残る文法の細則・追加の検証条件は、確定と記したものを除き二人で確認するための案として読む。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- **確定**: 省略された設定項目には下表のNGINX既定値を適用し、明示offを強制しない。明示された値を優先する。独自項目には対応するNGINX既定値がないため、別途省略値を決める。構文不正・不正な値をデフォルトで置き換えて受理する意味ではない。
- **確定**: `error_page` 未指定・読み取り失敗時は内蔵エラーページを使う。
- **案**: リダイレクト専用locationには配信設定を要求せず、配信項目のデフォルト補完・配信パスの検証も行わない。明示された配信設定との併記は拒否する。
- 1つのserverにつきlisten先は1組。複数のserverを並べて複数ポート・複数サイトに対応する。Hostによる仮想ホスト選択は行わない。
- **確定**: NGINXの `root` と `alias` は意味を分けて採用する。**案**: 同じlocationには片方だけを指定する。

### 省略時の既定値

C-05により、既定値のある設定を省略したことだけでは起動エラーにしない。以下は本課題で扱う設定項目に限った表であり、未対応のNGINXディレクティブや継承機構を追加する意味ではない。

| 項目 | 省略時の値・動作 | 位置付け |
|---|---|---|
| `listen` | NGINXの規則に従い、特権起動時は全IPv4アドレスの80番、その他は8000番 | 省略可。起動権限を判定する実装方法は許可関数と照合して確定する。bind失敗時に別ポートへ切り替える規則ではない |
| `client_max_body_size` | 1 MiB = 1,048,576バイト | NGINXの`1m`を値として採用。設定ファイルで単位接尾辞を受理するかは別の文法の論点 |
| `root` | `html` | 配信locationでroot/aliasの両方がなければ適用。aliasが明示されている場合は補完しない |
| `alias` | 指定なし | 勝手にaliasの対象を作らない |
| `index` | `index.html` | `index.py`は自動選択しない。CGIのindexを使う場合は明示する |
| `autoindex` | `off` | 明示offと省略は同じ動作 |
| `error_page` | 個別ファイル指定なし。内蔵エラーページを使う | C-02を維持 |
| `return` | リダイレクトなし | 通常の配信locationとして扱う案 |
| `upload_store`(独自) | **案**: `off` | NGINX既定値ではない。省略時はアップロード保存を無効にする案 |
| `cgi_extension`(独自) | **案**: `off` | NGINX既定値ではない。Pythonのパスを推測しない |
| `allow_methods`(独自) | **確定**: GETのみ許可 | C-06のチーム独自の既定値。POST・DELETEは必要なlocationで明示する。CGI用locationのGET/POST限定・DELETEへの405は維持する |

既定の`root html;`にもC-03を適用する。例えば`conf/default.conf`では`conf/html`を指す。NGINXのインストール先等のパスをそのまま使うわけではない。既定値の補完と上位設定の継承は別であり、継承なしの方針は維持する。必須のブロック構造や引数不足・重複・不正値の検証も、設定項目の省略とは区別する。

根拠: [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)。

### 文法

基本文法はC-04の5点で確定済み。以下では、必須個数など今回の合意に含まれない細則を案として区別する。

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブとlocationブロック }
location    := location URL-prefix { location用ディレクティブ }
単純設定    := 名前 引数... ;
```

- **確定**: 最上位には `server` だけを書き、server内に設定とlocationを置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。**案**: ファイルにはserverを1個以上、各serverにはlocationを1個以上置く。
- **確定**: 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。単純設定は`;`で終える。改行そのものは設定の終わりではなく、`;`を省略できない。ブロックは`{}`で囲む。**案**: ブロック末尾の `}` に `;` は付けない。
- **確定**: `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を使った未対応の書き方は設定エラーとする。
- **案**: 名前・メソッド・`on` / `off` は大文字小文字を区別する。ディレクティブの記述順には依存しない。
- **確定**: 未対応の書き方・文法間違いは理由を表示して起動を中止する。未知の名前や未閉じブロックを無視・補正して起動しない。**案**: 各項目の階層・引数数・重複禁止規則は下表と検証条件を確認して確定する。
- **確定**: `include`、正規表現location、設定reloadは実装対象外。設定変更後は再起動する。**案**: `=` / `^~` / `~` 等の修飾子、名前付きlocation、`if` / `rewrite` / `try_files` も扱わない。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | 省略可・明示は最大1回の案 | IPv4アドレスと1〜65535のポートを明示。`0.0.0.0`も可。ポートだけの指定、ホスト名、IPv6、追加オプションは扱わない |
| `client_max_body_size` | `client_max_body_size 1048576;` | 省略可・明示は最大1回の案 | 正の十進整数、単位はバイト。`0`、符号、`1m`等の単位、size_tの範囲を超える値は拒否。復号後ボディに適用する |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可 | 1行に400〜599のコード1個とファイルパス1個。同一コードの重複は禁止。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | 必須・1個以上 | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させない |

- 同じIPv4:portの重複は起動エラー。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。異なる特定IPv4アドレス同士は同じポートを指定できる。
- `error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- `server_name` は実装しない。接続を受け付けたlisten先のServerConfigを使用する。

### locationの設定

locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 省略可・既定GETのみ。明示は最大1回の案 | 同左 |
| `root` | `root ../www/site1;` | 省略可。root/aliasとも未指定なら`html` | 指定不可(案) |
| `alias` | `alias ../uploads/;` | 省略可。明示rootとの併記は禁止の案 | 指定不可(案) |
| `index` | `index index.html;` | 省略可・既定`index.html`。明示は最大1回の案 | 指定不可(案) |
| `autoindex` | `autoindex off;` | 省略可・既定off。明示は最大1回の案 | 指定不可(案) |
| `upload_store`（独自） | `upload_store off;` または `upload_store ../uploads;` | **案**: 省略時off、明示は最大1回 | 指定不可(案) |
| `cgi_extension`（独自） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;` | **案**: 省略時off、明示は最大1回 | 指定不可(案) |
| `return` | `return 301 http://127.0.0.1:8080/;` | 指定不可 | 必須・1回 |

- `allow_methods` はGET / POST / DELETEから1個以上、重複なし。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止されたメソッドは405とAllowヘッダーで応答する。
- **確定**: `allow_methods` を省略したlocationはGETのみ許可する。POSTを受けるCGI・アップロード用locationには、例えば `allow_methods GET POST;` を明示する。通常locationでDELETEも提供するなら `allow_methods GET POST DELETE;` 等を明示する。許可メソッドの設定だけでCGIやアップロード保存が有効になるわけではない。
- `index` は単一ファイル名のみ。`/`を含むパスや `.` / `..` は拒否する。指定ファイルがない場合はautoindexへ進み、`off`なら403。indexファイルの存在自体は起動条件にしない。
- **確定**: 設定されたindexとして `index.py` が選ばれた場合も、locationのCGI設定と拡張子で判定して実行する。index省略時は`index.html`を使い、`index.py`の自動補完は行わない。
- `autoindex` は `on` / `off` のみ。出力形式はHTMLに固定する。
- `upload_store off` はサーバーによるアップロード保存を無効にする。パス指定時はそのディレクトリへ保存する。GETで公開するには `root` / `alias` も保存先に対応させる。
- **確定**: `cgi_extension` で拡張子とPythonの絶対パスを指定する。必須部分はPythonのみ。locationのCGI設定と拡張子で実行対象を判定し、通常ファイルのスクリプトを `execve` で起動したPythonに渡す。FastCGI設定は扱わない。
- **案**: `cgi_extension` は `off`、または拡張子とインタプリタの組1個。拡張子は `.py` に固定せず、`.` と1文字以上の英数字からなる値を受理し、大文字小文字を区別する。受理する拡張子の文法と組数は未確定。
- **確定**: アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける。アップロード先でCGIを実行せず、CGI実行場所にはアップロードさせない。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- **案**: 配信locationでPOSTを許可するなら、アップロード保存またはCGIのどちらかを有効にする。
- **確定**: CGI用locationはGET / POSTに限定し、DELETEは405にする。通常locationでDELETEを提供し、CGIファイルの削除へ進めない。
- **確定**: `/cgi/test.py/extra` 等の基本的な後続パスはPATH_INFOとして渡す。スクリプトを特定する手順やパス正規化の詳細は別途合意する。
- **確定**: CGIは起動から10秒(出力による延長なし)、ヘッダー8 KiB、ヘッダーを含むstdout全体8 MiB。これらはコード内定数とし、追加のConfig項目は設けない。独自のCGI同時実行数上限も設けない。入力上限には既存の `client_max_body_size` を使う。
- `return` は301 / 302と `http://` または `https://` で始まる固定URLのみ。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。
- 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。これは本課題用の処理順序とする。

### locationの選択とパス

- **最長前方一致は確定**。詳細案は、queryを除いたURLパスに対する文字列前方一致とし、設定順には依存しない。例えば `/a` は `/abc` にも一致する。ディレクトリ単位の設定には `/a/` を使う。パス境界を考慮するかは、この案を二人で確認して確定する。
- `/` は全パスに一致する通常のlocation。定義は推奨するが必須ではなく、一致するlocationがなければ404。
- locationのprefixにはquery、`%`によるエンコード、`.` / `..` のパス要素を記述しない。要求URIのデコード・正規化・公開範囲外へのアクセス防止はHTTP/Router側の責務とし、詳細は別途合意する。
- `root` は設定ディレクトリにURLパス全体を追加する。`alias` は一致したlocationのprefixを設定ディレクトリへ置き換える。
- aliasはディレクトリ対応だけを扱い、**locationのprefixとaliasの値の両方を `/` で終える**。正規表現や単一ファイルへのaliasは扱わない。

| 設定 | 要求パス | 探すファイル |
|---|---|---|
| `location /images/` + `root /data;` | `/images/a.png` | `/data/images/a.png` |
| `location /images/` + `alias /data/pictures/;` | `/images/a.png` | `/data/pictures/a.png` |
| `location /kapouet/` + `alias /tmp/www/;` | `/kapouet/pouic/toto/pouet` | `/tmp/www/pouic/toto/pouet` |

最後の例で課題PDFのprefix置換を満たす。`root` にaliasの意味を持たせない。`/images/` は `/images` に一致しないため、末尾 `/` なしも受けたい場合のリダイレクト方針は別途HTTP側で決める。

- **確定**: ファイルシステムの相対パスは**設定ファイルを置いたディレクトリ基準**。例えば `conf/default.conf` 内の `root ../www/site1;` は `www/site1` を指す。URL-prefix・returnのURL・indexのファイル名にはこの基準を適用しない。
- root / alias / upload_storeの対象ディレクトリは起動前に用意する。ConfigParserは存在・ディレクトリ種別を検証し、自動作成しない。CGIインタプリタも通常ファイル・実行可能であることを確認する。
- 起動後のファイル消失や権限変更は各要求のエラーとして処理する。起動時の検証だけで実行時の確認を省略しない。

### 設定例

以下は、Config・CGIの確定事項と残りの詳細案を組み合わせた例。独自項目の省略値・検証詳細の最終確定後に実装用設定へ反映する。以下の例は既定値も一部明示しているが、NGINX既定値のある項目の記述を必須とする意味ではない。

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

### A/B間の契約と検証

- BのConfigParserが全設定を解析し、省略された設定の既定値を補完・パス解決・検証してから、不変のConfigをAへ渡す。1つでも設定エラーがあれば設定全体を不採用とし、listenを開始しない。
- AはServerConfigごとにlistenし、Clientにその設定を紐付ける。body上限は最初のappendDataより前にHttpRequestへ設定する。bind等の失敗で起動を中止する場合は、作成済みfdも閉じる。
- LocationConfigのデータ表現案は、共通の `path` / `allowedMethods` と、配信かリダイレクトかの種別を持つ。配信側は `pathMode(ROOT/ALIAS)` / `basePath` / `index` / `autoindex` / `uploadPath` / CGI設定、リダイレクト側はcode / URLを持つ。rootとaliasを区別する動作は確定済みだが、フィールド名や型は未確定。
- `upload_store off` は空のuploadPath、`cgi_extension off` は空のcgiExtensionsとして保持できる。解析時は明示の有無を保持し、明示値を優先して省略分だけを補完する。root/alias・returnの選択後に、そのlocationで使う設定だけを補完・検証する案。独自項目の省略時offを採用した場合、完成後のConfigでは省略と明示offを同じ値にできる。
- 文法エラーは既決定どおり、理由を表示して起動を中止する。行番号は実装しない。設定項目名と、分かる場合はserverのlisten先またはlocationをメッセージに含める。
- 二人で共有する確認例: 上記設定の受理、省略時の既定値、明示値による上書き、引数不足、未知設定、階層違い、root/alias併記、同一listen先、数値範囲外、redirect専用location、error_page未指定時の内蔵ページ、root/aliasの対応例。

### 合意状況と残る確認事項

チェック済みは今回の会話で確定した内容。未チェックは、先ほどの提案として残している詳細であり、すべてが承認済みという意味ではない。

| 状態 | 内容 |
|---|---|
| [x] | rootはURI全体追加、aliasはlocationのprefix置換として、NGINXと同じ意味で区別する |
| [x] | error_pageはエラー本文用のファイルパスを直接指定する。未指定・読み取り失敗時は内蔵ページを使う |
| [x] | 相対ファイルパスの基準を設定ファイルのディレクトリに統一する |
| [x] | cgi_extensionで指定したPythonの絶対パスを使い、locationのCGI設定と拡張子で起動を判定する。必須部分はPythonのみ |
| [x] | CGI用locationはGET/POST。DELETEは405。設定されたindex.pyにもCGI判定を適用し、基本的なPATH_INFOに対応する |
| [x] | アップロード保存とCGI実行はlocation・ディスク上の保存先を分ける |
| [x] | CGIの時間・出力上限はコード内定数とし、Configを増やさない。CGI同時実行数の独自上限は設けない |
| [x] | C-04の限定文法5点を採用する。server/location・`;`・`{}`・空白区切り・`#`コメントを使い、引用符・エスケープ・変数・include・正規表現・locationの入れ子・reload・http/eventsの外枠を省く。文法間違いは起動エラー |
| [ ] | 残る文法の細則(ブロック個数、ブロック後の`;`、名前の大小区別・記述順、location修飾子等の扱い)を確定する |
| [x] | 省略された設定にはNGINXの既定値を適用し、明示offを強制しない(C-05)。継承なし・相対パスの基準は維持する |
| [x] | allow_methods省略時はGETのみ許可する(C-06)。POST・DELETEは必要なlocationで明示する。CGI用locationへのDELETEは引き続き405 |
| [ ] | upload_store / cgi_extensionの省略時off案、listen省略時の権限判定方法、指定回数とリダイレクト専用locationの条件を確定する |
| [ ] | root/aliasを排他にし、aliasはprefixと値が末尾 `/` のディレクトリ対応に限定する |
| [ ] | listenはIPv4:portを1serverに1組、body上限は正のバイト数に限定する。0による無制限指定は採用しない |
| [ ] | allow_methodsの検証詳細、upload_storeの独自設定名・書式と、cgi_extensionの拡張子文法・指定できる組数を確定する |
| [ ] | locationの前方一致でパス境界を考慮するか、上記の文字列前方一致案を確認する |
| [ ] | returnを301 / 302と固定の絶対URLに限定する |
| [ ] | 重複・数値範囲・パス存在等の起動時検証と、Configのデータ表現・A/B間のAPIを確定する |

アップロード形式・ファイル名・上書き、URI正規化・symlinkはHTTP側の別議題として残す。CGI出力の基本動作は確定済みで、ヘッダー検証等の残る詳細は [03のC12](03_config_cgi_upload.md) に記録する。ConfigParserにそれらの処理を持たせない。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト
