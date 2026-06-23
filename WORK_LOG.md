# 作業ログ

## PR#1: N+1クエリ解消 (2026-06-23)

**ブランチ:** `fix/n-plus-1-make-posts`  
**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`makePosts` 関数が投稿ごとにループしてDBクエリを個別発行していた（N+1問題）:

| クエリ | 発行回数 |
|---|---|
| `SELECT COUNT(*) FROM comments WHERE post_id = ?` | 投稿数 × 1 |
| `SELECT * FROM comments WHERE post_id = ?` | 投稿数 × 1 |
| `SELECT * FROM users WHERE id = ?` (コメントユーザー) | コメント数 × 1 |
| `SELECT * FROM users WHERE id = ?` (投稿ユーザー) | 投稿数 × 1 |

### 解決策

`sqlx.In` を使ってIN句でまとめて取得し、Go側でmapに展開して組み立てる方式に変更:

1. **コメント数**: `SELECT post_id, COUNT(*) FROM comments WHERE post_id IN (?) GROUP BY post_id`
2. **コメント一覧**: `SELECT * FROM comments WHERE post_id IN (?) ORDER BY created_at DESC` → Go側で上位3件に絞る
3. **ユーザー情報**: 投稿・コメント双方のユーザーIDをSetで集約し `SELECT * FROM users WHERE id IN (?)` で一括取得

### 効果

DBクエリ数が **O(N) → 固定4クエリ** に削減

---

## PR#2: 投稿取得クエリのフルスキャン解消 (2026-06-23)

**ブランチ:** `fix/posts-query-full-scan`  
**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

PR#1 の `makePosts` バッチ取得化が原因で、新たなボトルネックが発生:

- `getIndex` / `getPosts` / `getAccountName` の posts クエリに `LIMIT` がなく、全件（最大10000件）取得していた
- `makePosts` に全件渡されるため、`IN(10000件)` のコメント取得クエリが発生
- ベンチマーク結果: `score: 0`、GET / / GET /@user / POST /login / POST /register でタイムアウト

### 解決策

| 箇所 | 修正内容 |
|---|---|
| `getIndex` | `JOIN users … del_flg = 0` を追加して削除ユーザーをSQL側でフィルタ + `LIMIT 20` |
| `getPosts` | 同上（`created_at <=` の条件も維持） |
| `getAccountName` | `LIMIT 20`（同一ユーザーなので JOIN 不要） |

`getIndex` / `getPosts` は SQL 側で削除ユーザーをフィルタすることで、`makePosts` に渡る件数を常に最大20件に抑える。

### 効果

`makePosts` の IN() クエリ対象が **全件 → 最大20件** に削減され、タイムアウト解消

---

## PR#3: DBインデックス追加 (2026-06-23)

**対象ファイル:** `private_isu/benchmarker/sql/schema.sql`

### 問題

`posts` / `comments` テーブルに PRIMARY KEY 以外のインデックスがなく、クエリが全件スキャンになっていた。

### 追加したインデックス

| テーブル | インデックス | 対象クエリ |
|---|---|---|
| `posts` | `idx_posts_created_at (created_at DESC)` | getIndex / getPosts の ORDER BY |
| `posts` | `idx_posts_user_id_created_at (user_id, created_at DESC)` | getAccountName の WHERE user_id + ORDER BY |
| `comments` | `idx_comments_post_id_created_at (post_id, created_at DESC)` | makePosts の WHERE post_id IN + ORDER BY |
| `comments` | `idx_comments_user_id (user_id)` | getAccountName のコメント数集計 |

### 備考

スキーマファイルに直接記述することで、DB 再初期化後もインデックスが自動で適用される。

---

## PR#4: digest関数のシェル起動をcrypto/sha512に置き換え (2026-06-23)

**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`digest` 関数がパスワードハッシュ計算のたびに `/bin/bash -c "... | openssl dgst -sha512 | sed ..."` をシェル起動していた。

