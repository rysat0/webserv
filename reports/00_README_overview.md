# 実装前に理解すべきこと・作業分解 — 全体像

> 対象: webserv(C++98 HTTPサーバー)。`Requirements.md` の要件とチーム決定(担当tasugiya=ネットワーク/イベントループ、rysato=HTTP層/設定、CGIは共同)を前提に、
> **実装に入る前の「調査 → 仕様作成」フェーズ**で必要な理解事項と作業量を整理したもの。確定事項と、これから合意する仕様・設計案を分けて扱う。

実装する範囲は [Requirements.md](../Requirements.md) を基準にする。HTTP/1.0で応答し、HTTP/1.0と必要なHTTP/1.1要求を受理する。1接続1要求・`Connection: close` とし、keep-aliveとパイプライン処理は対象外。HTTP/1.1のchunked入力の復号は実装する。

必須部分のCGIはPythonのみ。RFC 3875を参考に対応範囲を限定し、確定事項と残る論点は [03のC12](03_config_cgi_upload.md) に整理する。Configはroot/aliasの区別、error_pageのファイル直接指定、設定ファイル基準の相対パス、限定文法5点、省略時のNGINX既定値の適用、allow_methods省略時GETのみが確定済みで、追加のCGI合意と残るConfig案は [05](05_config_agreement.md) を参照する。決定を変更するときはRequirementsと関連する全資料を同時に更新する。

## 資料構成

| ファイル | 内容 |
|---|---|
| `00_README_overview.md`(本書) | 全体像、凡例、一次情報一覧、全トピックの担当・工数早見表、進め方 |
| `01_http_protocol.md` | HTTP仕様(RFC 9110/9112)の理解事項: メッセージ構造、パース規則、メソッド、ステータスコード、ヘッダー、chunked、接続管理 |
| `02_network_io.md` | ソケット・poll・ノンブロッキングI/O・タイムアウト・シグナル・エラー処理(errno禁止制約を含む) |
| `03_config_cgi_upload.md` | 設定ファイル設計、ルーティング、静的配信/autoindex、アップロード、CGIの確定事項・残る論点、Cookie(必須範囲外のボーナス資料) |
| `04_testing_and_workplan.md` | 検証方法(telnet/curl/nginx比較/負荷)と、調査→仕様→実装の進行計画、決定が必要な論点リスト |
| [05_config_agreement.md](05_config_agreement.md) | NGINXを参考にしたConfig詳細案、設定例、確定したroot/alias・error_page・相対パス・CGIの扱いと残る確認事項 |

## 凡例

- **担当**: `tasugiya`(従来のA)=ネットワーク/イベントループ、`rysato`(従来のB)=HTTP/設定、`共`=両者が理解必須(重複部分)。CGI内の具体的な担当境界は03の案に従って別途確定する。
- **調査量(R)** / **資料化量(D)**: 1人が一次情報を読んで理解する時間 / 理解した内容を仕様書(md)に落とす時間。
  - `S`≈1–2h、`M`≈3–5h、`L`≈6–10h、`XL`≈10h超。**あくまで目安**(RFCを初見で読む前提の粗い見積もり。実際は着手後に補正する)。
- **優先度**: `★★★`=ここが決まらないと他が止まる/0点リスク、`★★`=必須機能に直結、`★`=あると良い/後回し可。
- 「共」の項目は**両者が同じ内容を調べると無駄**になるため、片方が調査して要点を共有する運用を推奨(担当欄の `(主)` が調査主担当)。

## 一次情報(これを基準にする)

| 略称 | 文書 | 用途 |
|---|---|---|
| RFC 9110 | HTTP Semantics (2022) | メソッド、ステータスコード、ヘッダーの意味。RFC 7231 等を置き換え |
| RFC 9112 | HTTP/1.1 (2022) | メッセージ構文、パース、chunked、接続管理。RFC 7230 を置き換え |
| RFC 1945 | HTTP/1.0 | 課題が参照基準とする版。1.1との差分把握に使う |
| RFC 3875 | CGI 1.1 | 環境変数、CGI応答の形式 |
| RFC 3986 | URI 構文 | パーセントエンコーディング、パス正規化、`..` の扱い |
| RFC 7578 | multipart/form-data | ファイルアップロードのボディ形式 |
| RFC 2046 §5.1 | MIME multipart | boundary の規則 |
| RFC 6265 | Cookie | ボーナス用 |
| POSIX / Linux man | `poll(2)`, `socket(7)`, `accept(2)`, `recv(2)`, `send(2)`, `fcntl(2)`, `fork(2)`, `execve(2)`, `waitpid(2)`, `pipe(7)`, `signal(7)`, `getaddrinfo(3)` | システムコール仕様 |
| nginx docs | `ngx_http_core_module` 等 | 設定項目・挙動の比較基準(課題が推奨) |

