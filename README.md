# PowerCMS X Docker 開発環境

PowerCMS X をローカルで動かすための Docker Compose 構成です。

## 構成

| サービス | イメージ | URL / ポート |
|---|---|---|
| Apache | `httpd:2.4-bookworm` | http://localhost/ |
| PHP-FPM | `php:${PHP_VERSION}-fpm-bookworm` | – |
| Cron | php-fpm イメージを流用 | – |
| MySQL | `mysql:8.4` | localhost:3306 |
| Redis | `redis:7-alpine` | localhost:6379 |
| phpMyAdmin | `phpmyadmin` | http://localhost:8080/ |

### ディレクトリ

| パス（ホスト） | コンテナ内 | 内容 |
|---|---|---|
| `./src/` | `/var/www/powercmsx` | PowerCMS X アプリケーション本体 |
| `./html/` | `/var/www/html` | 公開済み静的 HTML（Apache DocumentRoot） |

### php-fpm に含まれるライブラリ

| ソフトウェア | 説明 |
|---|---|
| QDBM 1.8.78 (Alfasado gcc patched) | 全文検索用 DBM |
| Hyper Estraier 1.4.13 | 全文検索エンジン（MeCab 対応） |
| MeCab + IPAdic | 形態素解析 |
| Composer 2 | PHP パッケージ管理 |
| Node.js + pnpm | フロントエンドビルドツール（AxeRunnerプラグインで使用） |
| ExifTool | 画像・動画メタデータ取得 |
| Poppler (pdfinfo / pdftotext) | PDF 情報取得・テキスト抽出 |
| supercronic | cron コンテナ用デーモン |

PHP 拡張: `bcmath`, `calendar`, `exif`, `gd`, `intl`, `mbstring`, `mysqli`, `opcache`, `pdo`, `pdo_mysql`, `soap`, `zip`, `redis`, `mecab`

> OPcache はコード変更を即反映するため無効化しています（`opcache.enable = 0`）。

---

## セットアップ

### 1. 環境変数ファイルを作成

```bash
cp .env.example .env
```

`.env` を開いてパスワードを設定してください。`.env` は `.gitignore` で除外されているためコミットされません。

### 2. ソースを配置

```bash
# PowerCMS X 本体を src/ に配置
cp -r /path/to/powercmsx/* src/
```

`src/db-config.php` を以下の内容に変更し、DB 接続情報を環境変数から取得するようにしてください：

```php
<?php
define( 'PADO_DB_NAME',     getenv( 'DB_NAME' )     ?: 'powercmsx' );
define( 'PADO_DB_HOST',     getenv( 'DB_HOST' )     ?: 'localhost' );
define( 'PADO_DB_USER',     getenv( 'DB_USER' )     ?: 'powercmsx' );
define( 'PADO_DB_PASSWORD', getenv( 'DB_PASSWORD' ) ?: '' );
define( 'PADO_DB_PORT',     getenv( 'DB_PORT' )     ?: '3306' );
```

また、`src/config.json` の `config_settings` に以下を追加してください：

```json
"cfg_admin_url" : "http://localhost/powercmsx/index.php",
"temp_dir" : "/var/www/tmp",
"work_dir" : "/var/www/work",
```

`config_settings` で以下のようにすると Redis も利用できます。

```json
"cache_driver" : "Redis",
"memcached_servers": [
    "redis:6379"
],
```

### 3. ビルド・起動

```bash
# 初回ビルド（QDBM・HyperEstraier・MeCab のコンパイルがあるため時間がかかります）
docker compose up --build -d

# ログ確認
docker compose logs -f
```

### 3. アクセス

| URL | 内容 |
|---|---|
| http://localhost/ | 静的 HTML（`./html/`） |
| http://localhost/powercmsx/index.php | PowerCMS X 管理画面 |
| http://localhost:8080/ | phpMyAdmin |

---

## 日常操作

```bash
# 起動
docker compose up -d

# 停止
docker compose down

# ボリュームも含めて削除（DB・ファイルも消えます）
docker compose down -v

# ログ
docker compose logs -f [サービス名]

# php.ini などを変更した後の再ビルド
docker compose build php-fpm && docker compose up -d php-fpm
```

---

## PHP バージョンの切り替え

`.env` の `PHP_VERSION` を変更して再ビルドするだけです。

```bash
# .env を編集
PHP_VERSION=8.2

# 再ビルド・再起動
docker compose up --build -d
```

複数バージョンをローカルに保持しておきたい場合は、バージョンごとにビルドしておきます。

```bash
PHP_VERSION=8.1 docker compose build
PHP_VERSION=8.2 docker compose build
PHP_VERSION=8.3 docker compose build
```

切り替えは `.env` の `PHP_VERSION` を書き換えて `docker compose up -d` するだけです（再ビルド不要）。

> **注意:** `-fpm-bookworm` タグは PHP 8.1 以降で利用可能です。8.0 以前は `Dockerfile` の `-bookworm` を `-bullseye` に手動で書き換えてください。

## Node.js バージョンの切り替え

`.env` の `NODE_VERSION` を変更して再ビルドします。

```bash
NODE_VERSION=20
docker compose up --build -d
```

---

## cron

`docker/cron/crontab` を編集してください。変更後は再起動が必要です。

```bash
docker compose restart cron
docker compose logs -f cron
```

## 補足

### Apache バージョンを取得するタグの修正

Apache と php-fpm が別コンテナのため、下記のようにタグを修正する必要があります。

```
<mt:property name="apache_version" setvar="apache_version">
   ↓
<mt:setvar name="apache_version" value="24">
```

### config.json に記載する各種パス情報

```
"vendor_autoload": "/var/www/vendor-src/vendor/autoload.php",
"exiftool_path": "/usr/bin/exiftool",
"mecab_path": "/usr/bin/mecab",
"mecab_dic_path": "/var/lib/mecab/dic/debian",
"mecab_dict_index_path": "/usr/lib/mecab/mecab-dict-index",
"pdfinfo_path": "/usr/bin/pdfinfo",
```
