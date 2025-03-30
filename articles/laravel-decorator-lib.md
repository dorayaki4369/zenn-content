---
title: "Laravelで使えるデコレータライブラリを作った"
emoji: "🖼️"
type: "tech"
topics: ["PHP", "Laravel", "decorator"]
published: false
---

# はじめに
[Laravel](https://laravel.com)で使用できるデコレーターライブラリを作り、[packagist](https://packagist.org/packages/dorayaki4369/laravel-decorator)に公開しました。

このライブラリを作ったきっかけは、**サービスクラスなどで共通の処理（ログ出力やトランザクション管理など）をスマートに書きたかった**からです。Laravelでは、こうした処理をミドルウェアやイベントで書くこともできますが、メソッド単位で柔軟に適用したい場面も多く、うまくはまらないことがよくありました。
そこで注目したのがデコレーターパターンと、PHP8から導入された[アトリビュート](https://www.php.net/manual/ja/language.attributes.overview.php)の組み合わせです。

本記事では、このライブラリの概要・使用方法・仕組みについて解説します。以下のような方に特におすすめです。

+ Laravelで**アスペクト指向的な記述**をしたい方
+ サービスクラスに**共通処理を分離**して書きたい方
+ アトリビュートを使ったモダンなコード設計に興味がある方

# ライブラリ作成の背景・課題
Laravelでサービスクラスを活用して開発していると、次第に以下のような共通処理をあちこちで書く必要が出てくることがあります。

+ メソッドの実行前後でログに記録
+ DBトランザクションの開始・コミット・ロールバック処理
+ キャッシュの保存

こうした処理を、「毎回手書きで書く」、「トレイトでまとめる」、「基底クラスを作成して継承させる」といった方法でまとめるのも1つの手ですが、「共通処理をまとめたい」というニーズと、「柔軟にクラス設計したい」というニーズがぶつかり、中々良い方法が思いつきませんでした。
そんな中、このジレンマを解消する手段として思いついたのが、**デコレーター＋アトリビュートによるアスペクト的な仕組み**でした。

# ライブラリの特徴
このライブラリの最大の特徴は、**クラスのメソッドにアトリビュートを付与するだけで、共通処理を簡潔に追加できる点**です。
しかも、**既存の処理を変更せずに適用できる**のがポイントで、対象のメソッドに設定するだけでOKです。

実際のコード例を見てみましょう。
まず、以下のように`Decorator`インターフェイスを実装したアトリビュートを作成します。本例では、メソッドの実行前後にログを記録する処理を実装した`LogDecorator`を作成したとします。

```php
<?php

namespace App\Attributes;

use Attribute;
use Dorayaki4369\LaravelDecorator\Contracts\Attributes\Decorator;

#[Attribute(Attribute::TARGET_METHOD)]
class LogDecorator implements Decorator
{
    public function decorate(callable $next, array $args, object $instance, string $parentClass, string $method): mixed
    {
        // 対象メソッド前に実行
        \Illuminate\Support\Facades\Log::debug('Before the method is called');
        
        $result = $next($args, $instance, $parentClass, $method);
        
        // 対象メソッド実行後に実行
        \Illuminate\Support\Facades\Log::debug('After the method is called');
        
        return $result;
    }
}
```

そして、作成したデコレーターを使用したいクラスのメソッドに設定し、

```php
<?php

namespace App\Services;

use App\Attributes\LogDecorator;

class MyService
{
    #[LogDecorator]
    public function exec(string $str, array $options = []): string
    {
        // 何らかの処理
    }
}
```

Laravelのサービスコンテナにてインスタンス化されたクラスからメソッドを呼び出すことで、メソッドの前後でログを記録する処理が実行されます。

```php
<?php

namespace App\Controllers;

use App\Services\MyService;

class MyController
{
    public function index(MyService $service): string
    {
        // メソッドの実行前後でLogDecoratorのログ記録処理が実行される
        return $service->exec('hello', ['option' => 'value']);
    }
}
```

上記のように、**アトリビュートを付けてメソッドを呼び出すだけで、デコレーターの処理が自動的に実行されます。**
設定不要・コード変更最小で共通処理を注入できるのが、このライブラリの大きな魅力です！

:::message
**制約**
このライブラリは、以下の特徴を満たしたクラス・メソッドでのみ機能します。
特徴を満たさないクラスやメソッドに適用した場合は無視されるようになっています。

+ サービスコンテナでインスタンス化できること
+ クラスが継承可能なこと
+ メソッドがオーバーライド可能なこと
+ メソッドのアクセス修飾子が`public`になっていること
+ クラスが`readonly`になってないこと(PHP8.3以降は`readonly`修飾子ありでもOKです)

なぜこれらの制約があるのかについては、[仕組み](#仕組み)を見ていただけると分かります。
:::

## デフォルトデコレーター
さらに、本ライブラリではよく使わそうな3つのデコレーターを予め用意しています。

### `DBTransactionDecorator`クラス
対象メソッドをDatabase Transactionでラップするデコレーターです。

https://github.com/dorayaki4369/laravel-decorator/blob/main/src/Attributes/DBTransactionDecorator.php#L20-L34

### `SimpleCacheDecorator`クラス
対象メソッドの引数をキーに、戻り値を値としてキャッシュし、2回目以降の呼び出し時にキャッシュされた値を変わりに返すデコレーターです。

https://github.com/dorayaki4369/laravel-decorator/blob/main/src/Attributes/SimpleCacheDecorator.php#L15-L28

### `ValidationDecorator`クラス
対象メソッドの引数に対し[バリデーション](https://laravel.com/docs/12.x/validation)を実行します。

https://github.com/dorayaki4369/laravel-decorator/blob/main/src/Attributes/ValidationDecorator.php#L24-L33

# 仕組み
このライブラリは、**Laravelのサービスコンテナ経由でクラス解決したとき、自動的にデコレーターを適用できる仕組み**になっています。
例えば、Laravel開発では、使いたいサービスを直接インスタンス化するのではなく、以下のようにコントローラや他のクラスにインジェクションして使うことがあります。

```php
<?php

namespace App\Controllers;

use App\Services\MyService;

class MyController
{
    public function index(MyService $service)
    {
        return $service->exec('hello', ['option' => 'value']); // LaravelがMyServiceを自動解決してくれる
    }
}
```

このときLaravelは、サービスコンテナを通じて`MyService`のインスタンスを作成します。
このライブラリでは、**そのインスタンス生成のタイミングをフックして、対象クラスを継承する匿名クラスを作成する**ことで、デコレーターの処理を差し込んでいます。

仕組みは以下の3フェーズで構成されています。

1. **スキャン**
   アプリケーション内のクラスを解析し、デコレーターが設定されているメソッドを探します。

2. **インスタンス生成**
   Laravelのサービスコンテナがクラスを解決しようとしたとき、元のクラスを継承（ラップ）した「デコレーター付きインスタンス」を返します。

3. **メソッド実行**
   対象メソッドが呼び出された際に、設定されたデコレーターを順番に実行します。

## 1. スキャン

アプリケーションの起動時などに、デコレーターを適用するクラス・メソッドを探します。

## 2. インスタンス生成
Laravelのサービスコンテナからクラスが解決されるタイミングで、ライブラリ側がそれをラップし、デコレーター処理付きの匿名インスタンスを返します。

https://github.com/dorayaki4369/laravel-decorator/blob/main/src/LaravelDecoratorServiceProvider.php#L45-L50

例えば、冒頭の`MyService`は以下のような匿名クラスが生成・実行されます。

```php
<?php

return new class extends \App\Services\MyService
{
    public function exec(string $str, array $options = []): string
    {
        return \Dorayaki4369\LaravelDecorator\Facades\Decorator::handle($this, __FUNCTION__, [$str, $options]);
    }
};
```

この仕組みによって、**呼び出し側のコードには一切影響を与えずに、共通処理を差し込むことが可能**になります。
また、[ライブラリの特徴](#ライブラリの特徴)で書いた制約の存在も、この生成方法が要因になっています。

## 3. 実行
メソッドが呼び出された瞬間、ライブラリは事前に設定されたデコレーターを順番に実行します。
各デコレーターは、`decorate()`メソッド内で`callable $next`を使って処理をラップしていきます。

https://github.com/dorayaki4369/laravel-decorator/blob/main/src/DecoratorHandler.php#L34-L41

このようにクロージャをチェーンすることで、ミドルウェアと似た動作をするようにしています。

# 今後の展望
機能は一通りできたと思うので、最適化が課題かなと思っています。
特に、匿名クラス作成時にPHPコードを生成していますが、その処理が重い気がするので、生成コードをキャッシュする機能をつけようと思っています。
キャッシュ機能は`decorator:cache`みたいなArtisanコマンドとして呼び出せるようにし、`optimize`コマンドにフックできるようにしたいです。

# まとめ
本記事では、自作のデコレーターライブラリについて解説しました。

+ クラスの共通処理（ログ・トランザクション・キャッシュなど）をスマートに書きたい
+ メソッド単位でアスペクト的に処理を挿入したい
+ Laravelの書き方をほとんど変えずに柔軟な設計をしたい

といったニーズに応えられると思うので、ぜひ気になった方はインストールして試してみてください！  
まだまだ改善の余地があるので、使ってみた感想や「こんな機能ほしい」などのフィードバックも大歓迎です 🚀