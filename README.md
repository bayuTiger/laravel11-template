# 環境構築

```
docker compose up -d --build
```

## 記事⇩

https://qiita.com/hitotch/items/2e816bc1423d00562dc2

## 内容⇩

Laravel 11でのWebアプリ開発環境を、docker上に実現する方法を紹介します。

ゴール
WindowsやMac上に、dockerを使ったLaravel 11開発環境を作る。
サーバーはnginx, php, mysqlを別にする。
Viteの利点であるページリロード無しでのCSS/JS更新を実現する。
Laravel MixではなくViteを使う
前提
PHP 8の知識がある
Laravel 11の知識がある
Laravel 11に必要な環境構築ができる
Dockerで簡単な環境構築ができる
本投稿ではWindowsを使っているので、MacやLinuxで合わないところは読み替えてください。

Dockerに各サーバーを準備する
目指すフォルダ構成はこんな感じにします。(Docker関連のみ)

l11dev (プロジェクトフォルダ)
├ docker
│ ├ mysql
│ │ └ (中身たくさん)
│ ├ nginx
│ │ ├ logs
│ │ ├ default.conf
│ │ └ Dockerfile
│ └ php
│ 　├ Dockerfile
│ 　└ php.ini
└ docker-compose.yml
Dockerをインストールする
ホストOSにDockerをインストールした状態から開始。(個人、教育機関、中小企業は無償でDocker Desktopを使えます。大企業の場合Docker Desktopは有償なので注意。)

プロジェクトフォルダにDocker用ファイルを作る
今回はドキュメントフォルダの下にQiitaフォルダを作り、l11devフォルダを作ります。
作り終わったら、そのフォルダの中にDocker用ファイルを作っていきましょう。

docker-compose.yml
まずはdocker-compose.ymlから。

docker-compose.yml
version: '3'
services:
  l11dev-nginx:
    container_name: "l11dev-nginx"
    build:
      context: ./docker/nginx
    depends_on:
      - l11dev-app
    ports:
      - 80:80
    volumes:
      - ./:/src

  l11dev-app:
    container_name: "l11dev-app"
    build:
      context: ./docker/php
    depends_on:
      - l11dev-mysql
    volumes:
      - ./:/src
      - /src/node_modules
      - /src/vendor
      - ./docker/php/php.ini:/usr/local/etc/php/php.ini

  l11dev-mysql:
    image: mysql:8.0.37
    command: --max_allowed_packet=32505856
    container_name: "l11dev-mysql"
    volumes:
      - ./docker/mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=l11dev
    ports:
      - 3306:3306

  l11dev-redis:
    image: redis:alpine
    container_name: "l11dev-redis"
    ports:
      - 16379:6379
いくつかポイントを抑えます。

Webサーバーはnginxなので、appサーバー（PHPサーバー）を別に立ち上げてます。
→今回、後で出てくる説明で対象サーバーをわかりやすくするため意図的にWebサーバーとappサーバーを分けてます。Apache+PHPで同一サーバーにしても全く問題ありません。

Webサーバーはnginxで、ポート80をDocker外に見せて起動します。

AppサーバーはPHPが動作するようにし、ポートはDocker外に見せず、nginxからdocker内のネットワークで呼び出されて動作します。PHPの設定を指定しやすいように、php.iniだけdocker/phpフォルダで見えるようにしています。
また、/src/node_modulesフォルダと/src/vendorフォルダはボリュームマウントから外してます。これは依存ライブラリ群をマウント対象から外すことで動作の高速化を狙ってします。Viteを使った開発環境ではその特性上★超重要★で、これをしないとページのロードに平気で1分とかかかります。

DBサーバーはMySQLで、デフォルトポートの3306をDocker外に見せて起動します。
ポート3306をDocker外に見せなくても動作はしますが、開発中にHeidiSQLやSequel Aceを使ってDBの中身を見るためには必要です。MySQLユーザー情報は、ユーザーIDがroot、パスワードがenvironmentで指定したMYSQL_ROOT_PASSWORDになります。
MySQL（とredis）はここでイメージを指定し、個別のDockerfileを持たない設定とします。
max_allowed_packetオプションは指定しなくてもいいです。(が、指定したくなることが多いのでここに書いてます)

nginxのDockerfile等
まずは手動でプロジェクトフォルダ/docker/nginxフォルダを作ります。

nginxのDockerfileを作ります。
同時に、最低限必要な設定であるdefault.confを記述します。

