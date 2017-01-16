# DrupalOnDocker

DrupalOnDockerにはDrupalの開発を素早く行うためのDockerのコンテナーセットが含まれています。
開発マシンのスペックに応じて3種類のコンフィグが用意されており、5分から10分程度でDrupalの環境をDocker上に構築することができます。

## 概要

以下のコンテナー郡で構成されています。

| Container | Service name | Image | Exposed port |
| --------- | ------------ | ----- | ------------ |
| Nginx | web | <a href="https://hub.docker.com/_/nginx/" target="_blank">nginx</a> | 80 |
| MariaDB | db | <a href="https://hub.docker.com/_/mariadb/" target="_blank">mariadb</a> | 3306 |
| PHP-FPM 5.6 / 7.0 | php | <a href="https://hub.docker.com/r/blauerberg/drupal-php/" target="_blank">blauerberg/drupal-php</a> | 9000 (for Xdebug) |

## 動作環境

- macOS Sierra上で動作する[Docker for MAC](https://docs.docker.com/docker-for-mac/)の最新版
- Windows 10上で動作する[Docker for Windows](https://docs.docker.com/docker-for-windows/)の最新版
- Linux上で動作する[Docker engine](https://docs.docker.com/engine/installation/linux/ubuntulinux/)の最新版
- Docker for Windowsを使う場合は、 [共有ドライブ](https://blogs.msdn.microsoft.com/stevelasker/2016/06/14/configuring-docker-for-windows-volumes/) を有効にしてください。


## Getting started

### コンテナーを起動する

初めにレポジトリをクローンします。
```bash
$ git clone https://github.com/blauerberg/drupal-on-docker.git
$ cd drupal-on-docker
```

レポジトリの中には開発マシンのスペックに応じた3種類のコンフィグが用意されています。

- [tiny](https://github.com/blauerberg/drupal-on-docker/tree/master/tiny): メモリが8GB以下のマシン向け
- [standard](https://github.com/blauerberg/drupal-on-docker/tree/master/standard): メモリが16GB以下のマシン向け
- [huge](https://github.com/blauerberg/drupal-on-docker/tree/master/huge): 16GB以上のメモリを持つマシン向け

例えば、メモリが8GBのWindowsもしくはmacOSを使っている場合は、`tiny` を利用すると良いでしょう。
```bash
$ cd tiny
```

次にDrupalのソースコードをマウントするためのディレクトリを作成します。
デフォルトの設定では、ホストマシンの `volumes/drupal` をコンテナーのデータボリュームとしてマウントします。:w

```bash
$ mkdir -p volumes/drupal
```

Drupalのソースコードをダウンロードして展開します。
```bash
# Note: replace "X.Y.Z" in below to  Drupal's version you'd like to use.
$ curl https://ftp.drupal.org/files/projects/drupal-X.Y.Z.tar.gz | tar zx --strip=1 -C volumes/drupal
```

macOSを使っている場合、[パフォーマンスの問題](https://github.com/docker/for-mac/issues/77) を回避するために [docker-sync](https://github.com/EugenMayer/docker-sync/) を利用することを強く推奨します。
[Use docker-sync](#docker-sync-を使う) を参照してください。

コンテナーを生成して起動します。
```bash
$ docker-compose up -d
```

ホストマシンがLinuxの場合は、ディレクトリの権限を修正する必要があります。
```bash
$ docker-compose exec php chown -R www-data:www-data /var/www/html/sites/default
```

Drupalにアクセスします。
```bash
$ open http://localhost # もしくはブラウザで http://localhost へアクセス
```

Note: `docker-compose` コマンドは `docker-compose.yml` があるディレクトリ内で実行する必要があります。

### Drupalのインストール

データベースの認証情報は `docker-compose.yml` で設定されています。
デフォルト値は以下になります。

- Database Name: `drupal`
- Username: `drupal`
- Password: `drupal`

詳細については https://hub.docker.com/_/mariadb/ の "Environment Variables" を参照してください。

このコンテナーセットではnginx、mariadb、php-fpmは全て別々のコンテナーで動作します。
そのため、インストール時にデータベースサーバーのホスト名に `localhost` ではなく `db` を指定する必要があります。

ブラウザからウィザードでインストールする代わりに、以下のようにDrushを使ってインストールすることもできます。

```bash
$ docker-compose exec php drush -y --root="/var/www/html" site-install standard --site-name="Drupal on Docker" --account-name="drupal" --account-pass="drupal" --db-url="mysql://drupal:drupal@db/drupal" --locale=ja
```

## コンテナーを停止する

```
$ docker-compose stop
```

## Other tips

### コンテナの内部にアクセスする

sshではなく `docker-compose exec` を使いましょう。

```bash
$ docker-compose exec {Service name} /bin/bash
# ex. docker-compose exec php /bin/bash
```

### Drushを使う

Drushは`php`コンテナーにインストールされています。

```bash
$ docker-compose exec php drush st
```

### データベースに接続する

Drush経由で接続する:
```bash
$ docker-compose exec php drush sql-cli
```

データベースコンテナーは 127.0.0.1上でport 3306をlistenします。そのため、[MysqlWorkbench](https://www.mysql.com/products/workbench/) や [Sequel Pro](https://www.sequelpro.com/) のようなホストOS上で動作するGUIアプリケーションからコンテナー内のデータベースに接続することができます。

### docker-sync を使う

macOSを使っている場合、[パフォーマンスの問題](https://github.com/docker/for-mac/issues/77) を回避するために [docker-sync](https://github.com/EugenMayer/docker-sync/) を利用することを強く推奨します。次のコマンドでインストールすることができます。
```bash
$ gem install docker-sync
$ brew install fswatch
```

また、docker-sync を使う場合は `docker-compose.yml` を少し書き換える必要があります。

- `volumes_from` ブロックをコメントアウトする (2箇所):
```
# volumes_from:
# - datastore
```

- `drupal-source` ブロックのコメントアウトを解除する (2箇所):

```
# Replace volume to this to use docker-sync for mac OS users to resolve performance issue.
# See also: https://github.com/docker/for-mac/issues/77
- drupal-source:/var/www/html:rw
```

- 最下部にある `volumes` ブロックのコメントアウトを解除する
```
volumes:
  drupal-source:
    external: true
```

`docker-sync` コマンドで同期を開始します。
```bash
$ docker-sync start
```

最後に新しいシェルを立ち上げてコンテナーを起動します。
```bash
$ docker-compose up -d
```

もしくは、`docker-sync start` と `docker-compose up` を同時に実行することもできます。

```bash
$ docker-sync-stack start
```

詳細は https://github.com/EugenMayer/docker-sync/wiki を参照してください。