> 注: RFC 7230–7235 は RFC 9110/9112 で廃止されている。古いチュートリアルは旧RFC番号を引くので、**必ず9110/9112で確認**する。

## 全トピック早見表(担当・工数・優先度)

| ID | トピック | 担当 | R | D | 優先度 | 詳細 |
|---|---|---|---|---|---|---|
| H1 | HTTPメッセージの構造(start-line/headers/body、CRLF、改行寛容) | rysato(主)/共 | M | M | ★★★ | 01 |
| H2 | リクエスト行・URI・パーセントデコード・パス正規化 | rysato | M | M | ★★★ | 01 |
| H3 | ヘッダー規則(大小無視、OWS、重複、必須Host) | rysato | M | S | ★★★ | 01 |
| H4 | ボディ長の決定(Content-Length / chunked / 両方指定の扱い) | rysato(主)/tasugiya | L | M | ★★★ | 01 |
| H5 | chunked transfer coding | rysato | M | M | ★★ | 01 |
| H6 | メソッドの意味(GET/HEAD/POST/DELETE、冪等性、405/Allow) | rysato | M | M | ★★★ | 01 |
| H7 | **ステータスコード全体整理** | rysato(主)/共 | L | L | ★★★ | 01 |
| H8 | レスポンス必須/推奨ヘッダー(Date, Server, Content-Type/Length, Location, Allow) | rysato | M | M | ★★ | 01 |
| H9 | 接続管理(Connection, keep-alive, 1.0/1.1差、Expect: 100-continue) | 共 | M | M | ★★ | 01 |
| H10 | MIMEタイプ表・Content-Type | rysato | S | S | ★★ | 01 |
| H11 | リダイレクト(301/302/303/307/308の違い) | rysato | S | S | ★★ | 01 |
| N1 | TCP/ソケットAPI(socket/bind/listen/accept、SO_REUSEADDR、backlog) | tasugiya | M | M | ★★★ | 02 |
| N2 | `getaddrinfo` / host:port 解釈、複数listen、同一ポートの重複扱い | tasugiya | M | S | ★★★ | 02 |
| N3 | **poll() の厳密な意味(revents、POLLHUP/ERR/NVAL)** | tasugiya(主)/共 | L | L | ★★★ | 02 |
| N4 | ノンブロッキングI/O、部分read/write、バッファ設計 | tasugiya(主)/共 | L | L | ★★★ | 02 |
| N5 | **errno禁止制約下のエラー処理方針** | tasugiya(主)/共 | M | M | ★★★ | 02 |
| N6 | クライアント切断・半クローズ・shutdown・SIGPIPE | tasugiya | M | S | ★★★ | 02 |
| N7 | タイムアウト設計(一般60s/CGIは起動から10s・延長なし)、時刻管理 | tasugiya | S | S | ★★ | 02 |
| N8 | シグナル(SIGINT グレースフル終了、SIGPIPE、SIGCHLD) | tasugiya | M | S | ★★ | 02 |
| N9 | fd・メモリのリーク/上限(EMFILE、bad_alloc)、クラッシュ防止 | tasugiya(主)/共 | M | M | ★★★ | 02 |
| N10 | 通常ファイルI/Oの扱い(pollが不要な範囲と大容量ファイルの扱い) | 共 | S | S | ★★ | 02 |
| C1 | 設定ファイル文法(基本文法確定・クォート等は対象外、残る細則は05) | rysato | M | L | ★★★ | 03 |
| C2 | ディレクティブ一覧(必須/任意、型、検証規則)※一部確定、詳細は05 | rysato(主)/共 | M | L | ★★★ | 03 |
| C3 | location 最長前方一致・root/alias 変換・パス結合 | rysato | M | M | ★★★ | 03 |
| C4 | 静的ファイル配信(stat/access、ディレクトリ→index、権限エラーの対応) | rysato | M | M | ★★★ | 03 |
| C5 | autoindex(opendir/readdir、HTMLエスケープ) | rysato | S | S | ★★ | 03 |
| C6 | アップロード(multipart/form-data パース、保存先、サイズ制限) | rysato | L | L | ★★★ | 03 |
| C7 | DELETE の挙動設計 | rysato | S | S | ★★ | 03 |
| C8 | **CGIプロセス制御**(pipe/fork/dup2/execve/chdir、fd継承、waitpid、kill) | tasugiya(主)/共 | L | L | ★★★ | 03 |
| C9 | **CGI仕様(RFC 3875を参考に範囲限定)**: 環境変数、PATH_INFO、応答パース(Status/Location) | rysato(主)/共 | L | L | ★★★ | 03 |
| C10 | CGIとイベントループの統合(pipeをpoll、stdoutのEOFと子の終了を確認、復号済みボディ投入) | 共 | M | M | ★★★ | 03 |
| C11 | Cookie/セッション(ボーナス資料。必須範囲外) | rysato | M | M | ★ | 03 |
| T1 | テスト手法・ツール(telnet/nc/curl/ab/siege/自作スクリプト) | 共 | M | M | ★★ | 04 |
| T2 | nginx との挙動比較手順 | 共 | S | S | ★★ | 04 |

