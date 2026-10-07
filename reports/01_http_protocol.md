# 01 HTTPプロトコル — 理解すべきこと(主担当: rysato、一部共同)

一次情報: RFC 9112(メッセージ構文)、RFC 9110(意味論)、RFC 3986(URI)、RFC 1945(HTTP/1.0)。

本チームの確定範囲は[Requirements](../Requirements.md)を参照。要求行の1/10はHTTP-01〜05、入力ヘッダーの2/10はHTTP-06〜11、本文/chunkedの3/10はHTTP-12〜19、静的ファイルの4/10はHTTP-20〜26、autoindexの5/10はHTTP-27〜32、アップロードの6/10はHTTP-33〜44で合意済み。DELETE/途中ファイルの後始末の7/10はHTTP-45〜49の動作方針を合意済みで、HTTP-50の削除手段は未解決。HTTP応答生成の8/10はHTTP-51〜58で合意済み(Date省略は暫定案)。HTTP API/データの保持の9/10はHTTP-59〜64で合意済み。HTTP/1.2〜1.9は1.1の要求規則を適用し、受信表記を保持する。HTTP/1.0で応答し、HTTP/1.0と課題に必要なHTTP/1.1要求を受理する。1接続1要求で、keep-alive・pipelining・100 Continue・chunked応答は実装対象外。以下ではRFCの学習項目と、採用済みの方針・未決定事項を区別する。

---

## H1 メッセージの構造 【rysato(主)/共, R:M, D:M, ★★★】

**理解すべきこと**

- メッセージ = `start-line CRLF *(field-line CRLF) CRLF [message-body]`(RFC 9112 §2.1)。
- リクエストの start-line は `method SP request-target SP HTTP-version`、レスポンスは `HTTP-version SP status-code SP [reason-phrase]`。
- **確定(C-31)**: 要求行・ヘッダーはCRLF限定。不正な単独CR/LFは400、断片末尾のCRは続きを待つ。ボディには適用しない。CGI出力のLF/CRLF受理は別規則。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- RFC 9112 §2.2は単独LFの受理を許容するが必須ではない。本チームはC-31のCRLF限定を採用する。
- ヘッダー終端(空行 `\r\n\r\n`)の検出方法。TCPは断片で届くため**バッファ内の境界探索**が必要(逐次パース `appendData` 方式の根拠)。
- **確定(C-32)**: 要求行はCRLF込み8 KiB、ヘッダー合計は終端空行込み32 KiB。上限と同値は受理、超過は受信中に414/431。要求行・ボディをヘッダー合計へ含めず、行別・個数の追加上限は設けない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。

**資料化する内容**: 内部解析段階(要求行→ヘッダー→本文/chunked)と、外部へ公開するINCOMPLETE/COMPLETE/ERROR(HTTP-60)の対応。内部段階を外部Stateの列挙値として追加しない。異常入力と返すコードを既決定の表へ対応させる。

**調べる時の落とし穴**: 旧RFC(7230)の用語と混在した記事が多い。obs-fold(行折り返し)は9112では**拒否または空白置換**と規定されている。本チームはC-33で400による拒否を選択した。

## H2 リクエスト行・URI・パス処理 【rysato, R:M, D:M, ★★★】

**理解すべきこと**

