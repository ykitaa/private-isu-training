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

## 🟡 中: `makePosts` のコメント全件取得

### 問題
一覧表示（`allComments=false`）でも以下で **全コメントを取得**してから Go 側で上位3件に絞っている。

```go
q, args, err = sqlx.In("SELECT * FROM `comments` WHERE `post_id` IN (?) ORDER BY `created_at` DESC", postIDs)
```

コメント数の多い投稿があると、無駄な転送・メモリ確保が発生する。

### 対策
- MySQL 8 のウィンドウ関数（`ROW_NUMBER() OVER (PARTITION BY post_id ORDER BY created_at DESC)`）で per-post 上位3件に絞る、
- または件数集計（`commentCountMap`）と組み合わせて取得件数を抑える。

実装はやや複雑。中リスク。

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
