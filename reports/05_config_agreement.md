# 05 Configの設計案と合意事項

> 記録日: 2026-10-04。対象: webservの設定ファイル。記録先ブランチ: `research`。

NGINXを基本に、課題要件を満たす範囲でConfigの文法・設定項目・検証を小さくするための資料。先ほど作成したConfig案を収録し、今回承認された3点を確定事項として記録する。

## 今回確定した3点

| ID | 決定 | 具体例・動作 |
|---|---|---|
| C-01 | **rootとaliasを区別する** | rootはディレクトリにURI全体を追加する。aliasはlocationのprefixを指定ディレクトリへ置き換える。課題のprefix置換例はaliasで実現する |
| C-02 | **error_pageはファイルを直接指定する** | NGINXのURI指定・内部リダイレクトは行わない。指定ファイルを本文として読み、未指定・読み取り失敗時は内蔵エラーページを返す |
| C-03 | **相対パスは設定ファイルのディレクトリ基準** | `conf/default.conf` 内の `root ../www/site1;` はプロジェクトの `www/site1` を指す |

C-01はNGINXのroot / aliasと同じ意味。C-02は実装を簡略化するための意図的な差異。C-03はこのプロジェクトで採用するパス解決の基準である。

## 既存資料との関係

- [Requirements.md](../Requirements.md)の既決定である「NGINX風の名前と独自名の併用」「継承なし」「必須値の自動補完なし」「最長前方一致」「Pythonを最初のCGIにする」を前提とする。
- [03 設定・CGI・アップロード](03_config_cgi_upload.md)は調査論点の一覧。本書はConfigの具体案と合意状況を記録する。
- Requirements内のConfig節には以前の「未承認」表記があるが、今回の3点の合意状況は本書を参照する。その他の詳細案まで一括して確定したものとは扱わない。

## 設定の詳細案

以下は先ほどの提案をまとめたもの。上記3点と既決定を除く文法の制限・ディレクティブの必須性・追加の検証条件は、二人で確認するための案として読む。

### 基本方針

- `server` / `location`、`;`、`{}`、`#`コメントを採用する。NGINX設定全体との互換性は持たせない。
- 既決定の「継承なし」を維持する。serverの設定はserver自身の属性、locationの設定はそのlocation内だけで有効とする。
- 必須設定の値は補完しない。配信locationには無効な機能も `off` と明記する。
- **確定**: `error_page` 未指定・読み取り失敗時は内蔵エラーページを使う。
- **案**: リダイレクト専用locationは配信設定を必要としない。「全必須項目を明示」の適用範囲として二人で確認する。
- 1つのserverにつきlisten先は1組。複数のserverを並べて複数ポート・複数サイトに対応する。Hostによる仮想ホスト選択は行わない。
- **確定**: NGINXの `root` と `alias` は意味を分けて採用する。**案**: 同じlocationには片方だけを指定する。

### 文法

```text
設定ファイル := serverブロックを1個以上
server      := server { server用ディレクティブとlocationブロック }
location    := location URL-prefix { location用ディレクティブ }
単純設定    := 名前 引数... ;
```

- 最上位には `server` だけを書く。各serverにはlocationを1個以上置く。`http` / `events` ラッパーとlocationの入れ子は扱わない。
- 空白・タブ・改行を区切りにし、`;` / `{` / `}` は独立したトークンとして読む。ブロック末尾の `}` に `;` は付けない。
- `#` から行末まではコメント。引用符・エスケープ・変数展開は扱わず、空白や `;{}#` を含む値は記述できない。引用符・バックスラッシュ・`$` を含むトークンは設定エラーとする。
- 名前・メソッド・`on` / `off` は大文字小文字を区別する。ディレクティブの記述順には依存しない。
- 未知の名前、別階層の設定、引数数の違い、重複禁止項目の重複はすべて設定エラーとする。
- `include`、正規表現location、`=` / `^~` / `~` 等の修飾子、名前付きlocation、`if` / `rewrite` / `try_files`、設定reloadは実装対象外。

### serverの設定

