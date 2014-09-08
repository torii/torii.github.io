## Mojoliciousを使ったwebアプリケーション開発 -実践編-
#### 2014/8/29
#### YAPC::Asia 2014
### Yoshimitsu Torii

---
# Note!
### Mojoliciousを使ったことない人もWAFの概念が分かれば大丈夫です！

---

## 自己紹介
- 鳥居良光 @torii704 1987/7/4生
- Livedoor 新卒入社
- LINE 3年目(9月から正社員4年目)

---

## PerlでWebアプリ開発入門

> PerlでWebアプリを作る環境は年々恵まれてきている。  
> 特に**入門者向け資料が豊富**


- Perl入学式
- 入門記事・資料
- Amon2やMojoliciousなどのWAFの登場

___

### アプリを作った！
> でも、運用の壁にぶち当たる…

- ユーザ(データ)が増えたら、レスポンスが遅くなった…
- フォームのデータにいつも変な値入れてくる人がいる
- 変更を反映するたびにアプリを止めるのはもうやめたい


___

> Webアプリを運用してみると  
> 工夫しないといけない点が山ほどある

___

## 具体的な対応例を紹介

> 様々な問題を事前に知っておくことが大事

- 例)「YAPC::Asiaサイトで実際に動いているシステム」
  - https://github.com/jpa-perl/yapcasia2014-app

___

### http://yapcasia.org/2014/
![yapc site](images/yapc-site.png)

___

### https://github.com/jpa-perl/yapcasia2014-app
![yapcapp github](images/github-yapc.png)

___

## 想定観客層
> ***Perl Beginner***

- 簡単なアプリを作ったことがある
- 仕事でPerlを使ってWebアプリを書く新卒エンジニア
- 「Mojoliciousでつくる！Webアプリ入門」の内容がわかる
- 実際の処理を意識したWebアプリを書きたい
- ::Lite を卒業して中規模なアプリの書き方を知りたい
- 本番の環境構築・運用についてざっくり知りたい

---

## 1. 開発環境の整理と理解

___

## Mojoliciousの基本構成

> $ mojo generate app MyApp

    my_app
    ├── lib
    │   ├── MyApp
    │   │   └── Example.pm
    │   └── MyApp.pm
    ├── public
    │   └── index.html
    ├── script
    │   └── my_app
    ├── t
    │   └── basic.t
    └── templates
        ├── example
        │   └── welcome.html.ep
        └── layouts
            └── default.html.ep

___

### mojo generateしてできたディレクトリ達


| Dir | 説明 |
| :-- | :-- |
| lib       | このディレクトリ下でwebアプリを作っていく|
| public    | 静的ファイル用の公開ディレクトリ|
| script    | batch系（コマンドラインで実行する）や起動スクリプトなどの置き場|
| t         | テストスクリプト用のディレクトリ|
| templates | view用のテンプレートファイル置き場|

___

### その他よく出てくるディレクトリ達

| Dir | 説明 |
| :---------------- | :---------------------------------------------------------- |
| etc,config        | webアプリの設定ファイルとか置く用に使われることが多い             |
| misc,sql,services | ↑ほどではないけど、その他必要なファイルなど(SQLやcrontab, service)|
| local,extlib      | carton, cpanmでインストールするモジュール置場                   |

___

### ./etc
    etc
    ├── app.psgi          # PSGI
    ├── config.pl         # common(production)用 configファイル
    ├── config_local.pl   # local用 configファイル
    ├── container.pl      # container用 設定ファイル
    └── profiles.pl       # validator用 設定ファイル

### ./misc
    misc
    ├── YAPC2014.sql
    ├── crontab
    └── services          # service登録用ディレクトリ(svc)
        └── yapcasia2014-app
            ├── log
            │   └── run
            └── run

---

## ローカル環境構築

>  ***perlbrew*** または ***plenv*** を使いましょう

    $ brew install perlbrew
     or
    $ brew install plenv

    see:
        perlbrew - http://perlbrew.pl/
        plenv    - https://github.com/tokuhirom/plenv

- 基本的にはローカル環境で開発  
- perlのversionを柔軟に変えられると捗る

___

## Cartonでモジュール管理
> モジュール管理は ***Carton*** を使いましょう  

    $ cpanm Carton

- モジュールのバージョン固定
- モジュールのインストール
- モジュールパスのロード

___

### carton install

    $ vi cpanfile
    $ carton install

#### cpanmで例えると

    $ perl Makefile.pl
    $ cpanm -L local --installdeps .

___

### localディレクトリに<br>モジュールが格納される

    local
    ├── bin
    ├── cache
    └── lib
        └── perl5
___