- `calculatePasshash` → `calculateSalt`（`digest` ×1）+ `digest` ×1 = **シェル2回起動 / リクエスト**
- POST /login・POST /register のたびに発生しタイムアウトの原因になっていた

### 解決策

Go 標準ライブラリ `crypto/sha512` を使ったインメモリ計算に置き換え:

```go
func digest(src string) string {
    h := sha512.Sum512([]byte(src))
    return fmt.Sprintf("%x", h)
}
```

- `os/exec` インポートと `escapeshellarg` 関数を削除
- `calculateSalt` / `calculatePasshash` から不要になった `ctx` 引数を削除

### 効果

POST /login・POST /register のシェルプロセス起動が **2回 → 0回** に削減

---

## PR#5: 画像をDBから読まずファイル配信に切り替え (2026-06-23)

**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`GET /image/{id}.{ext}` のたびに `SELECT * FROM posts WHERE id = ?` で **mediumblob をDB から読み込んでいた**。

- インデックスページ表示1回 = 画像20枚 = DB クエリ20本 + 大量データ転送
- 高並列時にDBへの最大の負荷源になっていた

### 解決策

1. **`getInitialize`**: 既存ファイルを削除し、DBの全画像をディスクに書き出す
2. **`postIndex`**: 新規投稿時に DB への INSERT に加えてファイルにも保存
3. **`getImage`**: DB クエリを廃止し `http.ServeFile` でファイル配信に変更
4. **`imageDir`** 定数 (`../public/image`) と **`mimeToExt`** ヘルパーを追加

### 備考

ベンチマーカーは開始時に `/initialize` を自動で叩く（`benchmarker/cli.go:250`）ため、手動でのファイル展開は不要。

### 効果

画像リクエストごとの **DB クエリがゼロ** になり、DB への負荷を大幅削減

---

## 作業メモ: インデックスが EC2 DB に未適用だった (2026-06-23)

### 発覚した問題

PR#3 でスキーマファイルにインデックスを追加したが、EC2 の DB には反映されていなかった。

**原因:** スキーマファイルへの変更は DROP TABLE → CREATE TABLE のフルリセット時にしか効かない。ベンチの `/initialize` は DELETE/UPDATE のみで DROP しないため、既存 DB にはインデックスが当たらない。

```sql
-- 確認コマンド
sudo mysql isuconp -e "
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'isuconp'
AND TABLE_NAME IN ('posts', 'comments')
ORDER BY TABLE_NAME, INDEX_NAME;"
-- → PRIMARY KEY しか存在しないことが判明
```

### 対処

EC2 で手動 ALTER TABLE を実行:

```bash
sudo mysql isuconp << 'EOF'
ALTER TABLE `posts` ADD INDEX `idx_posts_created_at` (`created_at` DESC);
ALTER TABLE `posts` ADD INDEX `idx_posts_user_id_created_at` (`user_id`, `created_at` DESC);
ALTER TABLE `comments` ADD INDEX `idx_comments_post_id_created_at` (`post_id`, `created_at` DESC);
ALTER TABLE `comments` ADD INDEX `idx_comments_user_id` (`user_id`);
EOF
```

### 教訓

インデックス追加はスキーマファイルだけでなく、**本番 DB への ALTER TABLE も必ず実行する**。

---

## PR#6: nginx静的配信 + DBコネクションプール (2026-06-23)

**ブランチ:** `fix/nginx-static-and-db-pool`
**対象ファイル:** `private_isu/webapp/etc/nginx/conf.d/default.conf`, `private_isu/webapp/golang/app.go`

### 問題

1. nginx が全リクエスト（画像・css・js含む）を Go にプロキシするだけで、PR#5 でファイル化した画像も `http.ServeFile` 経由で Go を通っていた。画像リクエストはトラフィックの大半を占めるため最大のボトルネック。
2. `main()` で DB コネクションプールを未設定。Go デフォルトの `MaxIdleConns=2` により高並列時にコネクション張り直しが多発。

### 解決策

**nginx (`default.conf`):**