| ディレクティブ | 書式例 | 必須性・回数 | この課題での意味・制限 |
|---|---|---|---|
| `listen` | `listen 127.0.0.1:8080;` | 必須・1回 | IPv4アドレスと1〜65535のポートを明示。`0.0.0.0`も可。ポートだけの指定、ホスト名、IPv6、追加オプションは扱わない |
| `client_max_body_size` | `client_max_body_size 1048576;` | 必須・1回 | 正の十進整数、単位はバイト。`0`、符号、`1m`等の単位、size_tの範囲を超える値は拒否。復号後ボディに適用する |
| `error_page` | `error_page 404 ../www/errors/404.html;` | 任意・複数可 | 1行に400〜599のコード1個とファイルパス1個。同一コードの重複は禁止。NGINXの内部リダイレクトは行わず、指定ファイルをエラー本文として読む |
| `location` | `location /files/ { ... }` | 必須・1個以上 | `/`で始まるURL-prefix。同じserver内で同一prefixを重複させない |

- 同じIPv4:portの重複は起動エラー。同じポートで `0.0.0.0` と特定アドレスを併用する設定も拒否する。異なる特定IPv4アドレス同士は同じポートを指定できる。
- `error_page` 未指定、または指定ファイルを読めない場合は内蔵ページへフォールバックし、元のエラーステータスを維持する。別URLへの再ルーティングやCGI実行は行わない。
- `server_name` は実装しない。接続を受け付けたlisten先のServerConfigを使用する。

### locationの設定

locationは「配信」と「リダイレクト」の2種類に分ける。`return` があればリダイレクト、それ以外は配信として検証する。

| ディレクティブ | 書式例 | 配信location | リダイレクトlocation |
|---|---|---|---|
| `allow_methods`（独自） | `allow_methods GET POST DELETE;` | 必須・1回 | 必須・1回 |
| `root` | `root ../www/site1;` | `root` / `alias` の片方だけ必須 | 指定不可 |
| `alias` | `alias ../uploads/;` | `root` / `alias` の片方だけ必須 | 指定不可 |
| `index` | `index index.html;` | 必須・1回 | 指定不可 |
| `autoindex` | `autoindex off;` | 必須・1回 | 指定不可 |
| `upload_store`（独自） | `upload_store off;` または `upload_store ../uploads;` | 必須・1回 | 指定不可 |
| `cgi_extension`（独自） | `cgi_extension off;` または `cgi_extension .py /usr/bin/python3;` | 必須・1回 | 指定不可 |
| `return` | `return 301 http://127.0.0.1:8080/;` | 指定不可 | 必須・1回 |

- `allow_methods` はGET / POST / DELETEから1個以上、重複なし。NGINXの `limit_except` は実装せず、直接リストを指定する。対象locationで禁止されたメソッドは405とAllowヘッダーで応答する。
- `index` は単一ファイル名のみ。`/`を含むパスや `.` / `..` は拒否する。指定ファイルがない場合はautoindexへ進み、`off`なら403。indexファイルの存在自体は起動条件にしない。
- `autoindex` は `on` / `off` のみ。出力形式はHTMLに固定する。
- `upload_store off` はサーバーによるアップロード保存を無効にする。パス指定時はそのディレクトリへ保存する。GETで公開するには `root` / `alias` も保存先に対応させる。
- `cgi_extension` は `off`、または拡張子とインタプリタの組1個。拡張子は `.py` に固定せず、`.` と1文字以上の英数字からなる値を受理し、大文字小文字を区別する。動作確認はまずPythonで行う。インタプリタは絶対パスで明示する。拡張子一致で直接CGIを起動し、FastCGI設定は扱わない。
- 同じlocationでアップロード保存とCGIを両方有効にはしない。必要ならlocationを分ける。CGI自身がボディを処理してファイルを保存する機能は `upload_store` とは別。
- 配信locationでPOSTを許可するなら、アップロード保存またはCGIのどちらかを有効にする。CGIを有効にするlocationの許可メソッドはGET / POSTに限定する。
- `return` は301 / 302と `http://` または `https://` で始まる固定URLのみ。変数展開、元URI・queryの自動追加、応答本文の指定は行わない。
- 処理対象のlocationを決めた後は、許可メソッドを確認してからreturnまたは配信処理へ進む。これは本課題用の処理順序とする。