- **確定(HTTP-01〜05)**: 要求行の正本は [Requirements](../Requirements.md#http-request-line)。SP各1個の厳密な書式、メソッドのtoken検証、GET/POST/DELETE、未実装501、バージョンの書式400/未対応メジャー505を採用する。要求先はorigin-formに限定し、その他の形式を対応メソッドで受けたら400。absolute-formはRFC上必須だが、本課題向けの限定仕様として省く。
- 要求先の文字集合と%XXをHTTP-05で検査する。最初の?でパスとqueryを分離し、queryは構文検査だけで保持する。通常のURLのfragmentは送信対象外で、要求先に生の#が届いた場合は400。
- **確定(C-23)**: locationはパスに対する文字列の最長前方一致で選ぶ。設定順に依存せず、`/a`は`/abc`にも一致する。分離・デコード・処理順序はC-24に従い、正規化はC-25に従う。
- **確定(C-24)**: 最初のリテラル?でqueryを分離し、パスを1回デコードする。queryは保持、パスの+は変換せず、不正な%表記・%00は400。デコードで生じた?をquery区切りとして再解釈しない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- Configのlocation prefixの文字制限・大小文字を区別する照合・マッチなし404はC-38に従う。prefixのASCII英数字と/-._~限定を要求URLやファイル名の受付制限として使わない。例えば/images/hello%20world.pngはデコード後にlocation /images/と照合できる。
- **確定(C-25)**: デコード後、location選択前に連続/・単独.・..要素を正規化する。先頭/より上へ戻る要求は400。末尾のディレクトリを示す/を保持する。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(C-26)**: 管理する公開・CGI・アップロード領域にsymlinkを置かない。検出・拒否機能は作らないため、URL正規化だけでsymlink経由の領域外アクセスを防げるとは扱わない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(HTTP-02)**: メソッドは大小文字を区別する。tokenとして不正なら400、正常だが未実装なら501。getをGETへ補正しない。対応メソッドがlocationで禁止されている場合の405/AllowはC-17を維持する。
- **確定(HTTP-04)**: HTTP/1.0・1.1を受理し、1.2〜1.9は1.1として検証する。受信表記は保持する。HTTP/2.0等の未対応メジャーの表記は505、大文字HTTP/と1桁.1桁の書式が不正なら400。

**資料化する内容**: URIの分解手順、正規化アルゴリズム(擬似コード不要・規則の列挙で可)、トラバーサル攻撃のテストパターン一覧。

## H3 ヘッダーフィールド規則 【rysato, R:M, D:S, ★★★】

**理解すべきこと**

- **確定(C-34)**: ヘッダー名は空でないRFC tokenを検証し、ASCII小文字で保持する。不正名・コロンなしは400。値は一律に小文字化しない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- 根拠: [RFC 9110 §5.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1)と[§5.6.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.2)。名前の小文字保持は本チームの実装方針。
- **確定(C-33)**: 折り返し・非空行の行頭SP/HTAB・コロン直前のSP/HTABは400。値は前後のSP/HTABだけ除去し、完全に空のCRLF行を終端にする。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- 根拠: [RFC 9112 §5.1–5.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-5.1)。HTTP/1.0要求にも同じ規則を適用するのは本チームの受付方針。
- 要求のHostは重複400(C-28・C-30)、Content-Lengthは同値も重複400(C-35)。Content-Type・Transfer-EncodingはHTTP-07・08で最大1行。ConnectionはHTTP-10で全行処理。その他の通常処理とCGI転送の区別はHTTP-11に従う。RFCの一般的な結合規則と、本課題の受付範囲の制限は区別する。
- **Host ヘッダー(C-28)**: HTTP/1.1では必須で欠落・重複・不正は400。HTTP/1.0は省略可だが、指定されたHostの重複・不正は400。具体的なHost値の受付範囲はC-30に従う。
- **確定(C-30)**: HostはIPv4またはASCIIホスト名と任意portに限定する。空値・重複・不正値・IPv6・%表記は400。DNS確認や仮想ホスト選択は行わない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(HTTP-06〜11)**: 残る入力ヘッダー規則の正本は [Requirements](../Requirements.md#http-input-headers)。Content-Typeは全要求で最大1行とし、値の具体的な解釈は処理先へ任せる。専用機能を実装しないヘッダーは共通検査後に保持してHTTP処理では解釈しない。静的配信のRange/条件付き要求はHTTP-25・26で対応範囲から外す合意済み。アップロード固有の検証はHTTP-33〜39で合意済み。
- 未知ヘッダーをHTTP処理で解釈する必要はない。CGIの `HTTP_*` に渡すヘッダーの範囲・例外・重複時の扱いは[03 C12](03_config_cgi_upload.md)のCGI-39〜50・75〜83で合意済み。全ヘッダーを無条件で転送する決定ではない。

## H4 ボディ長の決定 【rysato(主)/tasugiya, R:L, D:M, ★★★】

**理解すべきこと**(RFC 9112 §6)

- HTTP/1.1リクエストでは、`Transfer-Encoding: chunked` または `Content-Length` でボディを区切る。HTTP/1.0のボディ付き要求はContent-Lengthを使い、Transfer-Encoding付きは400で切断する。**レスポンスと違い、リクエストに「接続を閉じるまで読む」はない**。
- HTTP/1.1で `Transfer-Encoding` と `Content-Length` が**両方**ある場合は400で切断する(確定済み)。
- **確定(C-35)**: 要求のContent-Lengthは最大1行。同値重複・カンマ入りも400とし、結合や補正をしない。数値不正・桁あふれ・Transfer-Encoding併記の400も維持する。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(HTTP-06)**: Content-Lengthは前後SP/HTAB除去後のASCII数字1文字以上。先頭0は受理、空値・符号・小数・途中の空白・非数字・桁あふれは400。根拠: [RFC 9110 §8.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.6)。
- **確定(HTTP-08・09)**: Transfer-Encodingは最大1行。単独chunkedだけ復号する。不正構文・空値・chunkedの複数回/パラメーター指定・最後がchunkedでない要求は400。正しい一覧がchunkedで終わり未対応符号化を含む場合は501。従来の「chunked以外なら501」という略記をこの分類で補う。
- ボディ最大サイズ(`client_max_body_size`)の**早期判定**: ヘッダー受信時点で `Content-Length` が超過なら本文を読まず413。chunkedは復号後の累積バイト数で判定する。413の送信完了後に切断する方針は確定済み。未読データがある場合の切断手順は02 N6で詰める。
- **確定(C-36)**: Content-LengthとTransfer-Encodingが両方ない要求は、POSTも本文0バイトとして受信完了する。省略だけで411にしない。不正な長さ指定を省略扱いにせず、要求完成後の余剰データを次の要求にしない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- HEAD/GET のレスポンスと `Content-Length` の関係、1xx/204/304 はボディ禁止(§6.3)。

**共同の合意(HTTP-59〜64)**: Clientが生成時に本文上限を渡し、recv断片をappendDataへ投入する。getStateのINCOMPLETEなら継続、COMPLETEならRouterを1回だけ呼び、ERRORならgetErrorCodeからエラー応答を作る。HttpRequestはI/O/応答生成をせず、終端状態を変えない。正本は [RequirementsのHTTP API](../Requirements.md#http-api)。

## H5 chunked 転送コーディング 【rysato, R:M, D:M, ★★】

**合意済み(HTTP-12〜19)**: 正本は [Requirementsの本文/chunked](../Requirements.md#http-request-body)。以下は理解のための要点。

- サイズ行→指定バイト数の本文→CRLFを繰り返し、サイズ0の後はtrailerと最後の空CRLF行まで読む。受信断片の境界は構文エラーではなく、状態を保持して次の受信を待つ。
- サイズは16進数。大小文字・先頭0を受理し、空・符号・0x・不正文字・桁あふれ、本文後の区切り不正は400。本文中のNUL/CR/LFは変更しない。
- extensionはRFC構文を検査して捨てる。trailerは共通ヘッダー構文を検査して捨て、初期ヘッダーやCGI環境変数へ追加しない。どちらも不正構文は400。
- サイズ＋extensionの1行はCRLF込み8 KiBで超過400、extensionの要求全体の合計は32 KiBで超過400、trailer全体は終端空行込み32 KiBで超過431。正確な数え方は正本を参照。Config項目は増やさない。
- 本文上限は復号後のバイト数に適用する。正常なサイズ行から次の本文で累積超過すると分かれば、本文を待たず413とする。
- 未完了の受信EOFでは要求を破棄してcloseし、追加応答は生成しない。正常な要求が完成するまで保存・削除・CGI起動をしない。既存の早期拒否は維持。受信/送信60秒の起点・408・切断待ちはHTTP-65〜70に従う。
- CGIには全要求を受信・復号してから、本文を加工せず渡す。CONTENT_LENGTHはCGI-31に従い、親側stdinパイプを閉じてEOFを通知する。
- chunked応答は実装しない既存方針を維持する。

例: `4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\n` の復号後の本文は `Wikipedia`(9バイト)。サイズ行・区切り・終端は本文へ含めない。

根拠: [RFC 9112 §7.1](https://www.rfc-editor.org/rfc/rfc9112.html#section-7.1)・[§8](https://www.rfc-editor.org/rfc/rfc9112.html#section-8)。実装・受入試験は未完了。

## H6 メソッドの意味 【rysato, R:M, D:M, ★★★】

| メソッド | 理解すべき点 |
|---|---|
| GET | 安全・冪等。要求本文に一般に定義された意味はないが、指定があればHTTP-12に従って受信する。静的ファイル/ディレクトリ/CGIでの用途は処理先が判断し、CGIには既決定どおり本文を渡す |
| HEAD | GETと同じヘッダー、**ボディ無し**(`Content-Length` は GET と同値)。課題必須ではなく、ここでは比較のために意味を学ぶ。現行Configでは許可リストに指定不可(C-17)。追加採用する場合はHTTP/Configを合わせて変更する |
| POST | 非冪等。Webservのアップロード保存はHTTP-42の200/結果HTML。CGIは既決定の出力コードを検証して使う。新規作成時の201/LocationというRFCの推奨と区別する |
| DELETE | 冪等。HTTP-45〜49で通常ファイルのみ・成功204/本文なし・対象なし404・種別/権限拒否403・長すぎるパス414・その他500を合意済み。HTTP-50の削除手段は未解決 |
| 他(PUT/OPTIONS等) | HTTP-02に従い、基本書式が正常な未実装メソッドは501。対応済みのGET/POST/DELETEがlocationで禁止される場合は405とし、**`Allow` ヘッダーを付ける**(RFC 9110 §15.5.6) |

CGIはGET/POSTのみと確定済み。CGI用locationへのDELETEは405とし、スクリプトの削除処理へ進めない。通常のDELETEは別のlocationで提供する。現行Configではallow_methodsに指定できる値をGET/POST/DELETEに限定し、HEAD指定は起動エラーとする(C-17)。HEAD/OPTIONSの追加は現在の必須設計の残件には含めない。将来範囲を広げる場合はHTTPとConfigの両方を変更する。CGIの対応メソッドを増やした決定ではない。
ConfigではCGI有効とDELETE許可の組み合わせ自体を起動エラーにする(C-41)。受理済みCGI用locationに届くDELETE要求への405は維持する。配信POSTの処理先とupload/CGI併用の起動時検証もC-41に従う。

locationの `allow_methods` 省略時はGETのみ許可する(確定、05のC-06)。POST・DELETEは必要なlocationで明示する。CGIを有効にしてもPOSTが自動的に許可されるわけではなく、POSTを受けるなら許可リストにも含める。

**資料化する内容**: 「メソッド × 対象種別(ファイル/ディレクトリ/CGI/存在しない) × location設定」のマトリクスと返却ステータス。これがRouter仕様の核になる。

## H7 ステータスコード整理 【rysato(主)/共, R:L, D:L, ★★★】

決定事項「主要コードを網羅的に実装(基本全部)」に対し、**まず分類して、サーバーが能動的に生成するコードを確定する**ことが必要。RFC 9110 §15 が一次情報。

**作業内容**

1. 全コードの一覧化(1xx/2xx/3xx/4xx/5xx)と reason phrase の表(`HttpResponse` 内 static 表の元データ)。
2. 各コードを3分類: **(a) 本サーバーが生成する (b) 設定次第で生成 (c) 生成しない(プロキシ/キャッシュ/WebDAV専用等)**。
3. 各コードごとに 発生条件・必須/推奨ヘッダー・ボディ有無を記載。
4. 発生モジュールの割り当て(Request/Router/EventLoop/CgiExecutor)。

**想定される主要コード(叩き台)**

| 範囲 | コード | 発生条件の例 | 付随要件 |
|---|---|---|---|
| 1xx | 100 | 今回は生成しない(確定済み) | Expectの扱いはH9 |
| 2xx | 200, 204。201はRFCの学習対象/CGI出力で扱う | 静的配信・一覧・アップロード成功200、DELETE成功204 | 新規作成時201/LocationのRFC推奨とHTTP-42の200方針を区別。204は本文/Content-Lengthなし |
| 3xx | Configのreturnは301, 302(C-21)。303, 307, 308は学習用の比較対象 | Configの固定URL指定と、実ディレクトリへのGETに対する末尾スラッシュ補完301(C-27) | `Location`必須。CGIのLocation付き応答はCGI-62〜66で合意済み |
| 4xx | 400 | 構文不正、両版のHost重複・不正とHTTP/1.1のHost欠落(C-28)、HTTP/1.0のTE、TE/CL併記 | HTTP/1.0はHost省略可 |
| | 403 | 権限なし、autoindex off のディレクトリ | |
| | 404 | 対象なし、locationなし | |
| | 405 | 許可メソッド外 | `Allow`必須 |
| | 408 | リクエスト受信タイムアウト | `Connection: close` |
| | 413, 414, 431 | ボディ過大/リクエスト行過長/ヘッダー合計過大 | 413は既存Config、414/431はC-32の8 KiB/32 KiBで確定 |
| | 415 | アップロードの受付形式・符号化 | HTTP-33・37で合意済み |
| | 411 | 長さ指定の要求 | 長さ指定省略への411は採用しない(C-36) |
| | 417 | HTTP/1.1のExpect付き要求 | ボディを待たずヘッダー完了時に応答。先に確定した400/413等を優先 |
| 5xx | 500 | 内部エラー、CGI用pipe/fork失敗 | 作成済み資源を後始末する |
| | 501 | 未実装メソッド/TE | |
| | 502 | CGIのexec失敗・子の異常終了・不正出力・出力上限超過 | Content-Lengthと本文の実長の不一致も含む |
| | 503 | 一般的な意味は一時的に利用不可。現在の設計では接続件数による503を生成しない(HTTP-72) | CGI同時実行数による503は設けない |
| | 504 | CGIタイムアウト | 起動から10秒で打ち切る。出力があっても延長しない |
| | 505 | 未対応HTTPバージョン | |

**合意済み(HTTP-57・58)**: 正常な要求はlocation選択→メソッド許可→return→各処理。要求検証の早期拒否と最初に確定したエラーを維持する。全コードの一律優先順位表は作らない。

**注意**: エラー応答には `Content-Length` と `Connection: close` を付け、エラーページのボディ(デフォルトHTML生成 / 設定の `error_page` ファイル読み込み)を返す。`error_page` ファイル自体が読めない場合のフォールバック(内蔵HTML)も仕様に含める。

要求ヘッダー値はCGI-79の共通検査を適用し、HTAB以外の制御文字とDELを400とする。非ASCIIバイトは変換せず、本文にはこの検査を適用しない。CGI要求のContent-TypeはCGI-80で最大1行・空値時の空文字列を確定済み。要求Connectionの値はCGI-78で全行を検査・処理する。HTTP/1.0のExpectは値によらず無視する(CGI-76)。

CGI出力の長さ・転送規則はCGI-67〜74で確定済み。出力の同名ヘッダーはすべて最大1行とし、接続用ヘッダーは除外する。非空Transfer-Encodingは502。204・205・304の本文は認めず、空本文時のHTTP Content-Lengthは204・304で省略、205で0とする。Statusの数値検査は100〜599だが、1xxだけで終了するCGIは最終応答にできないため502とする。

CGIのエラー対応は確定済み。スクリプトなしは404、読み取り不可は403。1件あたりヘッダー8 KiB、ヘッダーを含むstdout全体8 MiBを読み取り中に検査し、超過時は打ち切って502とする。CGI同時実行数に独自上限は設けず、件数による503や待ち行列も設けない。OS資源不足によるpipe/fork失敗は500。タイムアウト後に子が異常終了しても504を502へ上書きしない。

Configのerror_pageは100〜599のコードを受理する(C-20)。この設定だけで未対応のコードの発生処理を追加せず、1xx・204・304等に本文を付けない規則も維持する。本文差し替えの適用は400〜599に限定する(C-20追記)。100〜399の設定は受理するが応答に適用せず、通常の成功応答・リダイレクト・CGIの成功応答の本文は変更しない。

## H8 レスポンスヘッダー 【rysato, R:M, D:M, ★★】

**合意済み(HTTP-51〜58)**: 正本は [RequirementsのHTTP応答](../Requirements.md#http-response)。

- HTTP/1.0・CRLF・Connection: closeを生成する。通常のCLは最終本文のバイト数(空なら0)、CTは既決定のMIME表/text/htmlを使う。本文禁止のCGIステータス等は既決定を維持する。
- Webservの405にはlocationの許可メソッドをGET, POST, DELETEの固定順でAllowへ列挙する。Webservの301/302は既決定のLocationと空本文/CL0。CGIの転送規則は維持する。
- Serverは生成しない。Dateは使用可能な暦日時取得手段の確認まで生成しない暫定案。DateのRFC推奨/必須規定との差、免除と説明しないこと、見直し条件はHTTP-56を参照する。CGI由来のServer/Dateを除去する決定ではない。
- Last-Modified/ETagは静的配信で生成しない(HTTP-26)。charsetの一律付与・文字コード変換も追加しない。
- **CGI応答の変換(確定済み)**: 通常の文書応答はContent-Type・空行・本文を解釈し、Status省略時は200。Status指定時はHTTPステータス行へ反映し、Statusヘッダー自体は転送しない。CGIヘッダーのLF/CRLFを受理し、HTTPヘッダーはCRLFで生成する。本文の改行は変換しない。
- CGIのstdoutは上限付きで蓄積し、EOFと子の終了の両方を確認してから応答を生成する。Content-Lengthなしなら本文の実長から生成し、指定ありでもEOFまで読み、実長との不一致は502。ブラウザへの逐次転送はしない。
- 不正なCGI出力を推測・補正して受理する処理は実装しない。必要な検査で不正と判定したら502にする方針が確定済み。具体的な検証規則はCGI-52〜74で合意済み。説明は [03 C12](03_config_cgi_upload.md#cgi-details) を参照する。
- 絶対URLのLocationだけを返すCGI応答は302に変換する。ローカルパスのLocationだけによる内部再処理は対象外。RFC 3875を参考に対応範囲を限定しており、完全準拠は掲げない。Location・Status・Content-Typeを伴う形式は301/302/303/307/308のみ受理する。不完全な形式とローカル・相対・ネットワークパスのLocationは502(CGI-62〜66)。詳細は[03 C12](03_config_cgi_upload.md)。
- CGIの204/304は本文・Content-Lengthなし、205は本文なし・Content-Length: 0で応答する。これらに非空本文があれば502(CGI-73)。各処理の生成コードは合意済みのHTTP/CGI規則を使い、通常接続の期限・資源量はHTTP-65〜74に従う。HEADは現行Configの許可値に含めない。

## H9 接続管理 【共, R:M, D:M, ★★】

**合意済み(HTTP-65〜74)**: [Requirementsのネットワーク境界](../Requirements.md#http-network-boundary)が正本。受信/送信60秒、送信後の半閉鎖/読み捨て最大1秒、経過時間の採用案と見直し条件を含む。

**理解すべきこと**(RFC 9112 §9)

- 本チームは**HTTP/1.0応答固定・1接続1要求**。受信した要求がHTTP/1.1でも、応答はHTTP/1.0で `Connection: close` を付ける。CGIへ渡す `SERVER_PROTOCOL` は受信バージョンを使う。
- `Connection: close` 付きで応答後、サーバー側が close するタイミングと**送信バッファを送り切ってから閉じる**必要(→ 02 N4/N6)。
- keep-alive・pipeliningは今回の実装対象外。要求されても次の要求は処理せず、1回の応答を送信し終えて切断する。
- HTTP/1.1のExpect付き要求は、値によらずヘッダー完了時点で417とし、ボディを待たない。先に確定した400/413等があればそちらを優先する。100 Continueは実装しない。HTTP/1.0のExpectは値によらず無視して通常処理する(CGI-76)。
- ブラウザの挙動: 複数接続の並列オープン、空接続(データを送らず接続だけ張る投機接続)がある → 受信タイムアウトとの関係。

## H10 MIMEタイプ 【rysato, R:S, D:S, ★★】

- **合意済み(HTTP-23)**: 固定表の正本は [Requirementsの静的ファイル](../Requirements.md#http-static-files)。実際のファイル名の最後の拡張子をASCII大小文字非区別で判定し、未知・拡張子なしはapplication/octet-stream。内容推測・追加Config・文字コード変換は行わず、UTF-8を一律に宣言しない。
- `Content-Type` が誤ると**ブラウザで表示/DLの挙動が変わる**(課題の「完全な静的サイト配信」に直結)。

## H11 リダイレクト 【rysato, R:S, D:S, ★★】

- **確定(C-21)**: Configのreturnはlocation内の301/302＋固定HTTP(S)絶対URLの2引数のみ。URLにはホスト部分を要求し、相対URL・他のコード・不正な値/引数数は起動エラー。変数展開・元URI/queryの自動追加・本文指定は省き、設定URLをLocationで返す。301/302/303/307/308の違いは学習項目であり、全コードをConfigで受理する意味ではない。
- **確定(C-22)**: location選択後、許可メソッドを確認してからConfigのreturnを適用する。returnのみのlocationは省略時GETのみの規則によりGETには301/302、POST・DELETEには405とAllowを返す。allow_methodsを明示した場合はその許可リストを使う。
- RFC 9110 §10.2.2はLocationの相対参照も認める。一方、参考にするHTTP/1.0のRFC 1945 §10.11はabsoluteURIを定義する。本Configのreturnは固定HTTP(S)絶対URLのみ(C-21)。自動補完もC-28のhttp絶対URLとし、パスの再エンコードとqueryの保持はC-29に従う。
- **確定(C-27)**: location選択・メソッド確認後、GETの配信先が実ディレクトリで末尾/がなければqueryを保持して301。locationの一致規則は変更しない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(C-28)**: 末尾/補完のLocationはhttp絶対URL。有効なHostを明示portごと使い、port省略時は追加しない。1.0のHost欠落時だけ接続先の実IP/portを使う。ConnectionInfoのAPIはCGI-93で合意済み。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。
- **確定(C-29)**: 補完URLは内部のデコード・正規化済みパスから生成し、ASCII英数字・-._~・/以外を大文字%XXへ戻す。queryは元の表記で保持し、Configのreturnは変換しない。 詳細・例は [05](05_config_agreement.md#config-decisions) を参照。

ここは通常のHTTPリダイレクトの学習項目。ConfigからのHTTPリダイレクトは実装するが、CGIがローカルパスのLocationだけを返す内部リダイレクトは対象外という確定事項と区別する。

---

## この章の作業量サマリ

| 区分 | R合計の目安 | D合計の目安 | 備考 |
|---|---|---|---|
| H1–H5(メッセージ/パース) | L〜XL | L | 状態遷移図とエラーコード表が成果物。最も仕様依存が強い |
| H6,H8–H11 | M | M | 表中心の資料。調査は速いが決定事項が多い |
| H7(ステータスコード) | L | L | 分類+表作成。時間がかかるが機械的で分担しやすい |
