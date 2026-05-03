# nginx-systemd-troubleshooting-lab

## 目的

Nginx、Pythonアプリ、systemdを組み合わせた構成で、Webサーバ障害を再現し、ログ・プロセス・ポート・設定ファイルから原因を切り分ける練習を行う。

## 構成

Client / curl  
↓  
Nginx :80  
↓ /app/ をリバースプロキシ  
Python App :3000  
↓  
systemd myapp.service

## 記録する内容

- 症状
- ログ
- 原因候補
- 切り分け
- 原因
- 解決
- 学び

## 学習方針

エラーを暗記で解決するのではなく、ログと状態確認から原因を切り分ける力を身につける。
