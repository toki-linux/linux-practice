# トラブルシューティングまとめ

実務を想定し、発生したトラブルを「症状 → 原因 → 解決」の形で整理。

---

## 502 Bad Gateway

### 主な原因
- upstreamサーバが起動していない
- ポートが一致していない
- 接続拒否（Connection refused）

### 切り分け
- `ss -tulnp | grep ポート番号`
- `systemctl status サービス名`
- `error.log` の `connect() failed` を確認

---

## 403 Forbidden

### 主な原因
- indexファイルが存在しない
- ディレクトリ一覧表示が禁止されている
- パーミッション問題

### 切り分け
- `error.log` の `directory index of ... is forbidden`
- `ls -la` でファイル存在確認
- nginxの `index` 設定確認

---

## 404 Not Found

### 主な原因
- ファイルが存在しない
- パスの指定ミス
- try_files の設定

### 切り分け
- `ls` でファイル確認
- `error.log` の `No such file or directory`

---

## cronトラブル

### 主な原因
- 実行ユーザーの権限不足
- パス未指定（相対パス）
- 環境変数不足

### 切り分け
- `crontab -l` / `sudo crontab -l`
- ログ出力設定
- 絶対パスで実行

---

## 学び

- エラーコードごとに原因はある程度パターン化できる
- ログを見ることで原因特定の精度が上がる
- 再現 → 原因 → 解決 の流れが重要
