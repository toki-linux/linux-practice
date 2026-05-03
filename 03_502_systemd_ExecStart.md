# 03 502 Bad Gateway - systemd ExecStartミス

## 症状

`curl http://localhost/app/` でアクセスした時に、502 Bad Gateway が表示される。

## 期待される状態

Nginx経由でPythonアプリにアクセスし、PythonサーバのWebページが表示されること。

## 実際の状態

502 Bad Gateway が表示された。

## 原因候補

- Nginxサービスが停止している
- Nginxが80番ポートで待ち受けていない
- Nginxのproxy_pass設定が間違っている
- Pythonアプリサービスが停止、または起動に失敗している
- Pythonアプリが3000番ポートで待ち受けていない
- myapp.service の ExecStart に誤りがある

## 確認したログ・コマンド

### 1. Nginxの状態確認

実行コマンド：

```bash
systemctl status nginx
```

結果：

```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running)
```

判断：

```text
Nginxは active (running) のため、Nginxサービスは起動している。
```

---

### 2. Pythonアプリサービスの状態確認

実行コマンド：

```bash
systemctl status myapp
```

結果：

```text
● myapp.service - My Python App Service
     Loaded: loaded (/etc/systemd/system/myapp.service; disabled; preset: enabled)
     Active: failed (Result: exit-code) since Sun 2026-05-03 11:35:41 JST; 2min ago
    Process: 1602 ExecStart=/usr/bin/pythn3 -m http.server 3000 --bind 127.0.0.1 (code=exited, status=203/EXEC)
   Main PID: 1602 (code=exited, status=203/EXEC)

May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed to execute /usr/bin/pythn3: No such file or directory
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed at step EXEC spawning /usr/bin/pythn3: No such file or directory
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Main process exited, code=exited, status=203/EXEC
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed with result 'exit-code'.
```

判断：

```text
myapp.service は failed になっており、Pythonアプリサービスの起動に失敗している。
また、ExecStartに指定された /usr/bin/pythn3 が実行できていない。
```

---

### 3. myappのログ確認

実行コマンド：

```bash
journalctl -u myapp -n 30
```

結果：

```text
May 03 11:35:41 ubuntu-toki systemd[1]: Started myapp.service - My Python App Service.
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed to execute /usr/bin/pythn3: No such file or directory
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed at step EXEC spawning /usr/bin/pythn3: No such file or directory
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Main process exited, code=exited, status=203/EXEC
May 03 11:35:41 ubuntu-toki systemd[1]: myapp.service: Failed with result 'exit-code'.
```

判断：

```text
journalctlでも /usr/bin/pythn3: No such file or directory と出ている。
systemdがExecStartに指定されたコマンドを実行できていないと判断した。
```

---

### 4. ポート確認

実行コマンド：

```bash
ss -tulnp | grep -E '(:80|:3000)'
```

結果：

```text
tcp   LISTEN 0      511        0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=1280,fd=6))
```

判断：

```text
80番ポートはNginxがLISTENしている。
一方で、3000番ポートはLISTENしていないため、Pythonアプリは待ち受けていない。
```

---

### 5. Pythonアプリ単体の確認

実行コマンド：

```bash
curl http://127.0.0.1:3000
```

結果：

```text
curl: (7) Failed to connect to 127.0.0.1 port 3000 after 0 ms: Connection refused
```

判断：

```text
Pythonアプリ単体にも接続できない。
3000番で待ち受けるプロセスがないため、接続拒否されている。
```

---

### 6. Nginx access.logの確認

実行コマンド：

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

結果：

```text
127.0.0.1 - - [03/May/2026:11:37:05 +0900] "GET /app/ HTTP/1.1" 502 157 "-" "curl/8.5.0"
```

判断：

```text
access.logに /app/ へのアクセスと502が記録されているため、リクエストはNginxまで届いている。
```

---

### 7. Nginx error.logの確認

実行コマンド：

```bash
sudo tail -n 30 /var/log/nginx/error.log
```

結果：

