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
