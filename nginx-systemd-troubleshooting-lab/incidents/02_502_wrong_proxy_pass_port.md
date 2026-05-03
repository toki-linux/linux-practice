# 02 502 Bad Gateway - proxy_passのポート不一致

## 症状
curlでアクセスしようとした時に502と表示される

## 期待される状態
pythonサーバのwebページが見えること

## 実際の状態
502 bad gateway

## 原因候補
nginx未起動  
ポートが開いていない  
proxyの設定ミス  
pythonサービスの設定ミス  

## 確認したログ・コマンド
1. nginxの状態
実行コマンド：
systemctl status nginx
結果：
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running)

2. pythonアプリサービスの確認
実行コマンド：
systemctl status myapp
結果：
● myapp.service - My Python App Service
     Loaded: loaded (/etc/systemd/system/myapp.service; disabled; preset: enabled)
     Active: active (running)

3. ポート確認
実行コマンド：
ss -tulnp | grep -E '(:80|:3000|:3999)'

結果：
tcp   LISTEN 0      511        0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=1280,fd=6))
tcp   LISTEN 0      5        127.0.0.1:3000      0.0.0.0:*    users:(("python3",pid=1442,fd=3))

4. Pythonアプリ単体の確認
実行コマンド：
curl http://127.0.0.1:3000

結果：
Hello from Python App Service

5. Nginx経由の確認
実行コマンド：
curl http://localhost/app/

結果：
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0</center>
</body>
</html>

6. Nginx access.log
実行コマンド：
sudo tail -n 20 /var/log/nginx/access.log

結果：
127.0.0.1 - - [03/May/2026:11:02:10 +0900] "GET /app/ HTTP/1.1" 502 157 "-" "curl/8.5.0"

7. nginx error.log
実行コマンド：
sudo tail -n 30 /var/log/nginx/error.log

結果：
2026/05/03 11:02:10 [error] 1283#1283: *18 connect() failed (111: Connection refused) while connecting to upstream, client: 127.0.0.1, server: _, request: "GET /app/ HTTP/1.1", upstream: "http://127.0.0.1:3999/", host: "localhost"

8. Nginx設定ファイルの確認
実行コマンド：
sudo grep -n "proxy_pass" /etc/nginx/sites-enabled/default

結果：
48:        proxy_pass http://127.0.0.1:3999/;

## 切り分け
nginxはactiveなためnginx停止しているわけでわない
pythonサービスもactiveなため停止しているわけではない
ポートが開いているかの確認をしたところ、80番3000番共に空いていた
pythonアプリ単体でcurlで繋がるか確認したところ繋がっため、pythonの設定は問題なさそうだ
nginxを経由してcurlで繋がるか確認したところ、502が出た
nginx access.logも確認したが502出ていた
nginx errror.logを確認したところ、connect() failed (111: Connection refused) while connecting to upstreamが出た。
nginx設定ファイルを確認したらproxyの欄のポートを間違えていたためポートの不一致が起こっていた

## 原因
nginx設定ファイルのproxy_passを確認したらpythonサービスが待ち受けていないポート番号を設定していた

## 解決
nginx設定ファイルのproxy_passをpythonサービスが待ち受けている3000番に書き換える

## 解決後の確認
nginx設定ファイルに不備がないかチェック
sudo nginx -t
設定を反映
sudo systemctl reload nginx
curlでnginx経由で繋がるか確認
curl http://localhost/app/

## 学び
Pythonアプリ単体にアクセスできるのに、Nginx経由で失敗する場合は、
Nginx側の設定やNginxからupstreamへの中継部分を優先して確認する。