### locationの選択とパス

- 既決定どおり、queryを除いたURLパスに対する**最長の文字列前方一致**。設定順には依存しない。例えば `/a` は `/abc` にも一致する。ディレクトリ単位の設定には `/a/` を使う。
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

以下は、確定した3点と残りの詳細案を組み合わせた例。文法・必須項目の最終確定後に実装用設定へ反映する。

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

- BのConfigParserが全設定を解析・検証してから、不変のConfigをAへ渡す。1つでも設定エラーがあれば設定全体を不採用とし、listenを開始しない。
- AはServerConfigごとにlistenし、Clientにその設定を紐付ける。body上限は最初のappendDataより前にHttpRequestへ設定する。bind等の失敗で起動を中止する場合は、作成済みfdも閉じる。
- 採用時のLocationConfigは、共通の `path` / `allowedMethods` と、配信かリダイレクトかの種別を持つ。配信側は `pathMode(ROOT/ALIAS)` / `basePath` / `index` / `autoindex` / `uploadPath` / CGI設定、リダイレクト側はcode / URLを持つ。既存クラス案の `root` だけではrootとaliasを区別できないため、採用時に変更する。
- `upload_store off` は空のuploadPath、`cgi_extension off` は空のcgiExtensionsとして保持できる。構文解析時は「未指定」と「明示off」を別に記録し、必須性を検証してからConfigを完成させる。
- 文法エラーは既決定どおり、理由を表示して起動を中止する。行番号は実装しない。設定項目名と、分かる場合はserverのlisten先またはlocationをメッセージに含める。
- 二人で共有する確認例: 上記設定の受理、必須項目不足、未知設定、階層違い、root/alias併記、同一listen先、数値範囲外、redirect専用location、error_page未指定時の内蔵ページ、root/aliasの対応例。

### 合意状況と残る確認事項

チェック済みは今回の会話で確定した内容。未チェックは、先ほどの提案として残している詳細であり、すべてが承認済みという意味ではない。

| 状態 | 内容 |
|---|---|
| [x] | rootはURI全体追加、aliasはlocationのprefix置換として、NGINXと同じ意味で区別する |
| [x] | error_pageはエラー本文用のファイルパスを直接指定する。未指定・読み取り失敗時は内蔵ページを使う |
| [x] | 相対ファイルパスの基準を設定ファイルのディレクトリに統一する |
| [ ] | NGINX風の限定文法として、http/events・変数・include・正規表現・引用符を省略する |
| [ ] | ディレクティブ一覧の必須性・回数・明示offと、リダイレクト専用locationの条件を確定する |
| [ ] | root/aliasを排他にし、aliasはprefixと値が末尾 `/` のディレクトリ対応に限定する |
| [ ] | listenはIPv4:portを1serverに1組、body上限は正のバイト数に限定する。0による無制限指定は採用しない |
| [ ] | allow_methods / upload_store / cgi_extensionの独自設定名と、CGI・アップロードの組み合わせ制限を確定する |
| [ ] | returnを301 / 302と固定の絶対URLに限定する |
| [ ] | 重複・数値範囲・パス存在等の起動時検証と、Configのデータ表現・A/B間のAPIを確定する |

アップロード形式・ファイル名・上書き、URI正規化・symlink、CGI出力解析の詳細はHTTP側の別議題として残す。ConfigParserにそれらの処理を持たせない。

参照した一次資料:

- [NGINX Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html): ディレクティブ・ブロック・コメント、静的配信、FastCGIとの区別
- [NGINX root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)、[alias](https://nginx.org/en/docs/http/ngx_http_core_module.html#alias)、[location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location): パス対応・prefix選択
- [NGINX listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)、[client_max_body_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)、[error_page](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page): 元の設定と、この案で省略・変更した機能
- [NGINX index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index)、[autoindex](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html#autoindex)、[return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return): ディレクトリ配信・リダイレクト
