# WEBSERV

チーム用の資料案内。提出用の英語READMEは [Requirements第9節](Requirements.md) に従って別途整える。

## 担当分担

| 担当者 | Requirementsの担当記号 | 主な担当 |
|---|---|---|
| tasugiya | A | ネットワーク層・イベントループ |
| rysato | B | HTTP層・設定ファイル・ルーティング |
| 両名 | 共同 | CGIの結合・共通処理 |

詳細は [Requirements.md](Requirements.md) と [作業分解](reports/00_README_overview.md) を参照。Config・CGIの設計判断は合意済み。CGI内の担当境界・APIは [03](reports/03_config_cgi_upload.md#cgi-api)、Configの決定表は [05](reports/05_config_agreement.md#config-decisions) を参照。CGIの起動・計時方法は検証後に見直し得る採用案。HTTPの1/10(要求行)・2/10(入力ヘッダー)・3/10(本文/chunked)・4/10(静的ファイル)・5/10(autoindex)・6/10(アップロード)は合意済み。7/10(DELETE/途中ファイルの後始末)は動作方針を合意済みで、削除手段は未解決。8/10(HTTP応答生成)は合意済みで、Date省略は暫定案。9/10(HTTP API/データの保持)も合意済み。10/10(ネットワーク境界・計時・通常ファイルI/O、HTTP-65〜74)も合意済み(採用案を含む)。10項目の動作方針の判断は完了し、削除手段の未解決・Dateの暫定案・採用案の実環境検証は [04](reports/04_testing_and_workplan.md#remaining-design) で管理する。

## 設定ファイル参考
https://props-room.com/articles/handbook/nginx-guide-237
https://qiita.com/ponponnsan/items/23e1aa6f7dd4eadde5df
https://nginx.org/en/docs/beginners_guide.html#conf_structure

## 関数参考
https://momo-chienoki.com/C/SystemCall/C_SystemCall_poll/

## CGI参考
https://www.coins.tsukuba.ac.jp/~syspro/2016/2016-06-15/index.html
https://www.coins.tsukuba.ac.jp/~syspro/2022/2022-07-27/cgi-python.html

## 全体参考
https://qiita.com/ryhara/items/c46fe320332b237b5c0d

## レビュー参考
https://www.42evalhub.com/common/webserv
https://github.com/JUNNETWORKS/42-webserv/blob/main/docs/review.md#レビュー