| 項目 | 内容 |
|---|---|
| `/image/` | `try_files $uri @app` でファイルがあれば nginx 直接配信、無ければ app へフォールバック |
| `/(css\|js\|img\|favicon)` | 同上で静的アセットを nginx 直接配信 |
| `sendfile` / `tcp_nopush` / `open_file_cache` | 静的配信を高速化 |
| `gzip` | HTML/CSS/JS/JSON を圧縮 |
| `upstream app { keepalive 64 }` + `proxy_http_version 1.1` | app への接続を keepalive 化 |
| `expires 1d` / `Cache-Control: public` | 静的ファイルにキャッシュヘッダ |

**Go (`app.go`):**

```go
db.SetMaxOpenConns(100)
db.SetMaxIdleConns(100)
db.SetConnMaxLifetime(0)
```

### 効果

- 画像・静的アセットのリクエストが **Go を経由せず nginx で完結**し、app の負荷を大幅削減
- DB コネクションの張り直しが減り、高並列時のレイテンシが改善

### 注意

- nginx コンテナは `./public:/public` をマウント済み。Go が `../public/image` に書き出した画像は `/public/image/` として nginx から見える。EC2 直接デプロイ時は nginx の `root` が実際の public パスを指しているか要確認。

---

## 作業メモ: pprof 導入 (2026-06-23)

**対象ファイル:** `private_isu/webapp/golang/app.go`

`net/http/pprof` をブランクインポートし、ベンチに影響しないよう **ポート 6060** で別サーバーを起動。

```go
import _ "net/http/pprof"

// main() 内
go func() {
    log.Println(http.ListenAndServe(":6060", nil))
}()
```

### 使い方

ベンチ実行中に EC2 上で:

```bash
# CPUプロファイル（30秒収集）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# pprof 対話シェルで上位関数を確認
(pprof) top10
```

---

## PR#7〜#10: 画像ダンプの安定化 (2026-06-23)

**ブランチ:** `fix/nginx-static-and-db-pool`
**対象ファイル:** `private_isu/webapp/golang/app.go`

PR#6（nginx静的配信）で nginx から画像ファイルを直接配信するようにしたが、ファイル書き出し側に複数の不具合が判明し、段階的に修正した。

### 問題と修正

| 問題 | 症状 | 修正 |
|---|---|---|
| `image` ディレクトリが存在しない | `open ../public/image/9992.jpg: no such file or directory` で全書き込み失敗 | 書き出し前に `os.MkdirAll(imageDir, 0755)` |
| `/initialize` 内で全画像をリクエストコンテキストで書き出し | ベンチの initialize タイムアウトで切断され `context canceled` | 書き出しを `dumpImages(context.Background())` に切り出し、リクエスト切断の影響を受けないように |
| 毎回 全削除→全書き出し | initialize が遅い | `os.Stat` で**既存ファイルはスキップ**。ベース投稿(1〜10000)の画像は不変なので2回目以降はほぼ即時 |
| 起動直後に画像が未展開 | 初回 initialize が重い | `main()` で**起動時に一度 `dumpImages`** を実行 |

### 効果

`/initialize` のタイムアウト由来の `context canceled` を解消し、2回目以降の initialize を高速化。

---

## 作業メモ: .gitignore に生成画像を追加 (2026-06-23)

**対象ファイル:** `.gitignore`

DB から書き出される画像は実行時生成物なので、リポジトリに含めないよう除外。

```
private_isu/webapp/public/image/
```

jpg/png/gif すべてを対象にするためディレクトリごと除外。

---

## PR#11: 画像ダンプのメモリ安全化（バッチ取得） (2026-06-23)

**ブランチ:** `fix/dumpImages`
**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`dumpImages` が `SELECT id, mime, imgdata FROM posts` で**全画像（mediumblob 1万件）を一括メモリロード**していた。app コンテナは 1GB 制限のため、起動時ダンプでメモリスパイク→GC多発／スワップが起き、ベンチ実行中ずっと全体が遅くなり、各ハンドラで `context canceled` が多発していた。

### 解決策

