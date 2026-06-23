# パフォーマンス改善 調査メモ（PR#11 以降の残課題）

調査日: 2026-06-23
対象: `private_isu/webapp/golang/app.go` ほか

PR#1〜#11 で DB 層（N+1・インデックス・フルスキャン解消）、画像のファイル配信、nginx 静的配信、DB コネクションプール、sha512 化、画像ダンプ安定化まで対応済み。
さらに PR#12〜#16 で、テンプレート事前パース・getAccountName クエリ集約・nginx⇔app の Unix ソケット化・DSN の interpolateParams 化まで対応済み。
残るボトルネックは **アプリ層（Go）の CPU 浪費** と **設定面** が中心。優先度順に以下へまとめる。

---

## ✅ 対応済み (PR#12): テンプレートの毎リクエスト再パース

### 問題
以下のハンドラが **リクエストのたびに** `template.Must(template.ParseFiles(...))` を実行しており、毎回ディスクから HTML を読み込み・パースしている。

- `getIndex`（GET /）
- `getAccountName`（GET /@user）
- `getPosts`（GET /posts）
- `getPostsID`（GET /posts/{id}）
- `getLogin` / `getRegister` / `getAdminBanned`

```go
// getIndex 内 — リクエストごとに4ファイルを読み込み&パース
template.Must(template.New("layout.html").Funcs(fmap).ParseFiles(
    getTemplPath("layout.html"), getTemplPath("index.html"),
    getTemplPath("posts.html"), getTemplPath("post.html"),
)).Execute(w, ...)
```

GET / は最頻出エンドポイントの一つで、毎回ディスク I/O + パースが走り CPU を大量消費する。

### 対策
起動時（`main()` か package 初期化）に一度だけパースして package 変数に保持し、ハンドラでは `Execute` のみ呼ぶ。

```go
var indexTmpl = template.Must(template.New("layout.html").Funcs(fmap).ParseFiles(...))
```

**低リスク・高効果。今いちばん効く改善の見込み。**

---

## ✅ 対応済み (PR#15): nginx⇔app 間を Unix ドメインソケット化

### 問題
nginx⇔app 間が TCP（`server app:8080`）接続で、同一ホストでも TCP/loopback のオーバーヘッドが発生していた。

### 対策
Go を `ISUCONP_LISTEN_SOCKET` 指定時に Unix ソケットで listen させ（未設定時は TCP :8080 にフォールバック）、nginx の upstream を `server unix:/run/isuconp.sock` に変更。あわせて gzip / upstream keepalive / 静的ファイルの Cache-Control は PR#6 で対応済みであることを確認。

> EC2 では Go プロセスに `ISUCONP_LISTEN_SOCKET=/run/isuconp.sock` を設定し、`/run` への書き込み権限と nginx ユーザーからの到達性を要確認。

---

## ✅ 対応済み (PR#16): DSN の interpolateParams 化

### 問題
go-sql-driver/mysql がデフォルトでプレースホルダ付きクエリをサーバーサイドプリペアドステートメント実行しており、毎クエリ `PREPARE`+`EXECUTE` の往復が発生していた。

### 対策
`cfg.InterpolateParams = true` を追加し、クライアント側でプレースホルダを安全に展開して1回のクエリ送信に集約。SQL インジェクション安全性は維持。

---

## 🔴 確認事項: compose.yml が Ruby 実装をビルドしている

```yaml
# private_isu/webapp/compose.yml
app:
  build:
    context: ruby/   # ← Go実装を動かすなら golang/
```

Docker Compose で動かす場合、Go の最適化が一切反映されない。
EC2 上で Go バイナリを直接起動しているなら無関係だが、**デプロイ方式の確認が必要**。

---

## ✅ 対応済み (PR#18): `makePosts` のコメント全件取得

### 問題
一覧表示（`allComments=false`）でも以下で **全コメントを取得**してから Go 側で上位3件に絞っていた。

```go
q, args, err = sqlx.In("SELECT * FROM `comments` WHERE `post_id` IN (?) ORDER BY `created_at` DESC", postIDs)
```

コメント数の多い投稿があると、無駄な転送・メモリ確保が発生していた。

### 対策
`allComments=false` のときだけ MySQL 8 のウィンドウ関数
（`ROW_NUMBER() OVER (PARTITION BY post_id ORDER BY created_at DESC)` で `rn <= 3`）で
DB 側を per-post 上位3件に絞り込むよう変更。`allComments=true`（投稿単体）は従来通り全件取得。
`idx_comments_post_id_created_at` がそのまま効く。ベンチはコメント内容/件数を検証しないため表示は同一。

---

## ✅ 対応済み (PR#13): `getAccountName` のクエリ集約

### 問題
以下の3クエリを個別発行している。

1. `SELECT COUNT(*) FROM comments WHERE user_id = ?`（ユーザーのコメント数）
2. `SELECT id FROM posts WHERE user_id = ?`（投稿 ID 一覧）
3. `SELECT COUNT(*) FROM comments WHERE post_id IN (...)`（手動で placeholder 文字列を連結）