Dockerfile
FROM nginx:1.27
COPY ./default.conf /etc/nginx/conf.d/default.conf
さらに、nginxの動作を設定するdefault.confを作ります。

default.conf
server {
    
    listen 80;
    server_name _;

    client_max_body_size 1G;

    root /src/public;
    index index.php;

    access_log /src/docker/nginx/logs/access.log;
    error_log  /src/docker/nginx/logs/error.log;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;    
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass l11dev-app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
        
}
開発を進める上でnginxの設定を変更したい場合はここに記載することになります。

phpのDockerfile等
まずは手動でプロジェクトフォルダ/docker/phpフォルダを作ります。

phpのDockerfileを作ります。
同時に、最低限必要な設定であるphp.iniを記述します。

Dockerfile
FROM php:8.3-fpm
RUN apt-get update \
    && apt-get install -y \
    git \
    zip \
    unzip \
    vim \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpng-dev \
    libfontconfig1 \
    libxrender1

RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install gd
RUN docker-php-ext-install bcmath
RUN docker-php-ext-install pdo_mysql mysqli exif
RUN cd /usr/bin && curl -s http://getcomposer.org/installer | php && ln -s /usr/bin/composer.phar /usr/bin/composer

ENV NODE_VERSION=20.15.0
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
ENV NVM_DIR=/root/.nvm
RUN . "$NVM_DIR/nvm.sh" && nvm install ${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm use v${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm alias default v${NODE_VERSION}
ENV PATH="/root/.nvm/versions/node/v${NODE_VERSION}/bin/:${PATH}"
RUN node --version
RUN npm --version

WORKDIR /src
ADD . /src/storage
RUN chown -R www-data:www-data /src/storage
大事なポイントは、ComposerとNodeを入れている部分です。ComposerやNodeが古いと動作しませんので、アクティブなバージョンを指定します。ここではComposerは最新、Nodeは20.15.0以上を指定します。

開発中にNodeのバージョンコントロールもしたい場合があるので、NVMも入れています。
gdなどいくつかライブラリを入れていますが、このあたりは趣味（私の場合、自分がよく使うレンタルサーバーに寄せてる）なので好きなライブラリを入れましょう。

php.ini
upload_max_filesize=256M
post_max_size=256M
上記では例としてupload_max_filesizeとpost_max_sizeを指定していますが、特に設定がなければ空のファイルでもOKです。

実行テストする
ターミナル（WindowsならPowerShell）を起動し、プロジェクトフォルダに移動します。

たとえばWindowsのPowerShellを起動して、ドキュメントフォルダのQiitaフォルダに作ったl10devプロジェクトフォルダに移動した場合は：

image.png

ターミナルで、nginxとMySQLの起動に必要なフォルダを準備します。
※gitを使う場合、これらはignoreすべきで、gitからcloneした直後に実行すべきターミナルコマンドです。

# ログファイルを格納するディレクトリを作る（Gitを使う場合はこの``logs``フォルダをignoreすべき）
> mkdir ./docker/nginx/logs

# MySQLで使用するディレクトリを作る（Gitを使う場合はこの``mysql``フォルダをignoreすべき）
> mkdir ./docker/mysql
これでDocker起動に必要なすべてのファイルが揃いました。起動しましょう。

> docker compose up -d
うまく行けば以下のように４つのコンテナが正常起動します。

image.png

Webサーバーとして機能しているか確認するため、Chromeを開いてhttp://localhostにアクセスしてみましょう。

image.png

まだドキュメントルートに何もアップロードしてないので「File not found.」で正解です。サーバーは正しく起動できました。

Laravelをインストールする
Laravelのインストール先は、appサーバー＝phpサーバーです。
appサーバーに入って、composerを使ってLaravelをインストールします。

appサーバーのコンソールに入る
> docker compose exec l11dev-app bash
以下のように入れたらOKです。

image.png

ここからは「>」がホストOS（ここではWindows）、「$」がDockerコンテナ内のOSとして表記します。
上記/src#の#部分が$だと思って読み替えてください。

サブフォルダにLaravelをインストールする
Laravelをプロジェクトフォルダ（l11devフォルダ）にインストールします。
ただ、プロジェクトフォルダにはすでにdocker関連ファイルがあるので、以下のようにサブフォルダ「l11dev_tmp」に一旦インストールし、その後中身をプロジェクトフォルダに移動することにします。

まずサブフォルダを作ってそこにLaravelをインストール。ここではバージョン11を指定しています。

$ cd /src
$ mkdir l11dev_tmp
$ cd l11dev_tmp
$ composer create-project "laravel/laravel=11.*" . --prefer-dist
次に中身をプロジェクトフォルダに移動し、一時サブフォルダを削除。

$ cd /src
$ mv l11dev_tmp/* ./
$ mv l11dev_tmp/.* ./
$ rm l11dev_tmp -rf
このとき以下のように、/.、/..、vendor、node_modulesのフォルダが移動できないというメッセージ以外に問題がなければ成功です。

依存のインストール
サブフォルダからプロジェクトフォルダに移動したので、このフォルダで依存をインストールします。

$ cd /src
$ composer install
$ npm install
もしnpm installのときにnpm ERR! Tracker "idealTree" already existsのようなエラーが出たら、npm cache clear --forceしてからnpm installしなおしてみましょう。

最終的に、フォルダ構成が以下のようになるはずです。

image.png

ここまでrootユーザーでLaravelをインストールしましたが、storageフォルダ以下のフォルダはWeb用のユーザーwww-dataが書き込みできなくてはいけないので、書き込み権限を付与しておきます。
これをやっておかないと、実行テストをした時点で「The stream or file "/src/storage/logs/laravel.log" could not be opened in append mode」というエラーになります。

$ chmod -R guo+w storage
Laravel公式のstorage利用手順にも指定のあるように、storageのシンボリックリンクを設置します。

$ php artisan storage:link
LaravelとDBを接続する
Laravel11から、セッションの保存先がデフォルトでDBになりました。これにより、DB設定をしてセッション保存先テーブルを準備しておかないとLaravelのフロントが正常に起動できません。DB設定は/src/.envファイルにDB_で始まる変数が記述部分にあるので、デフォルトの記述をコメントアウトするなどして以下のように記述します。

.env
# DB_CONNECTION=sqlite
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=laravel
# DB_USERNAME=root
# DB_PASSWORD=

DB_CONNECTION=mysql
DB_HOST=l11dev-mysql
DB_PORT=3306
DB_DATABASE=l11dev
DB_USERNAME=root
DB_PASSWORD=root
さらに、セッションの保存先テーブルを含むデフォルトのテーブルを作ります。

$ php artisan migrate
実行テストする
WebサーバーにアクセスするとLaravelが動作するはずなので、Chromeを開いてhttp://localhostにアクセスしてみましょう。

image.png

Viteを使う
この記事ではViteを使ってCSSを変更し、その変更がフロントに反映されるまでを実装してみます。

依存のインストール
上ですでに実行していますが、再掲しておきます。再実行しても問題ありません。

$ npm install
ViewにViteを埋め込む
\resources\views\welcome.blade.phpをエディタで開き、以下のように<head>の最後にViteを埋め込みます。

welcome.blade.php
<!DOCTYPE html>
<html ...>
    <head>
        {{-- ... --}}
 
        @vite(['resources/css/app.css', 'resources/js/app.js'])
    </head>
埋め込もうとしているapp.cssやapp.jsは、Laravelインストール時に自動生成されているファイルです。

まずはこの状態でhttp://localhostを見てみましょう。

image.png

Viteを起動していないので、当然エラーになるわけです。

Viteを起動する
次に、Viteを起動します。

$ npm run dev
image.png

起動成功したので、http://localhostを見てみましょう。

image.png

ブラウザのコンソールログにいくつかエラーが出ます。これは、Viteから提供されるはずのファイルが読めない状態です。

image.png

通信先は127.0.0.1:5173です。これが失敗しているのは、以下２つが原因です。

そもそもポート5173がDocker外に開かれていない
ViteはLocal:は見ているがNetwork:は見ていない
先程Viteを起動したときにLocal: http://localhost:5173/と表示されているのがローカル（appサーバー）に反応するという意味です。
Network: use --host to exposeとなっているのは、ネットワーク経由でもアクセスするなら--hostオプションをつけろという意味です。

appサーバーのポート5173を見せる
Docker外にいるブラウザから、appサーバー内にいるViteが見えるよう、ポート5173を見せるため、docker-compose.ymlにポートを追記します。

docker-compose.yml
...
  l11dev-app:
    container_name: "l11dev-app"
    build:
      context: ./docker/php
    depends_on:
      - l11dev-mysql
    ports:
      - 5173:5173
    volumes:
      - ./:/src
      - /src/node_modules
      - /src/vendor
      - ./docker/php/php.ini:/usr/local/etc/php/php.ini
...
ports:が追記されました。これでDocker外からlocalhost:5173にアクセスすると、appサーバーのポート5173につながります。

また、/docker/php/Dockerfileの先頭にもEXPOSEの設定が必要です。

Dockerfile
FROM php:8.3-fpm
EXPOSE 5173
RUN apt-get update \
...
Dockerfileへの変更を反映するには、Dockerコンテナを再起動する必要があります。
Viteがまだ実行中であれば、Ctrl + Cで停止してからターミナルを抜けて、Dockerコンテナを再起動することになります。
この時、composerとnpmのinstallコマンドを再実行することに注意してください。理由はこの記事の下部に書きます。

$ exit
> docker compose down
> docker compose up -d --build
> docker compose exec l11dev-app bash
$ composer install
$ npm install
--host オプションを追加する
--host オプションは、Vite起動オプションなのでvite.config.jsまたはnpm run devの中身を変更することになります。どちらの方法でも大丈夫ですが、あとでvite.config.jsは追記するのでvite.config.js側に設定を集約するほうが後々楽でしょう。

vite.config.jsで設定する場合は、以下のようにserver hostを設定します。

vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        ...
    ],
    server: {
        host: true,
    },
});
package.jsonで設定する場合は、以下のように--hostオプションを追記します。

package.json
{
    "private": true,
    "scripts": {
        "dev": "vite --host",
        "build": "vite build"
    },
    ...
}
Viteを起動（すでに起動中の場合はCtrl+Cで停止してから再度起動）します。

$ npm run dev
今度はNetwork:も見ていることがわかります。

image.png

もう一度、http://localhostを見てみましょう。

image.png

image.png

またエラーが出ています。
今度はhttp://[::]:5173/以下のファイルが見えないというエラーです。
これはViteの真骨頂であるホットリロード機構HMRと通信できていないことを意味します。解決には、HMRの設定でホストをlocalhostにすればOKです。

vite.config.js
...
    server: {
        host: true,
        hmr: {
            host: 'localhost',
        },
    },
});
ファイルを保存したら、Viteを再起動して、もう一度、http://localhostを見てみましょう。

image.png

コンソールのエラーが消えました。

CSSを変更し、フロントに反映する
ではCSSを変更してみましょう。
/resources/css/app.cssをエディタで開くと空ですので、以下のように入力し保存します。

app.css
.text-xl{
    color: orange;
}
ブラウザでhttp://localhostを見てみましょう。（Windowsでは）反映されていません。
※MacやLinuxの場合は問題が出ないと思われますので、読み飛ばしてください。

image.png

Viteを再起動して、もう一度、http://localhostを見てみましょう。

image.png

反映されました！
しかし、これではViteの最重要機能「CSS保存で瞬時に反映」が実現できていません。
問題は、appサーバー内でViteがファイル編集イベントに反応できていないことで起こっています。この問題はどうもWSL2に関連する問題のようで、Vite公式でも言及されています。

こういった場合、Viteをポーリングモードで動作させることができます。Viteの設定ファイルにポーリングモードで動作するようusePollingを記述します。

vite.config.js
...
    server: {
        host: true,
        hmr: {
            host: 'localhost',
        },
        watch: {
          usePolling: true,
        },
    },
});
Viteを再起動し、ブラウザでhttp://localhostを開き、リンク文字がオレンジであることを確認したら、以下のようにリンク文字のCSS設定を黄色にして保存してみましょう。

app.css
.text-xl{
    color: blue;
}
保存後すぐにViteが以下のように反応するはずです。

image.png

ブラウザ側でもすぐリンク文字が青くなります。

image.png

Dockerコンテナの操作におけるノウハウ
Dockerコンテナの起動と終了
起動

> docker compose up -d
リビルドと起動

> docker compose up -d --build
停止

> docker compose down
Dockerコンテナ起動時に毎回実行すべきコマンド
今回は、動作を高速にするために/src/node_modulesフォルダと/src/vendorフォルダはボリュームマウントから外してます。
これにより、composerやnpmでインストールされた依存ライブラリはDockerコンテナを終了するごとに消えてしまいます。したがって、docker compose upするときは毎回直後に依存ライブラリをインストールする必要があります。

> docker compose up -d
> docker compose exec l11dev-app bash
$ composer install
$ npm install
面倒なので自動実行できるようにしましょう。appサーバーのDockerfile（docker/php/Dockerfile）の最後に以下の行を入れれば解決です。修正後のDocker起動で、--buildオプションを忘れずにつけましょう。

Dockerfile
ENTRYPOINT [ "bash", "-c", "composer install; npm install; exec php-fpm" ]
以上です！お疲れ様でした！