2段階方式に変更:

1. **`SELECT id, mime FROM posts`** で blob を含まない軽いメタ情報だけ取得し、ディスクに無いファイルを洗い出す
2. 不足分の `imgdata` だけを **100件ずつバッチ**（`sqlx.In` の `IN(?)`）で取得して書き出す

### 効果

全 blob の一括メモリ確保を排除。**定常状態（全ファイル存在）では blob を1件も読まない**ため、起動時のメモリ圧迫と GC 由来の遅延を解消。

---

## PR#12: HTMLテンプレートの事前パース (2026-06-23)

**ブランチ:** `fix/precompile-templates`
**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`getIndex` / `getAccountName` / `getPosts` / `getPostsID` / `getLogin` / `getRegister` / `getAdminBanned` のすべてが、**リクエストのたびに** `template.Must(template.ParseFiles(...))` を実行していた。

- HTML テンプレートファイルを毎回ディスクから読み込み・パースしており、CPU と I/O を浪費
- 特に最頻出の GET / は毎回 4 ファイル（layout/index/posts/post）を読み込んでいた

### 解決策

テンプレートを **起動時に一度だけパース**して package 変数に保持し、ハンドラでは `Execute` のみ呼ぶ方式に変更:

```go
var templFuncMap = template.FuncMap{"imageURL": imageURL}

var (
    loginTmpl    = template.Must(template.ParseFiles(...))
    registerTmpl = template.Must(template.ParseFiles(...))
    bannedTmpl   = template.Must(template.ParseFiles(...))
    indexTmpl    = template.Must(template.New("layout.html").Funcs(templFuncMap).ParseFiles(...))
    accountTmpl  = template.Must(template.New("layout.html").Funcs(templFuncMap).ParseFiles(...))
    postsTmpl    = template.Must(template.New("posts.html").Funcs(templFuncMap).ParseFiles(...))
    postIDTmpl   = template.Must(template.New("layout.html").Funcs(templFuncMap).ParseFiles(...))
)
```

各ハンドラ内の `fmap` 定義と `template.Must(template.ParseFiles(...))` 呼び出しを削除し、対応する package 変数の `.Execute(w, ...)` に置き換えた。

### 効果

リクエストごとのテンプレート再パース（ディスク I/O + パース CPU）が **毎回 → ゼロ** に削減。最頻出エンドポイントの GET / で特に効果が大きい。

---

## PR#13: getAccountName のクエリ集約 (2026-06-23)

**ブランチ:** `fix/precompile-templates`
**対象ファイル:** `private_isu/webapp/golang/app.go`

### 問題

`getAccountName`（GET /@user）が以下の非効率なクエリを発行していた:

1. `SELECT COUNT(*) FROM comments WHERE user_id = ?`（コメント数）
2. `SELECT id FROM posts WHERE user_id = ?`（**全 post_id を Go 側に取得**）
3. 2 の結果から `?` プレースホルダ文字列を手動連結し `[]int → []any` 変換して
   `SELECT COUNT(*) FROM comments WHERE post_id IN (...)`

2 で投稿 ID を全件メモリに読み込み、3 で文字列連結・型変換していたのが無駄。

### 解決策

| クエリ | 変更前 | 変更後 |
|---|---|---|
| 投稿数 | `SELECT id` を全件取得して `len()` | `SELECT COUNT(*) FROM posts WHERE user_id = ?`（行を materialize しない） |
| 投稿へのコメント数 | アプリ側で IN プレースホルダを手動構築 | `... WHERE post_id IN (SELECT id FROM posts WHERE user_id = ?)` のサブクエリで集約 |

- `posts.user_id`（`idx_posts_user_id_created_at`）/ `comments.post_id`（`idx_comments_post_id_created_at`）の各インデックスを利用
- 手動の placeholder 連結・`[]int → []any` 変換を撤廃

### 効果

全 post_id のメモリ取得とアプリ側での文字列組み立てを排除。クエリも index-backed な集計のみになり、投稿数の多いユーザーで特に軽量化。