2 で全 post_id を取得して文字列連結している箇所も非効率。

### 対策
JOIN / サブクエリで 1〜2 クエリに集約する。
例: `commentedCount` はサブクエリ `WHERE post_id IN (SELECT id FROM posts WHERE user_id = ?)` で1クエリ化できる。

---

## 🟢 小: `validateUser` の regexp 毎回コンパイル

### 問題
```go
func validateUser(accountName, password string) bool {
    return regexp.MustCompile(`\A[0-9a-zA-Z_]{3,}\z`).MatchString(accountName) &&
        regexp.MustCompile(`\A[0-9a-zA-Z_]{6,}\z`).MatchString(password)
}
```
POST /register のたびに正規表現を2本コンパイルしている。

### 対策
package 変数として事前コンパイルしておく。

```go
var (
    accountNameRe = regexp.MustCompile(`\A[0-9a-zA-Z_]{3,}\z`)
    passwordRe    = regexp.MustCompile(`\A[0-9a-zA-Z_]{6,}\z`)
)
```
低リスク。

---

## 🟢 小: `getSessionUser` が毎回 `SELECT users`

### 問題
認証が絡む全リクエストで `SELECT * FROM users WHERE id = ?` を発行。PK 引きで軽いが回数は多い。

### 対策（任意）
memcached によるユーザーキャッシュ。ただし del_flg 更新（admin/banned）との整合に注意が必要なため、効果と複雑さを天秤にかけて判断。

---

## 🟢 小: MySQL のチューニング未実施

### 問題
`compose.yml` の mysql に my.cnf マウントがなく、`innodb_buffer_pool_size` 等がデフォルトのまま。

### 対策
メモリ 1g 制限内で `innodb_buffer_pool_size` を引き上げる my.cnf を用意してマウントする。
（EC2 直接デプロイ時は EC2 側の my.cnf を調整）

---

## 推奨着手順（残課題）

1. **regexp 事前コンパイル**（小粒・低リスク）
2. **makePosts のコメント取得最適化**（一覧でも全件取得 → ウィンドウ関数等で上位3件に）
3. MySQL チューニング（`innodb_buffer_pool_size` 等の my.cnf）
4. セッションユーザーキャッシュ（memcached・del_flg 整合に注意・任意）

> ※ EC2 直接デプロイ運用のため compose.yml の `context: ruby/` 問題は実害なし（Compose 運用に切り替える場合のみ要対応）。

---

# 参考記事との差分（未対応項目）