## 作業量の特徴(重い所)

調査にも資料化にも時間がかかる(`L`以上)のは次の項目。ここを早めに着手し、担当の偏りを調整する。

1. **H7 ステータスコード整理** — 「基本全部実装」という決定があるため対象が多い。`1xx〜5xx` の全体表を作り、各コードについて「本サーバーで発生する/しない」「必須ヘッダー」「ボディ有無」を決める。分類しないと実装時に迷う。
2. **H4/H5 ボディ長決定とchunked** — 仕様の読み込みが必要で、リクエストスマグリング絡みの規則(CL+TE同時指定など)がある。
3. **N3–N5 poll/ノンブロッキング/errno禁止** — 課題の0点条件に直結。「errnoを見ずにEAGAIN等をどう扱うか」の方針は設計で決め打ちが必要。
4. **C6 アップロード** — multipartのboundary処理は仕様の細部が多い(ボディ全体をメモリに持つか等の設計判断を含む)。
5. **C8–C10 CGI** — プロセス制御(tasugiya寄り)と仕様(rysato寄り)の両面があり、共同箇所のため調査を分担して要点共有しないと重複する。

## 重複部分(両者が共通理解を持つべき範囲)

| 共通理解 | 理由 |
|---|---|
| ボディ長決定とバッファの責務境界(H4 ↔ N4) | tasugiyaはバイトをためてrysatoのパーサーに渡す側、rysatoは完成判定側。境界(「どこまで読めばリクエスト完了か」)が食い違うと結合で破綻する |
| ステータスコードの発生箇所(H7) | 要求の400/413/417、ルート解決の404/403/405、CGIのpipe/fork失敗500・exec失敗/異常終了/不正出力502・時間超過504などを区別する。makeErrorでの生成は一元化し、具体的なtasugiyaとrysatoの間の失敗理由の受け渡しは確定する |
| CGI(C8–C10) | 決定事項で「共同」。起動(tasugiya)と仕様・環境変数(rysato)を分けて調べ、結合部(pipe の fd を誰が poll するか)で合意する |
| エラー時の接続処理(N5/N6 ↔ H9) | 1接続1要求・応答後closeは確定済み。未受信ボディの読み捨て、shutdown、closeの具体的な手順は両者で揃える |

## 進め方(推奨)

1. **Phase 0(0.5日)**: 本資料を読み、担当・優先度を確認、不明点を `04` の「決定が必要な論点」に追記。
2. **Phase 1 調査(各自)**: 担当トピックを★★★から調査。共通項目は主担当が調査し、**1ページの要点メモ**を作って相手に渡す。
3. **Phase 2 仕様作成**: トピックごとに「規則 / 本プロジェクトの採用方針 / 受け入れテスト(telnet/curlコマンド)」の3点セットを書く。`Requirements.md` の「未決定」(ディレクティブ一覧、シグネチャ)をここで確定。
4. **Phase 3 仕様レビュー**: 互いの仕様を読み合う(課題が求めるピアレビューの練習にもなる)。
5. 実装開始(`Requirements.md` の実装順序に従う)。

> AI利用の注意: 課題はAI生成物を「完全に理解し説明できるもののみ」使用とし、READMEへの利用箇所記載を義務づけている。本資料は調査の出発点であり、**一次情報で各自が裏取りしてから採用すること**。
