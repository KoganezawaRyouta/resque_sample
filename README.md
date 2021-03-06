## Redisの環境構築 ##

* 01. Macへのインストール(Homebrew)
```
$ brew install redis
```

* 02. Redisの起動
```
$ redis-server /usr/local/etc/redis.conf
```

* 03. Redisの設定ファイルは下記
```
$ /usr/local/etc/redis.conf
```

### Tips ###

* 停止とか状況確認とか
```
$ redis-cli shutdown
$ redis-cli monitor
```

* コマンドラインから操作
```
$ redis-cli
$ redis 127.0.0.1:6379> GET myKey
$ (nil)
```

* 文字列の格納と参照
```
$ redis 127.0.0.1:6379> SET myKey myFoobar
$ OK
$ redis 127.0.0.1:6379> GET myKey
$ "myFoobar"
```

* キーの削除
```
$ redis 127.0.0.1:6379> DEL mykey
$ (integer) 1
$ redis 127.0.0.1:6379> GET mykey
$ (nil)
```

## Hello World Resque (Railsにresqueを導入する) ##
* Railsをインストール
```
$ sudo gem install rails
```

* アプリの生成
```
$ rails new resque_sample --skip-bundle
```

*  Gemに追加
```
$ gem 'resque'
```

* コマンドラインの設定を変更してbundle install
```
$ ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future bundle install --without production --path vendor/bundle/
```

* config/initializers/resque.rbを追加、中身は下記
```
$ Resque.redis = 'localhost:6379'
$ Resque.redis.namespace = "resque:resque_sample:#{Rails.env}" # アプリ毎に異なるnamespaceを定義しておく
```

* コントローラーを追加
```
$ rails g controller home
```

* routing設定
```
$ vi route.rb
$ +get "hello/:message" => 'home#hello'
```

* タスクの追加(lib/tasks/resque.rakeを作成する)
```
$ +require 'resque/tasks'
$ +task "resque:setup" => :environment
```

* Workerの作成
```
$ mkdir app/workers
```
* 中身は下記ファイルを参照
```
+hello_queue.rb
+koganezawa_queue.rb
```

* Resque workerを起動
```
$ QUEUE=<キューの名前> rake environment resque:work
$ QUEUE=resque_sample rake environment resque:work
$ QUEUE=* rake environment resque:work
```

### resqueの管理画面の表示 ###

* 下記を実行すると管理画面が表示される。
```
$ bundle exec resque-web
```

* 下記のようにGemfileとrouteのサーバーをマウントすれば、これで"http://localhost:3000/resque/overview"アクセスできる。
```
$ vi Gemfile
$ +gem 'resque', :require => 'resque/server'
$ vi route.rb
$ +mount Resque::Server, :at => "/resque"
```

## 各起動コマンド ##
```
$ redis-server /usr/local/etc/redis.conf
$ bundle exec rails s
$ QUEUE=resque_sample rake environment resque:work
$ http://localhost:3000/hello/world
$ http://localhost:3000/resque/overview
```

## コマンドラインよりredisの中身を確認してみた ##
* 確認手順と実行結果
```
$ redis-cli
$ 127.0.0.1:6379> keys 'resque:*'
$ 1) "resque:resque_sample:development:queue:resque_sample"
$ 2) "resque:resque_sample:development:stat:processed"
$ 3) "resque:resque_sample:development:queue:resque_sample_koganezawa"
$ 4) "resque:resque_sample:development:queues"
$ 127.0.0.1:6379>type 'resque:resque_sample:development:queue:resque_sample'
$ list
$ 127.0.0.1:6379> LRANGE 'resque:resque_sample:development:queue:resque_sample' 0 -1
$ 1) "{\"class\":\"HelloQueue\",\"args\":[\"world\"]}" 
$ 2) "{\"class\":\"KoganezawaQueue\",\"args\":[\"world\"]}"
```

* これが中身
```
$ 1) "{\"class\":\"HelloQueue\",\"args\":[\"world\"]}" 
$ 2) "{\"class\":\"KoganezawaQueue\",\"args\":[\"world\"]}"
クラス名とパラメータが設定されているのがわかる。
ワーカーはこの値を取得して、各クラスに定義されたキューの名称とワーカー実行時に
パラメーターとして渡されたキューの名称が一致するクラスのみ実行するっぽい。
```

### redisの中身を確認してみた結果、各モジュールが何をしてるか予想してみた ###

* HomeControllerについて
```
このコントローラーでリクエストを受け取ると、 redisへQUEUEのクラス名と
QUEUEへ引き渡すパラメータを保存する。（enqueueする）
```

* Workerの実行について
```
 $ QUEUE=resque_sample rake environment resque:work
キューを指定しワーカーを実行する。
```

* hello_queue.rbについて
```
これは実際のQUEUEの処理が書かれている。
ワーカー実行時はQUEUE名を指定する。QUEUEクラス名ではない。
ややこしいのは、このQUEUEクラス名とQUEUE名は別もの。
redisにはQUEUEクラス名が保存されている。

workerはredisに保存されたQUEUEクラス名を取得し、worker実行時に渡された引数（QUEUE）と
QUEUEクラス内に記載されたQUEUEが一致してるものを実行するっぽい。
ここらへんどういうふうに裏で動いてるのか実行結果とRedisの中身から予測してみた。
```

```
参考記事1：http://blog.hello-world.jp.net/?p=895 ※今回はこっちに記載されているアプリを模写した
参考記事2：http://blog.livedoor.jp/sasata299/archives/51889303.html
参考記事3：http://railscasts.com/episodes/271-resque?language=ja&view=asciicast ※これわかりやすい
参考記事4：http://kitak.hatenablog.jp/entry/2013/08/11/232218
Redisのドキュメント：http://redis.shibu.jp/index.html
Redisコマンドラインから操作基本：http://gihyo.jp/dev/feature/01/redis/0002

※iQONのバックエンドの非同期処理について(具体的な実装編)は下記参照
http://tech.vasily.jp/2013/02/iqon_backend_resque/
 
※Resque がぶっ壊れる話（参考）
https://gist.github.com/ssig33/9382101
```