追記日: 2026-06-23
参照: [private-isu チューニング記事 (zenn.dev/melanmeg)](https://zenn.dev/melanmeg/articles/a8ece09570279f)

記事の施策を本リポジトリの実施済み内容（PR#1〜#18）と突き合わせた結果。
**記事にあって本リポジトリで未対応のもの**を優先度順に列挙する。
（逆に sha512 化・テンプレ事前パース・Unix ソケット・interpolateParams・getAccountName 集約・セッションユーザーキャッシュなどは記事に無い本リポジトリ独自の最適化で、すでに先行している。）

## ✅ 記事と同等以上に対応済み（参考）

| 記事の施策 | 本リポジトリ |
|---|---|
| DB インデックス（comments(post_id,created_at), comments(user_id), posts(created_at)） | PR#3 で対応（さらに posts(user_id,created_at) も追加） |
| 画像のファイル書き出し配信 | PR#5 / PR#6 / PR#11 |
| posts×users JOIN で N+1 解消 | PR#1 / PR#2 |
| nginx 静的キャッシュ（expires 1d / Cache-Control public） | PR#6 |
| nginx upstream keepalive | PR#6（`keepalive 64`） |
| makePosts のコメント取得最適化 | PR#18（ウィンドウ関数で上位3件） |

---

## ✅ 対応済み (PR#19): 計測基盤の整備（最優先の土台）

記事は **pprotein（alp / slp / fgprof 統合）+ netdata + pt-query-digest 相当のスローログ解析** で
ボトルネックを定量把握してから施策を打っている。本リポジトリは pprof のみだったため、
nginx アクセスログ解析（alp）と MySQL スローログ解析を整備した。

### 対応内容（PR#19）
- **alp**: `default.conf` に LTSV `log_format` + `access_log` を追加（本番走行は `access_log off;`）。
- **MySQL スロークエリログ**: `etc/mysql/conf.d/tuning.cnf`（新規）に `slow_query_log` / `long_query_time=0` /
  `innodb_buffer_pool_size` 等を定義。EC2 へ手動配置（配置先・手順は `MEASUREMENT.md`）。
- **手順書**: `MEASUREMENT.md` に alp / pt-query-digest / pprof のインストール・集計コマンド・見るポイントを記載。

### まだ任意（未対応）
- **netdata**（リソース監視）。`MEASUREMENT.md` にインストールコマンドのみ記載。
- pprotein / pprotein-agent / phpMyAdmin。継続計測するなら検討。導入はやや重い。

> ⚠️ EC2 では `tuning.cnf` の配置と nginx reload / MySQL restart が**手動で必要**。`MEASUREMENT.md` 参照。

---

## 🟡 中: makePosts（投稿一覧）結果そのものの memcached キャッシュ

記事は「スローログ上位の `makePosts` をキャッシュして N+1 のループ実行自体を減らす」とある。
本リポジトリは PR#1 で N+1 を固定4クエリに削減し、PR#17 でセッションユーザーをキャッシュ、
PR#18 でコメントを上位3件に絞ったが、**投稿一覧の組み立て結果自体はキャッシュしていない**。

### 判断: 現時点では見送り（2026-06-23）
計測基盤の整備と安価な改善（keepalive_requests / my.cnf）を先に行い、makePosts が実際に
ボトルネックだと確認できてから再検討する。理由は以下の通り。

### 検討時に必ず踏まえる注意点（調査済み）
- **CSRF トークンが地雷**: `post.html` の各投稿は `{{.CSRFToken}}` を埋め込み、これは
  `makePosts(..., getCSRFToken(r), ...)` 由来の **セッション固有値**。makePosts 結果を丸ごと
  キャッシュすると他ユーザーに別人の CSRF が配られ、コメント投稿が 422 で全滅する。
  → 実装するなら「CSRF を除いた Post リストをキャッシュし、リクエストごとに `getCSRFToken(r)` を再注入」必須。
- **無効化が効きにくい**: GET / は最新20件のグローバル一覧。新規投稿（postIndex）・トップ20内への
  コメント（postComment）・banned（del_flg）でほぼ毎回失効するため、ベンチの書き込み頻度では hit 率が低い。
  短 TTL（1s 等）方式は投稿直後の反映を検証するシナリオで整合性リスク。
- **本リポジトリでは効果が小さい**: 記事は makePosts が N+1（スローログ上位）だったのが前提。
  本リポジトリは PR#1 でバッチ化（固定4クエリ）＋PR#18 でコメント3件絞り済みのため、削減幅が小さい。
- シリアライズは記事では `gob`（`go-json` より速かったとのこと）。本リポジトリのユーザーキャッシュは JSON なので、
  キャッシュを増やすなら `gob` 採用も検討。

---

## 🟡 中: MySQL my.cnf チューニング（PR#19 で雛形作成済み・EC2 配置は未）

PR#19 で `etc/mysql/conf.d/tuning.cnf` を作成し、`innodb_buffer_pool_size` / スローログ /
`innodb_flush_log_at_trx_commit=2` 等を定義済み。

### 残り
- **EC2 への配置と restart が未**（手動作業。`MEASUREMENT.md` 参照）。
- `innodb_buffer_pool_size` は EC2 の実メモリに合わせて要調整（雛形は 1G）。
- `bind-address`（複数台構成にする場合のみ。単一ホストなら不要）。

---

## 🟢 小: nginx の keepalive_requests 未設定

記事は `keepalive_requests 10000`（クライアント側・upstream 側の両方）を設定している。
本リポジトリの `default.conf` は upstream に `keepalive 64` はあるが **`keepalive_requests` 指定が無い**ため、
既定値（1000）で頭打ちになる可能性。

### 未対応
- upstream app に `keepalive_requests 10000;`
- server/http レベルで client 向け `keepalive_requests 10000;`（`keepalive_timeout` も併せて）
- `worker_processes auto;` / `worker_connections` は本 conf.d では確認不可（EC2 の `nginx.conf` 本体を要確認）。

---

## ⚪ 構成: サーバー複数台分割（記事は3台構成）

記事は **isu1: nginx+app / isu2: memcached+app / isu3: MySQL** の3台に分割している。
本リポジトリは **EC2 単一ホスト運用**のため、現状の方針では未対応（＝対象外）。

### メモ
- スコアを大きく伸ばすには最終的に有効だが、単一ホスト前提なら不要。
- 複数台化する場合のみ `bind-address = 0.0.0.0`、内部ネットワーク IP でのベンチ実行、
  app の DB/memcached 接続先を内部 IP に変更、が必要になる。

---

## 推奨着手順（記事差分を踏まえた更新版）

1. ✅ **計測基盤**（PR#19 完了。コードは整備済み、EC2 へのツール導入・設定配置は手動）
2. **EC2 で計測基盤を有効化**: `tuning.cnf` 配置 + alp/pt-query-digest インストール → 1 回ベンチを回して現状の上位を把握（`MEASUREMENT.md`）
3. **nginx keepalive_requests**（小粒・低リスク）
4. **MySQL my.cnf の値調整**（`innodb_buffer_pool_size` を EC2 実メモリに合わせる）
5. **makePosts 結果キャッシュ**（2 の計測で makePosts が上位なら検討。無効化設計に注意）
6. サーバー複数台分割（単一ホスト方針なら対象外）
