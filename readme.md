# Docker+Laravel+OpenAPIGenerator

## はじめに

[OpenAPIGenerator](https://github.com/OpenAPITools/openapi-generator)を使って生成されたLaravelのソースコードで環境構築してみました。
DockerでLaravel環境の構築～スタブサーバの生成、リクエストの確認まで行います。

- windowsのGitBashを使う場合は`exec winpty bash`をやっておくと便利

**プロジェクトディレクトリ構成**

```bash
project/
 ├ www/                 # Laravel Project Container
 ├ generator/           # generator Container
 ├ docker-compose.yml
 ├ oas.yml
```

## Sample OAS

- プロジェクト直下に`oas.yml`を用意します。
- 簡易的なCRUDが行えるAPIを想定しています。

```yaml:oas.yml
openapi: 3.0.0
info:
  title: Task API
  version: 0.0.1
servers:
  - url: http://localhost
    description: Laravel Server
tags:
  - name: Task
paths:
  /api/task:
    get:
      tags: [ "Task" ]
      summary: Show Task List.
      description: Show Task List.
      operationId: listTask
      responses:
        '200':
          description: Successful response
    post:
      tags: [ "Task" ]
      summary: Create Task One 
      operationId: createTask
      requestBody:
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              properties:
                title:
                  type: string
                sort:
                  type: integer
      responses:
        '200':
          description: Successful response
  /api/task/{tid}:
    get:
      tags: [ "Task" ]
      summary: Show Task One.
      description: Show Task One.
      operationId: showTask
      parameters:
        - name: tid
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Successful response
    put:
      tags: [ "Task" ]
      summary: Update Task One.
      operationId: updateTask
      description: Update Task One.
      parameters:
        - name: tid
          in: path
          required: true
          schema:
            type: string
      requestBody:
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              properties:
                title:
                  type: string
                sort:
                  type: integer
      responses:
        '200':
          description: Successful response
    delete:
      tags: [ "Task" ]
      summary: Delete Task One.
      operationId: deleteTask
      description: Delete Task One.
      parameters:
        - name: tid
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Successful response
```

## Docker環境を用意する

- `docker-compose.yml`をproject直下に用意します。

```yaml:docker-compose.yml
version: '3.4'
services:
  generator:
    build: ./generator
    volumes:
      - .:/app
    tty: true
    command: sh

  www:
    build: ./www
    volumes:
    - ./www:/var/www
    ports:
      - 80:80
```
- `generator`コンテナはOASからPHPコードを生成するコンテナになります。
- `www`コンテナはApacheで動作するLaravel環境のコンテナになります。

- 続いて、`generator`コンテナを構築するためのDockerfileを`generator/`配下に用意します。

```Dockerfile
FROM openjdk:8-jdk-alpine

WORKDIR /app

RUN apk --update add bash maven git
RUN rm -rf /var/cache/apk/*ls
RUN git clone https://github.com/openapitools/openapi-generator /generator
RUN cd /generator && mvn clean package
```
- OpenAPIGeneratorはJVMでありJava環境が必要なため`openjdk:8-jdk-alpine`をベースにしています。
- ライブラリのインストーラとして`maven`を使用します。

- 続いて、`www`コンテナを構築するためのDockerfileを`www/`配下に用意します。

```www/Dockerfile
FROM php:7.3-apache

RUN apt-get update && apt-get install -y git libzip-dev libxml2-dev
RUN docker-php-ext-configure zip --with-libzip
RUN docker-php-ext-install pdo_mysql mbstring zip xml
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
RUN a2enmod rewrite && a2enmod headers
RUN sed -ri -e 's!/var/www/html!/var/www/public!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!/var/www/public!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data
​
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_HOME /composer
ENV PATH $PATH:/composer/vendor/bin

WORKDIR /var/www
​
RUN composer global require "laravel/installer"
```
- 今回はApache上で動作するPHP（`php:7.3-apache`）をベースにしています。
- Laravelを使いたいので`composer`をインストールしています。
- URLを書き換えやリダイレクトを有効にするため、`a2enmod rewrite && a2enmod headers`を実行しています。
- `sed`でLaravel用のドキュメントルートに変更しています。
- Apacheを動かす`www-data`ユーザがLaravelから吐き出されるlogファイルにアクセスできるように権限を付与しています。

- 上記の準備ができたらコンテナを起動します。
- 各リソースのDLやInstall処理が一気に走るため結構な時間を要するかと思います。

```bash
$ docker-compose up -d --build
```

## Laravel用のソースコードを自動生成する

- OpenAPIGeneratorを使用してソースコードを生成します。
- `generator`コンテナに入って作業します。

```bash
$ docker-compose exec generator bash
bash-4.4# java -jar /generator/modules/openapi-generator-cli/target/openapi-generator-cli.jar generate \
   -i oas.yml \
   -g php-laravel \
   -o server-generate
bash-4.4# cp -rf server-generate/lib/. www
```
- 上記を実行すると、`server-generate`に自動生成されたPHPのソースコードが出来上がると思います。
- 生成されたソースコード一式は`www`コンテナへコピーします。
- ちなみに、自動生成できるコードの種類は[こちら](https://openapi-generator.tech/docs/generators.html)から確認できます。

## 生成されたソースコードからLaravel環境を立ち上げる

```bash
$ docker-compose exec www bash
root@5295f267d6a1:/var/www# composer install
root@5295f267d6a1:/var/www# cp .env.example .env
root@5295f267d6a1:/var/www# php artisan key:generate
```

- 続いて、生成されたコードが機能しているのか確認してみます。
- `php artisan route:list`を実行してAPIのエンドポイントを確認してみます。

```bash
$ docker-compose exec www bash
root@5295f267d6a1:/var/www# php artisan route:list
+--------+----------+----------------+------+------------------------------------------------+------------+
| Domain | Method   | URI            | Name | Action                                         | Middleware |
+--------+----------+----------------+------+------------------------------------------------+------------+
|        | GET|HEAD | /              |      | App\Http\Controllers\Controller@welcome        | web        |
|        | POST     | api/task       |      | App\Http\Controllers\TaskController@createTask | api        |
|        | GET|HEAD | api/task       |      | App\Http\Controllers\TaskController@listTask   | api        |
|        | DELETE   | api/task/{tid} |      | App\Http\Controllers\TaskController@deleteTask | api        |
|        | GET|HEAD | api/task/{tid} |      | App\Http\Controllers\TaskController@showTask   | api        |
|        | PUT      | api/task/{tid} |      | App\Http\Controllers\TaskController@updateTask | api        |
+--------+----------+----------------+------+------------------------------------------------+------------+
```
- エンドポイントの確認が行えたので、続いて[swagger-editor](https://editor.swagger.io/)からGETリクエストを試したいと思います。
- ただ、現時点のままだとCORSに引っかかりリクエストできません。
- そのため、リクエストのテストを行う前にCORS対策を行います。
- まずは`barryvdh/laravel-cors`をインストールします。

```bash
$ docker-compose exec www bash
root@5295f267d6a1:/var/www# composer require barryvdh/laravel-cors
```
- `www\config\app.php`に下記を追加します。

```php:
'providers' => [
    // ...
    Barryvdh\Cors\ServiceProvider::class,
]
```
- `app/Http/Kernel.php`に下記を追加します。

```php:
'api' => [
    // ...
    \Barryvdh\Cors\HandleCors::class,
],
```
- 設定ファイルを以下のコマンドで作成します。

```bash
root@02eec0e47db9:/var/www# php artisan vendor:publish --provider="Barryvdh\Cors\ServiceProvider"
```
- 設定が完了したので、[swagger-editor](https://editor.swagger.io/)でリクエストテストを行ってみます。