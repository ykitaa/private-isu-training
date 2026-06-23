# 計測手順書（private-isu / EC2 直デプロイ）

ボトルネックを「推測ではなく計測」で特定するための手順。
1 ベンチ走行ごとに **alp（nginx）→ pt-query-digest（MySQL）→ pprof（Go）** の順で見て、
最も重い箇所から施策を選ぶ。

計測の基本ループ:

1. ログをローテート（前回分を退避して空にする）
2. ベンチマーカーを走らせる
3. 各ツールで集計して上位を確認
4. 施策を 1 つ入れて再計測 → 差分を見る

> ⚠️ **本番スコア計測時はログ出力を止める**こと（ログ I/O 自体がスコアを下げる）。
> - nginx: `/etc/nginx/nginx.conf`（http{}）の `access_log` を `access_log off;` に（または `default.conf` の server{} に `access_log off;` を1行足して上書き）→ `sudo nginx -t && sudo systemctl reload nginx`
> - MySQL: `tuning.cnf` の `slow_query_log = 0`（または `long_query_time` を上げる）

> 📌 **nginx の LTSV ログ定義の置き場所**: 本環境では `/etc/nginx/nginx.conf` の `http{}` に
> `log_format ltsv ...` と `access_log /var/log/nginx/access.log ltsv;` を直接記述して運用している。
> リポジトリの `conf.d/default.conf` 側では **再定義しない**（同名 `log_format ltsv` が二重になると
> `duplicate log format "ltsv"` で `nginx -t` が落ちるため）。`access_log ... ltsv;` の「参照」は何箇所書いてもよいが、
> 「定義」は1箇所だけにすること。

---

## 0. セットアップ（初回のみ・EC2 上で実行）

### alp（nginx アクセスログ解析）
```bash
# 最新版は GitHub Releases を確認
curl -L -o alp.tar.gz https://github.com/tkuchiki/alp/releases/latest/download/alp_linux_amd64.tar.gz
tar xzf alp.tar.gz
sudo install alp /usr/local/bin/alp
alp --version
```

### pt-query-digest（MySQL スローログ解析・Percona Toolkit）
```bash
sudo apt-get update && sudo apt-get install -y percona-toolkit
# 入らない場合は単体取得:
#   curl -L -o pt-query-digest https://raw.githubusercontent.com/percona/percona-toolkit/3.x/bin/pt-query-digest
#   chmod +x pt-query-digest && sudo mv pt-query-digest /usr/local/bin/
pt-query-digest --version
```

### 設定ファイルの反映
- nginx: LTSV ログは `/etc/nginx/nginx.conf` の `http{}` に定義済み（`log_format ltsv` + `access_log ... ltsv;`）。
  alp が必要とするキー（最低でも `method` `uri` `status` `reqtime:$request_time`、できれば `apptime:$upstream_response_time` `size`）が
  含まれているか確認する。変更したら反映:
  ```bash
  sudo nginx -t && sudo systemctl reload nginx
  # 実際に効いている定義の確認
  sudo nginx -T 2>/dev/null | grep -nE 'log_format|access_log'
  ```
- MySQL: 本リポジトリの `private_isu/webapp/etc/mysql/conf.d/tuning.cnf` を
  `/etc/mysql/mysql.conf.d/`（または `/etc/mysql/conf.d/`）へ配置
  ```bash
  sudo cp private_isu/webapp/etc/mysql/conf.d/tuning.cnf /etc/mysql/mysql.conf.d/zz-tuning.cnf
  sudo systemctl restart mysql
  mysql -e "SHOW VARIABLES LIKE 'slow_query_log%'; SHOW VARIABLES LIKE 'long_query_time';"
  ```

### Go pprof
すでに app が `:6060` で pprof を公開済み（`app.go`）。追加セットアップ不要。

---

## 1. nginx を alp で解析

```bash
# (1) ログをローテート（ベンチ直前に実行）
sudo truncate -s 0 /var/log/nginx/access.log

# (2) ベンチを走らせる（別端末）

# (3) 集計。URI を正規表現でまとめて、レスポンスタイム合計の降順で見る
sudo alp ltsv --file /var/log/nginx/access.log \
  --sort sum -r \
  -m '^/image/[0-9]+\.(jpg|png|gif)$,^/posts/[0-9]+$,^/@[0-9a-zA-Z_]+$,^/posts$'
```

見るポイント:
- **Sum（合計レスポンスタイム）が大きい行＝全体への寄与が最大**。まずここを潰す。
- Count（呼ばれた回数）, Avg, P99 も併せて確認。
- 静的ファイル（/image/ /css/ /js/）が上位なら nginx 直配信が効いているか要確認。

---

## 2. MySQL スロークエリを pt-query-digest で解析

```bash
# (1) スローログをローテート（ベンチ直前）
sudo truncate -s 0 /var/log/mysql/mysql-slow.log
# テーブルが詰まっているとローテートが効かない場合は MySQL に FLUSH させる:
#   mysql -e "FLUSH SLOW LOGS;"   # 8.x は SET GLOBAL で一旦 off/on でもよい

# (2) ベンチを走らせる

# (3) 集計（実行時間合計の降順で上位クエリを表示）
sudo pt-query-digest /var/log/mysql/mysql-slow.log | less
```

見るポイント:
- **Response time の割合（%）が高いクエリ＝DB のボトルネック**。
- そのクエリに `EXPLAIN` をかけてインデックスが効いているか確認:
  ```bash
  mysql isuconp -e "EXPLAIN <該当クエリ>;"
  ```
- `Rows examine` が `Rows sent` に対して極端に多い＝フルスキャン/非効率インデックスの疑い。

---

## 3. Go アプリを pprof で解析

```bash
# CPU プロファイル（ベンチ走行中に 30 秒収集）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top20
(pprof) list <関数名>   # 行単位でホットスポットを確認

# ヒープ（メモリ割り当て）
go tool pprof http://localhost:6060/debug/pprof/heap
```

見るポイント:
- CPU 上位が DB ドライバや encoding なら DB 側 or シリアライズの見直し。
- テンプレート・正規表現・ハッシュ計算などアプリ固有処理が上位なら該当箇所を最適化。

---

## 任意: 追加で入れると便利なもの

- **netdata**: CPU / メモリ / ディスク I/O / ネットワークのリアルタイム監視。どのリソースが張り付いているかの俯瞰に有用。
  `bash <(curl -Ss https://my-netdata.io/kickstart.sh)`
- **pprotein**: alp / slp / fgprof を Web UI に統合。継続計測するなら検討（導入はやや重い）。
