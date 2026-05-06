# x-cli

X (Twitter) コマンドラインツール。Rustで記述され、単一のバイナリ `x` としてコンパイルされます。ツイートの読み書きを含む、包括的な操作をサポートしています。

すべての出力は**軽量なJSON**形式です（GraphQLの生のレスポンスから主要なフィールドを自動抽出し、データ量を約98%削減しています）。

## インストール

### 方法 1：ワンクリックインストール（推奨）

システムとアーキテクチャを自動検出し、対応するバイナリをダウンロードします：

```bash
curl -fsSL https://raw.githubusercontent.com/haloowhite/twitter-cli/main/install.sh | bash
```

インストールディレクトリをカスタマイズする場合：

```bash
INSTALL_DIR=~/bin curl -fsSL https://raw.githubusercontent.com/haloowhite/twitter-cli/main/install.sh | bash
```

### 方法 2：手動ダウンロード

[GitHub Releases](https://github.com/haloowhite/twitter-cli/releases) から、お使いのプラットフォームに対応するパッケージをダウンロードしてください：

| プラットフォーム | ファイル名 |
|------|------|
| Windows x64 | `x-windows-amd64.zip` |
| Linux x86_64 | `x-linux-amd64.tar.gz` |
| Linux ARM64 | `x-linux-arm64.tar.gz` |
| macOS Intel | `x-darwin-amd64.tar.gz` |
| macOS Apple Silicon | `x-darwin-arm64.tar.gz` |

**Windowsでのインストール：**
`x-windows-amd64.zip` をダウンロードし、解凍した `x.exe` をシステムパス（PATH）の通ったディレクトリ（例：`C:\Windows\System32` や任意のフォルダ）に配置してください。

### 方法 3：ソースからビルド

[Rust ツールチェーン](https://rustup.rs/) が必要です：

**Unix (Linux/macOS):**
```bash
git clone https://github.com/haloowhite/twitter-cli.git
cd twitter-cli
cargo build --release
sudo cp target/release/x /usr/local/bin/
```

**Windows:**
Windowsでのビルドには [NASM](https://www.nasm.us/)（TLS指紋ライブラリのコンパイルに使用）のインストールが必要です。
```powershell
git clone https://github.com/haloowhite/twitter-cli.git
cd twitter-cli
# NASMがPATHに含まれていることを確認してください
cargo build --release
# target/release/x.exe をPATHの通ったディレクトリにコピーしてください
```

## 認証

認証情報は `~/.x-cli/credentials.json` に保存されます。形式は以下の通りです：

```json
{
  "auth_token": "あなたのauth_token",
  "ct0": "自動生成されるcsrf_token",
  "extra_cookies": "完全なcookie文字列（書き込み操作に必須。不足すると226エラーが発生します）"
}
```

### ケース 1：ローカルPCのブラウザでログイン済みの場合（最も簡単）

```bash
x auth --browser chrome   # サポート：chrome, firefox, edge, safari
```

### ケース 2：エージェントがクラウド上にあり、ブラウザがローカルPCにある場合

**ステップ 1：ローカルPCで完全なCookieを取得する**

Chrome/Edge/Firefoxで x.com にログインし、以下の操作を行います：

1. `F12` キーを押してデベロッパーツールを開く
2. **Network**（ネットワーク）タブをクリック
3. ページをリフレッシュするか閲覧して、リクエストを表示させる
4. `x.com` 宛の任意のリクエストをクリック
5. 右側の **Request Headers** → **Cookie:** の行を探す
6. **右クリック → 値をコピー** し、完全なCookie文字列を取得する

**ステップ 2：Cookieからフィールドを抽出して設定する**

```bash
# 完全なCookie文字列から auth_token と ct0 を抽出
FULL_COOKIE="<コピーした完全なCookie文字列を貼り付け>"
AUTH_TOKEN=$(echo "$FULL_COOKIE" | grep -oE 'auth_token=[a-f0-9]+' | cut -d= -f2)
CT0=$(echo "$FULL_COOKIE" | grep -oE 'ct0=[a-f0-9]+' | cut -d= -f2)

mkdir -p ~/.x-cli
cat > ~/.x-cli/credentials.json << CREDENTIALS
{
  "auth_token": "$AUTH_TOKEN",
  "ct0": "$CT0",
  "extra_cookies": "$FULL_COOKIE"
}
CREDENTIALS
chmod 600 ~/.x-cli/credentials.json
```

> **⚠️ 重要**：書き込み操作（ポスト、返信、引用リポスト）には**完全なCookie**（`extra_cookies` フィールド）が**必須**です。`auth_token` + `ct0` だけでは 226 エラー（"looks like automated behavior"）が発生します。読み取り操作（タイムライン閲覧、検索など）は影響を受けません。

### ケース 3：モバイル端末のみ、またはエージェントがクラウドにある場合

> **⚠️ 注意**：以下の方法で取得したCookieは不完全な場合があり（モバイルブラウザの制限）、読み取り専用となる可能性があります。書き込み操作が必要な場合は、PCブラウザから取得することをお勧めします（ケース 2）。

**方法 A：モバイルブラウザ + JavaScript**

1. モバイルブラウザ（Chrome/Safari）で https://x.com にアクセスしログインする
2. アドレスバーに以下を入力して実行する（`javascript:` 接頭辞は手動で入力する必要があります。貼り付けはできません）：

```
javascript:void(document.title=document.cookie)
```

3. ページタイトルがCookie文字列に変わるので、**完全なCookie文字列をコピー**する
4. コピーした文字列をエージェントに渡し、ケース 2 のステップ 2 の手順で設定する（`extra_cookies` を含む）

**方法 B：モバイルブラウザ DevTools (Android Chrome)**

1. モバイル Chrome で x.com にアクセスしログインする
2. PC の Chrome で `chrome://inspect/#devices` を開き、モバイル端末を接続する
3. リモートデバッグ画面の Console で以下を実行する：

```javascript
document.cookie
```

4. 出力された**完全なCookie文字列**をコピーする

**方法 C：パケットキャプチャ (iOS/Android 共通)**

1. パケットキャプチャツール（例：Stream/HTTP Catcher/Charles）をインストールする
2. X/Twitter アプリを開き、適当に閲覧する
3. キャプチャ記録から `api.x.com` または `x.com` 宛のリクエストを探す
4. リクエストヘッダー内の**完全な `Cookie` 値**をコピーする

**方法 D：設定ファイルを直接転送する**

既に認証済みのデバイスから設定を書き出し、クラウドに転送します：

```bash
# 認証済みデバイスで
cat ~/.x-cli/credentials.json
# 出力内容をコピー

# クラウドサーバーで
mkdir -p ~/.x-cli
cat > ~/.x-cli/credentials.json << 'EOF'
（コピーした JSON 内容を貼り付け）
EOF
chmod 600 ~/.x-cli/credentials.json
```

### 注意事項

- **書き込み操作には完全なCookie（`extra_cookies` フィールド）が必須です**。不足すると 226 エラーが発生します。
- `auth_token` はログイン情報そのものです。**他人に教えないでください**。
- `auth_token` の有効期限は通常数ヶ月と長いですが、期限が切れた場合は再取得が必要です。
- パスワードを変更すると、すべての `auth_token` が無効になります。
- `ct0` は省略可能です。ツールがランダムな値を自動生成します。
- `credentials.json` には `chmod 600` 等で適切な権限を設定することを推奨します。

## 出力形式

すべてのコマンドは精簡された JSON を出力します。

ツイートの例：
```json
{
  "id": "2030159267689632121",
  "url": "https://x.com/elonmusk/status/2030159267689632121",
  "text": "Only Grok speaks the truth...",
  "created_at": "Sat Mar 07 05:51:02 +0000 2026",
  "lang": "en",
  "author": { "id": "44196397", "handle": "elonmusk", "name": "Elon Musk" },
  "stats": { "views": 25806169, "likes": 58482, "retweets": 10789, "replies": 10750, "quotes": 967, "bookmarks": 4533 },
  "referenced_tweet": { "id": "2030151922968318104", "type": "quote" }
}
```

ユーザーの例：
```json
{
  "id": "44196397",
  "screen_name": "elonmusk",
  "name": "Elon Musk",
  "description": "",
  "followers_count": 236140595,
  "following_count": 1292,
  "tweet_count": 98635,
  "is_verified": true,
  "created_at": "Tue Jun 02 20:12:29 +0000 2009",
  "profile_image_url": "https://pbs.twimg.com/..."
}
```

## コマンドリファレンス

スクリーンネーム（例：`elonmusk`）またはユーザーIDをサポートしています。

### 読み取り操作

```bash
x me                                    # 現在のユーザー情報を表示
x user elonmusk                         # ユーザー情報を表示
x timeline --limit 20                   # タイムラインを表示
x tweets elonmusk --limit 50            # ユーザーのツイートを表示
x replies elonmusk --limit 20           # ユーザーの返信を表示
x followers elonmusk --limit 100        # フォロワー一覧を表示
x following elonmusk --limit 100        # フォロー中一覧を表示
x search "rust lang" --limit 30         # ツイートを検索
x detail 1234567890                     # ツイート詳細を表示
x detail 1234567890 --context           # 前後の文脈を含めて詳細を表示
```

### コンパクトモード

`-c` フラグを付けると出力を最小限にし、主要なフィールドのみを表示します（LLM やパイプ処理に最適です）：

```bash
x -c timeline                           # コンパクトなタイムライン
x -c tweets elonmusk --limit 10         # コンパクトなツイート
x -c search "AI" --limit 20             # コンパクトな検索
```

### 書き込み操作

```bash
x post "Hello from x-cli!"             # ツイートを投稿
x reply 1234567890 "Great tweet!"       # 返信を投稿
x quote 1234567890 "Interesting"        # 引用リポスト
x like 1234567890                       # いいね
x unlike 1234567890                     # いいね解除
x retweet 1234567890                    # リポスト
x unretweet 1234567890                  # リポスト解除
x follow elonmusk                       # フォロー
x unfollow elonmusk                     # フォロー解除
```

## jq との組み合わせ

```bash
x tweets elonmusk --limit 5 | jq '.[].text'                    # ツイート本文のみ抽出
x tweets elonmusk | jq 'sort_by(.stats.likes) | last.url'      # 最もいいねが多いツイートのURL
x search "AI" --limit 10 | jq '[.[] | select(.stats.likes > 100)]' # 100いいね以上のツイートをフィルタ
x user elonmusk | jq '.followers_count'                         # フォロワー数のみ抽出
```

## トラブルシューティング

| エラー内容 | 解決策 |
|------|----------|
| `No credentials found` | `x auth --browser chrome` を実行してください |
| 書き込み操作での 226 エラー | **完全なCookieが必須です**：ブラウザの Network タブから完全な Cookie 文字列をコピーし、credentials.json の `extra_cookies` フィールドに設定してください |
| 404 エラー | `~/.x-cli/transaction_cache.json` を削除して再試行してください |

## 詳細な使い方

詳細は [skill.md](skill.md) を参照してください。

## 技術仕様

- rquest (Chrome TLS 指紋) + reqwest (フォールバック) のデュアル HTTP クライアント
- `x-client-transaction-id` の自動抽出
- GraphQL レスポンスから主要フィールドを自動抽出（データ量を 98% 削減）