```text
2026/05/03 11:37:05 [error] 1283#1283: *22 connect() failed (111: Connection refused) while connecting to upstream, client: 127.0.0.1, server: _, request: "GET /app/ HTTP/1.1", upstream: "http://127.0.0.1:3000/", host: "localhost"
```

判断：

```text
Nginxはupstreamである http://127.0.0.1:3000/ に接続しようとしている。
しかし3000番で待ち受けているプロセスがないため、Connection refused になっている。
```

---

### 8. myapp.serviceのExecStart確認

実行コマンド：

```bash
grep -n "ExecStart" /etc/systemd/system/myapp.service
```

結果：

```text
8:ExecStart=/usr/bin/pythn3 -m http.server 3000 --bind 127.0.0.1
```

判断：

```text
ExecStartのPythonコマンドが /usr/bin/pythn3 になっており、綴りミスの可能性がある。
```

---

### 9. python3のフルパス確認

実行コマンド：

```bash
which python3
```

結果：

```text
/usr/bin/python3
```

判断：

```text
正しいPythonコマンドのフルパスは /usr/bin/python3 である。
myapp.serviceのExecStartに書かれている /usr/bin/pythn3 は誤りだと判断した。
```

## 切り分け

Nginxは `active (running)` であり、80番ポートもLISTENしていたため、Nginxサービス停止や80番ポートの待ち受け不備ではないと判断した。

一方で、`systemctl status myapp` では myapp.service が `failed` になっていた。  
また、`journalctl -u myapp` でも `/usr/bin/pythn3: No such file or directory` と表示されていた。

ポート確認でも3000番はLISTENしておらず、`curl http://127.0.0.1:3000` でも Connection refused となった。  
このことから、Pythonアプリが3000番で待ち受けていない状態だと判断した。

Nginxのaccess.logには `/app/` へのアクセスと502が記録されており、リクエストはNginxまで届いている。  
Nginxのerror.logでは、`http://127.0.0.1:3000/` への接続に失敗していた。

myapp.serviceのExecStartを確認すると、`/usr/bin/pythn3` と書かれていた。  
`which python3` の結果は `/usr/bin/python3` だったため、ExecStartに指定したPythonコマンドの綴りミスが原因だと判断した。

## 原因

myapp.service の `ExecStart` に指定したPythonコマンドのパスが誤っていた。

正しくは `/usr/bin/python3` だが、サービスファイルでは `/usr/bin/pythn3` となっていた。

そのため、systemdがPythonを実行できず、myapp.service が起動失敗した。  
結果として3000番ポートで待ち受けるプロセスが存在せず、Nginxがupstreamへ接続できなかったため、502 Bad Gateway が発生した。

## 解決

myapp.service の `ExecStart` を正しいパスに修正する。

```bash
sudo nano /etc/systemd/system/myapp.service
```

修正前：

```ini
ExecStart=/usr/bin/pythn3 -m http.server 3000 --bind 127.0.0.1
```

修正後：

```ini
ExecStart=/usr/bin/python3 -m http.server 3000 --bind 127.0.0.1
```

設定変更をsystemdに反映する。

```bash
sudo systemctl daemon-reload
```

myapp.serviceを再起動する。

```bash
sudo systemctl restart myapp
```

## 解決後の確認

Pythonアプリサービスの状態確認。

```bash
systemctl status myapp
```

3000番ポートの確認。

```bash
ss -tulnp | grep :3000
```

Pythonアプリ単体の確認。

```bash
curl http://127.0.0.1:3000
```

Nginx経由の確認。

```bash
curl http://localhost/app/
```

どちらも `index.html` の内容が返れば復旧完了。

## 学び

502 Bad Gateway はNginx側のエラーとして表示されるが、原因がNginxにあるとは限らない。

今回のように、裏側のPythonアプリサービスがsystemdのExecStartミスで起動できていない場合も、Nginxから見るとupstreamに接続できないため502になる。

エラー画面だけでは原因を特定できないため、`systemctl status`、`journalctl`、`ss`、Nginxのerror.log、サービスファイルの設定を順番に確認することが大切だと学んだ。


