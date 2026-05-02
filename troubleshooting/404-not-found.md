404 not-found　まとめ

パターン①：　指定した位置にファイルが存在しない
リクエストされたファイルが指定されたパスに存在しない
curl http://localhost:8080なら403が出る
ディレクトリまでは見つかっていてそこの先にファイルがない＋一覧表示がない
curl http://localhost:8080/index.htmlなら404が出る
ファイルまでを指定しているため、そこになければnot found

また、この時、nginxサービスとしては異常な行動だと認識しないことの方が多い。そのため/var/log/ndinx/error.logにログに出ないこともある
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>

直し方
指定している位置にファイルを置く
sudo touch /var/www/html/index.html

パターン②：　全く見当違いのページを入力している。ディレクトリも違う
 open() "/var/www/html/image" failed (2: No such file or directory), client: 10.0.2.2, server: _, request: "GET /image

直し方
正しいURLを打つ

## 学び
404が出る時は、リクエストを送る側のミスの可能性が高い
もちろんサーバ設定ミスでも起きる（root間違いなど）