### Carton exec

    vi etc/app.psgi
    $ carton exec -I lib -- plackup etc/app.psgi

#### perl から始まるコマンドラインに書き換えると  

    $ perl -Mlib::core::only -Ilocal/lib/perl5 -Ilib -- plackup app.psgi

---

## 環境によって<br>読み込む config を変えよう

    etc/config.pl              # 共通configファイル
    etc/config_local.pl        # local環境用configファイル
    etc/config_dev.pl          # dev環境用configファイル
    etc/config_production.pl   # 本番環境用configファイル

___

### YAPCApp.pm でのconfigファイル読み込み
> $ENV{PLACK_ENV} で分岐

    # lib/YAPCApp.pm

    # This method will run once at server start
    sub startup {
        my $self = shift;
        my $config_file = 'etc/config.pl';
        if( $ENV{PLACK_ENV} =~ /^development/ ){
            $config_file = 'etc/config_dev.pl';
        } elsif ( $ENV{PLACK_ENV} =~ /^local/ ){
            $config_file = 'etc/config_local.pl';
        }

        $self->plugin(Config => { file => $config_file });
        ...
---

## 2. アプリケーション

___

### YAPCAppのlibディレクトリ構成

    lib
    ├── YAPCApp
    │   ├── API                     # Model(logic)層
                …
    │   │   ├── WithCache.pm        # Model(DB)層 の Cache機構
    │   │   ├── WithContainer.pm    # APIでContainerを使うためのモジュール
    │   │   └── WithDBI.pm          # Model(DB)層
    │   ├── Constants.pm            # 不変の値を定義
    │   ├── Container.pm            # 独自Containerの実装クラス
    │   ├── Controller              # Controller層
                …
    │   ├── Controller.pm
    │   ├── Renderer.pm             # 独自Rendererの実装クラス
    │   └── Utils.pm                # 便利メソッド集
    └── YAPCApp.pm

___

### MVC 対応表

| Dir/File | 説明 |
| :-- | :-- |
| API/* | Model層 (Logic)<br>"M","Model" という名前の場合も|
| Controller/* | Controller層<br>Routerから呼ばれる |
| Renderer.pm | View層<br> ./template内にあるテンプレを読み込む |

---

## 2.1 MVCのModelをつくる
> Tengがおすすめ、SQL::Maker + DBI系で自作もOK

- 独自O/R mapper (YAPCApp)
- Teng (Amon2推奨モジュール)
- DBIx::Skinny、DBIx::Class、DBIx…

___

### O/R MapperでDB操作

    # Tengの場合
    use Your::Model;

    my $teng = Your::Model->new(\%args);
    # insert new record.
    my $row = $teng->insert('user',
        {
            id   => 1,
        }
    );
    $row->update({name => 'nekokak'}); # same do { $row->name('nekokak'); $row->update; }

    $row = $teng->single_by_sql(q{SELECT id, name FROM user WHERE id = ?}, [ 1 ]);
    $row->delete();

___

#### 基本的には継承するだけでOK

    # lib/Your/Model.pm
    package Your::Model;
    use parent 'Teng';
    1;

#### Schema情報が必要

    # lib/Your/Model/Schema.pm
    package Your::Model::Schema;
    use Teng::Schema::Declare;
    table {
        name 'user';
        pk 'id';
        columns qw( foo bar baz );
    };
    1;

> refer: http://search.cpan.org/~satoh/Teng-0.25/lib/Teng.pm


---

## 2.2 Cacheを導入して<br>データアクセスを高速化
> 結果をメモリに保存して上手に再利用する

___

### KVSを扱うモジュール

- Redis.pm
- Cache::Redis
  - Cache::Cache like interface.

- Cache::Memcached::Fast
- Cache::Memcached::Fast::Safe  
  - memcached injection防止 & get_or_setが付いてる  

___

### Cacheの方法

> user毎に$cache_keyを変えてCacheする

    # DBの場合
    my $user_id = '1';

    my $cache = Cache::Memcached::Fast::Safe->new( $args );
    my $cache_key = CACHE_KEY . '::' . $user_id;
    if ( my $data = $cache->get( $cache_key ) ){
        return $data;
    } else {
        $row = $teng->single_by_sql(
            q{SELECT id, name FROM user WHERE id = ?},
            [ $user_id ]
        );
        $cache->set( $cache_key, $row, $expire );
        return $row;
    }
    ...


---

## 2.3 Containerの<br>仕組みと特性を理解しよう
> よく使う道具を一箇所に用意しておくための箱

- 独自Container (YAPCApp)
- Object::Container
- MojoliciousのHelperメソッド

___

### 例）Object::Containerで作ると

    # lib/MyApp/Container.pm
    package MyApp::Container;
    use Object::Container '-base';
    use JSON ();
    use Furl;
    use Text::Markdown ();
    use Cache::Memcached::Fast;

    register JSON => JSON->new->utf8;
    register Furl => Furl::HTTP->new;
    register Markdown => Text::Markdown->new();
    register Memcached => sub {
        my $config = $_[0]->get('config');
        Cache::Memcached::Fast->new($config->{Memcached});
    };

___

### MyApp::Container を use するだけでOK

    # script/hoge.pl
    use MyApp::Container 'api';

    api('Furl')->get('http://google.com');
    api('Memcached')->get($cache_key);

### YAPCAppの場合

    Object::Container            <=>  YAPCApp::Container
    MyApp::Container             <=>  etc/container.pl

    use MyApp::Container 'api';  <=>  $self->setup_container;
    api('Memcached')             <=>  $self->get('Memcached')

___

#### 例）container を Mojo::Helper に関連付け

    # lib/YAPCApp.pm
    has container => sub { YAPCApp::Container->new };

    sub setup_container {
        my $self = shift;

        my $file = $self->home->rel_file("etc/container.pl");
        my $container = $self->container;
        $container->register(config => $self->config);
        foreach my $f (applyenv($file)) {
            #...エラー処理中略...
            $self->log->info( " + Loading container $file" );
            $self->load_file(
                $f => ( register => sub (@) { $container->register(@_) } )
            );
        }
        $self->helper(get => sub { $container->get($_[1]) });
    }

___

### containerのメリット

> use、newが１度のみでOK

  - インスタンス生成回数を減らす → 生成コスト削減
  - 同じコードを繰り返し書かなくなる → コードの簡単化
  - 使いまわすインスタンスをContainerで一元管理 → コードの集約化

___

### containerを扱う上での注意点

- インスタンスの状態によって返り値が変わるものは不向き
- どこでも使われるインスタンスであることを意識

___

### Amon2の場合

> MyApp.pmにメソッドを生やすだけ

    # lib/Myapp.pm

    use Cache::Memcached::Fast;
    sub cache {
        my $c = shift;
        if (!exists $c->{memcache}){
            my $memd = Cache::Memcached::Fast->new( $c->config->{Memcached} );
            $c->{memcache} = $memd;
        }
        $c->{memcache}
    }

- $c から引けるので簡単

___

### Flyweight/Singletonパターンで<br>インスタンスを再利用できるように工夫したもの


---

## 2.4 RendererをText::Xslateに変更
> Xslateは便利で超高速な Renderer

- Text::Xslate
- MojoX::Renderer::Xslate

___

### テンプレートのsyntaxを TT に変更

> Text::Xslate::Bridge::TT2Like

- Template Toolkit
- [% と %] で挟んで記述
- 変数の置換え、for, if, include, フィルタ, マクロなどができる
- see: http://www.template-toolkit.org/

___

### テンプレートで注意すること

- テンプレートキャッシュが機能しているか
- 大量の処理があるテンプレのレンダリングは本当に高速か
- 独自関数の取り回し

___

### テンプレートキャッシュを意識
> View層で難しいことをしたら負け

- Xslateはキャッシュされる(前提)
  - 複雑な処理はModelで
  - 処理結果をstashに

___

例）200以上ある member icon
![indiv.png](images/indiv.png)

___

### テンプレート上の処理は遅い
> View層で難しいことをしたら負け

    # NG
    $self->stash( member_api => $self->get('API::Member'));
    [% FOREACH member IN members %]
        [% member_api.get_member_icon('') %]
        ....
    [% END %]

    # OK
    map{ $member->{icon} =  $member_api->get_icon($_->{twitter_id}) } @members;
    $self->stash( members => \@members );
    [% FOREACH member IN members %]
        [% member.icon %]
        ....
    [% END %]


- **テンプレ内の処理はシンプルに**

---

## 2.5 Session と CSRF
> フォームを扱ったり、ログインさせるときのお話

___

### Mojoliciousのsessionは使いずらい？
> Key/Value+"$secret"をBase64エンコード → cookieに保存
- Mojolicious::Plugin::DefaultHelpers の csrf_token (Mojolicious 4.59~)

___

### Plack::MiddleWare::Session(::Simple)を使う
> session IDだけをcookieに保存 → 実データはserver側

    # etc/app.psgi
    use Plack::Middleware::Session::Simple;
    use Cache::Memcached::Fast;
    builder {
        enable 'Session::Simple',
            store => Cache::Memcached::Fast->new({servers=>[..]}),
            cookie_name => 'myapp_session';
        ;
        ...
    }

    # lib/YAPCApp.pm
    $self->helper( sessions => sub {
        Plack::Session->new($_[0]->req->env);
    });

___

### CSRF

- 独自にsubsession_idを発行・管理 (YAPCApp)
- Mojolicious::Plugin::CSRFDefender
- HTTP::Session2
  - (Amon2::Plugin::Web::CSRFDefender)

___

#### Pluginはcsrf_token埋込み型

    <input type="hidden" value="[% csrf_token $]" />が自動で入る

    <input type="hidden" name="csrf_token" value="zxjkzX9RnCYwlloVtOVGCfbwjrwWZgWr" />

___

![form](images/form.png)

___

![form](images/form2.png)

---

## 3. 本番環境
> 構築/構成/デプロイ/運用

___

### 本番のperl環境を構築する (xbuild)
> 指定したバージョンのperlをインストール

    $ git clone https://github.com/tagomoris/xbuild
    $ ./xbuild/perl-install 5.20.1 ~/local/perl-5.20
    $ export PATH=$HOME/local/perl-5.20/bin:$PATH

    $ cd /path/to/myapp
    $ carton install

    $ carton exec -- perl -Ilib my_script.pl   # batch処理
    $ start_server --port 5000 -- carton exec -- plackup app.psgi   # アプリ起動

- Pathを通すだけでいいので怖くないし楽ちん！便利！
- Perlの場合はCartonやServer::Starterなども一緒に入る
- **Ruby,Node.js,PHP,Pythonにも対応！**

---

### システム構成
> nginx(httpd) + perl(psgi) + mysql

- ApacheでもOK
  - ただし reverse proxy として使う

___

### httpdを立ててアプリにproxyする
> httpd:80 -> app:5014 のポートフォワーディング  
> staticファイルは nginx で配信

    # /etc/nginx/conf.d/yapcasia.org.conf
    upstream yapcasia2014-app {
        server 127.0.0.1:5014;
    }
    server {
        listen 80;
        server_name yapcasia.org;
        access_log /var/log/nginx/yapcasia.org.access.log main;
        root /var/lib/jpa/yapcasia.org/docs;   # staticファイル置き場
        location ~ ^/2014/(member|talk|auth|...|event){
            proxy_set_header X-Real-IP  $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://yapcasia2014-app;
        }
    }

___

### daemontoolsでプロセス生存管理
> /service以下のディレクトリを管理

    /usr/local/bin/svc で操作
    /service/myapp -> $APP_HOME/misc/services/myapp

    /usr/local/bin/svc -t or -h /servise/yapcasia2014-app   # restart
    /usr/local/bin/svstat /servise/yapcasia2014-app   # check

- serviceディレクトリ以下からシンボリックリンクでつなぐ

#### ./misc
    misc
    └── services                 # service登録用ディレクトリ(svc)
        └── yapcasia2014-app
            ├── log
            │   ├── run          # log用 runファイル
            |   └── main/current # logfile
            └── run              # app起動用 runファイル

___

### app起動用 runファイル

    # misc/services/yapcasia2014-app/run
    #!/bin/sh
    APP_HOME=/var/lib/jpa/yapcasia.org/yapcasia2014-app

    cd $APP_HOME
    exec $APP_HOME/script/carton.sh plackup etc/app.psgi -p 5014 -o 127.0.0.1 2>&1

___

## script/carton.sh

    #!/bin/sh
    export MOJO_MODE=production
    export PLACK_ENV=production
    export LC_ALL=en_US
    export PATH=/opt/local/perl-5.16/bin:$PATH
    APP_HOME=/var/lib/jpa/yapcasia.org/yapcasia2014-app

    cd $APP_HOME
    exec carton exec -Ilib -- $@

___

### Server::Starter で hot deploy

- 再起動の時にリクエストの処理を続けながら、変更の内容を反映

    $ start_server --port 5000 -- carton exec -- plackup app.psgi   # アプリ起動

>  見かけ上落とさずに再起動できる

- 失敗してもstart_serverが前の世代を握っているのでサービスは止まらない
- see: http://lestrrat.ldblog.jp/archives/38601017.html

---

## 以上、<br>入門から一歩先の案内をしてみました
> そして来年、YAPCシステムを  
> 担当してくれる人を募集します！

---

### x. 今回扱わなかったお話
- 複数人開発 git
- 過負荷に耐えるためのサーバ構成
- CLI、batchのお話
- フロントエンドの話
- サービス運用のためのサービスのお話 (HRForecastとか)
- などなど

---

## 投票受付中！
### http://yapcasia.org/2014vote/ballot
![vote](images/vote.png)

---

## ご清聴ありがとうございました
