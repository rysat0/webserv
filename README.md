# webserv

C++98でHTTPサーバーを作る42の課題です。開発と動作確認はLinuxを前提とします。このREADMEはチーム向けの実装案内です。提出時は課題で指定された英語READMEの構成に従います。[提出用READMEの要件](Requirements.md#submission-readme)

## まず何を作るか

HTTP担当は、`HttpRequest`で本文のないGETを解析するところから始めます。たとえば `GET / HTTP/1.1\r\nHost: localhost\r\n\r\n` を複数回に分けて渡し、最後の空行がそろうまで `INCOMPLETE`、そろったら `COMPLETE` になることを確認します。ソケットの実装前でも、この確認はできます。[実装の始め方](reports/06_implementation_guide.md)

設定担当の`ConfigParser`とネットワーク担当の最小サーバーも並行して進められます。HTTP側は`HttpResponse`で応答を組み立て、`Router`で要求と設定に応じた処理を選ぶところまで進めたら、`Client`とつなぎます。

## 誰が何を担当するか

| 担当 | 作るもの |
|---|---|
| rysato | 設定の読み込みと検証、HTTP要求の解析、ルーティング、HTTP応答の生成 |
| tasugiya | 待ち受けソケット、各接続を扱うClient、単一の`poll()`で入出力を調整するEventLoop |
| 共同 | CGIの結合、HTTP側とネットワーク側の受け渡し |

## どの資料を読むか

| 知りたいこと | 資料 |
|---|---|
| 着手順と最初に確認する動作 | [実装の始め方](reports/06_implementation_guide.md) |
| 課題の必須要件、チームが決めた動作 | [要件・設計の正本](Requirements.md) |
| 担当、資料の全体像、実装順 | [全体像と作業分担](reports/00_README_overview.md) |
| HTTPの構文と処理の背景 | [HTTPプロトコル](reports/01_http_protocol.md) |
| ソケット、`poll()`、期限、接続の後始末 | [ネットワークとI/O](reports/02_network_io.md) |
| 静的配信、アップロード、CGIと共有API | [ルーティングとCGI](reports/03_config_cgi_upload.md) |
| 設定ファイルの文法、検証、型 | [Configの設計と合意事項](reports/05_config_agreement.md) |
| 確認する動作、負荷試験、保留の項目 | [試験項目と進行計画](reports/04_testing_and_workplan.md) |

## 調査・レビュー用の参考資料

- 設定ファイルの構造は、[NGINXの入門ガイド](https://nginx.org/en/docs/beginners_guide.html#conf_structure)、[設定の解説](https://props-room.com/articles/handbook/nginx-guide-237)、[設定例](https://qiita.com/ponponnsan/items/23e1aa6f7dd4eadde5df)を参照。
- I/Oは[システムコール `poll()` の解説](https://momo-chienoki.com/C/SystemCall/C_SystemCall_poll/)を参照。
- CGIは筑波大学の[CGIの資料](https://www.coins.tsukuba.ac.jp/~syspro/2016/2016-06-15/index.html)と[Pythonでの例](https://www.coins.tsukuba.ac.jp/~syspro/2022/2022-07-27/cgi-python.html)を参照。
- 全体の調査とレビューには、[webservの解説](https://qiita.com/ryhara/items/c46fe320332b237b5c0d)、[評価項目の参考](https://www.42evalhub.com/common/webserv)、[他チームのレビュー資料](https://github.com/JUNNETWORKS/42-webserv/blob/main/docs/review.md#レビュー)を参照。
