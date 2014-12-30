リソース
========

RESTful API は、つまるところ、*リソース* にアクセスし、それを操作するものです。
MVC の枠組の中では、リソースは [models](structure-models.md) として見ることが出来ます。

リソースをどのように表現すべきかについて制約がある訳ではありませんが、Yii においては、通常は、次のような理由によって、リソースを [[yii\base\Model]] またはその子クラス (例えば [[yii\db\ActiveRecord]]) のオブジェクトとして表現することになります。

* [[yii\base\Model]] は [[yii\base\Arrayable]] インタフェイスを実装しています。
  これによって、リソースのデータを RESTful API を通じて公開する仕方をカスタマイズすることが出来ます。
* [[yii\base\Model]] は [入力値のバリデーション](input-validation.md) をサポートしています。
  これは、RESTful API がデータ入力をサポートする必要がある場合に役に立ちます。
* [[yii\db\ActiveRecord]] は DB データのアクセスと操作に対する強力なサポートを提供しています。
  リソースデータがデータベースに保存されているときは、アクティブレコードが最適の選択です。

この節では、主として、[[yii\base\Model]] クラス (またはその子クラス) から拡張したリソースクラスにおいて、どのデータを RESTful API を通じて返すことが出来るかを指定する方法を説明します。
リソースクラスが [[yii\base\Model]] から拡張しない場合は、全てのパブリックなメンバ変数が返されます。


## フィールド <a name="fields"></a>

RESTful API のレスポンスにリソースを含めるとき、リソースは文字列にシリアライズされる必要があります。
Yii はこのプロセスを二つのステップに分けます。
最初に、リソースは [[yii\rest\Serializer]] によって配列に変換されます。
次に、その配列が [[yii\web\ResponseFormatterInterface|レスポンスフォーマッタ]] によって、リクエストされた形式 (例えば JSON や XML) の文字列にシリアライズされます。
リソースクラスを開発するときに主として力を注ぐべきなのは、最初のステップです。

[[yii\base\Model::fields()|fields()]] および/または [[yii\base\Model::extraFields()|extraFields()]] をオーバーライドすることによって、リソースのどういうデータ (*フィールド* と呼ばれます) を配列表現に入れることが出来るかを指定することが出来ます。
この二つのメソッドの違いは、前者が配列表現に含まれるべきフィールドのデフォルトのセットを指定するのに対して、後者はエンドユーザが `expand` クエリパラメータで要求したときに配列に含めることが出来る追加のフィールドを指定する、という点にあります。
例えば、

```
// fields() に宣言されている全てのフィールドを返す。
http://localhost/users

// id と email のフィールドだけを返す (ただし、fields() で宣言されているなら) 。
http://localhost/users?fields=id,email

// fields() の全てのフィールドと profile のフィールドを返す (ただし、profile が extraFields() で宣言されているなら)。
http://localhost/users?expand=profile

// id、email、profile のフィールドだけを返す (ただし、それらが fields() と extraFields() で宣言されているなら)。
http://localhost/users?fields=id,email&expand=profile
```


### fields()` をオーバーライドする <a name="overriding-fields"></a>

デフォルトでは、[[yii\base\Model::fields()]] は、モデルの全ての属性をフィールドとして返し、[[yii\db\ActiveRecord::fields()]] は、DB から投入された属性だけを返します。

`fields()` をオーバーライドして、フィールドを追加、削除、名前変更、または再定義することが出来ます。
`fields()` の返り値は配列でなければなりません。
配列のキーはフィールド名であり、配列の値は対応するフィールドの定義です。
フィールドの定義は、プロパティ/属性の名前か、あるいは、対応するフィールドの値を返す無名関数とすることが出来ます。
フィールド名がそれを定義する属性名と同一であるという特殊な場合においては、配列のキーを省略することが出来ます。
例えば、

```php
// 明示的に全てのフィールドをリストする方法。(API の後方互換性を保つために) DB テーブルやモデル属性の
// 変更がフィールドの変更を引き起こさないことを保証したい場合に適している。
public function fields()
{
    return [
        // フィールド名が属性名と同じ
        'id',
        // フィールド名は "email"、対応する属性名は "email_address"
        'email' => 'email_address',
        // フィールド名は "name"、その値は PHP コールバックで定義
        'name' => function ($model) {
            return $model->first_name . ' ' . $model->last_name;
        },
    ];
}

// いくつかのフィールドを除去する方法。親の実装を継承しつつ、公開すべきでないフィールドを
// 除外したいときに適している。
public function fields()
{
    $fields = parent::fields();

    // 公開すべきでない情報を含むフィールドを削除する
    unset($fields['auth_key'], $fields['password_hash'], $fields['password_reset_token']);

    return $fields;
}
```

> Warning|警告: 既定ではモデルの全ての属性がエクスポートされる配列に含まれるため、データを精査して、
> 公開すべきでない情報が含まれていないことを確認すべきです。そういう情報がある場合は、
> `fields()` をオーバーライドして、除去すべきです。上記の例では、`auth_key`、`password_hash`
> および `password_reset_token` を選んで除去しています。


### `extraFields()` をオーバーライドする<a name="overriding-extra-fields"></a>

デフォルトでは、[[yii\base\Model::extraFields()]] は何も返しませんが、[[yii\db\ActiveRecord::extraFields()]] は DB から取得されたリレーションの名前を返します。

`extraFields()` によって返されるデータの形式は `fields()` のそれと同じです。
通常、`extraFields()` は、主として、値がオブジェクトであるフィールドを指定するのに使用されます。
例えば、次のようなフィールドの宣言があるとしましょう。

```php
public function fields()
{
    return ['id', 'email'];
}

public function extraFields()
{
    return ['profile'];
}
```

`http://localhost/users?fields=id,email&expand=profile` というリクエストは、次のような JSON データを返すことが出来ます。

```php
[
    {
        "id": 100,
        "email": "100@example.com",
        "profile": {
            "id": 100,
            "age": 30,
        }
    },
    ...
]
```


## リンク <a name="links"></a>

[HATEOAS](http://en.wikipedia.org/wiki/HATEOAS) は、Hypermedia as the Engine of Application State (アプリケーション状態のエンジンとしてのハイパーメディア) の略称です。
HATEOAS は、RESTful API は自分が返すリソースについて、どのようなアクションがサポートされているかをクライアントが発見できるような情報を返すべきである、という概念です。
HATEOAS のキーポイントは、リソースデータが API によって提供されるときには、関連する情報を一群のハイパーリンクによって返すべきである、ということです。

あなたのリソースクラスは、[[yii\web\Linkable]] インタフェイスを実装することによって、HATEOAS をサポートすることが出来ます。
このインタフェイスは、[[yii\web\Link|リンク]] のリストを返すべき [[yii\web\Linkable::getLinks()|getLinks()]] メソッド一つだけを含みます。
典型的には、少なくとも、リソースオブジェクトそのものへの URL を表現する `self` リンクを返さなければなりません。
例えば、

```php
use yii\db\ActiveRecord;
use yii\web\Link;
use yii\web\Linkable;
use yii\helpers\Url;

class User extends ActiveRecord implements Linkable
{
    public function getLinks()
    {
        return [
            Link::REL_SELF => Url::to(['user/view', 'id' => $this->id], true),
        ];
    }
}
```

`User` オブジェクトがレスポンスで返されるとき、レスポンスはそのユーザに関連するリンクを表現する `_links` 要素を含むことになります。
例えば、

```
{
    "id": 100,
    "email": "user@example.com",
    // ...
    "_links" => [
        "self": "https://example.com/users/100"
    ]
}
```


## コレクション <a name="collections"></a>

リソースオブジェクトは *コレクション* としてグループ化することが出来ます。
各コレクションは、同じ型のリソースのリストを含みます。

コレクションは配列として表現することも可能ですが、通常は、[データプロバイダ](output-data-providers.md) として表現する方がより望ましい方法です。
これは、データプロバイダがリソースの並べ替えとページネーションをサポートしているからです。
並べ替えとページネーションは、コレクションを返す RESTful API にとっては、普通に必要とされる機能です。
例えば、次のアクションは投稿のリソースについてデータプロバイダを返すものです。

```php
namespace app\controllers;

use yii\rest\Controller;
use yii\data\ActiveDataProvider;
use app\models\Post;

class PostController extends Controller
{
    public function actionIndex()
    {
        return new ActiveDataProvider([
            'query' => Post::find(),
        ]);
    }
}
```

データプロバイダが RESTful API のレスポンスで送信される場合は、[[yii\rest\Serializer]] が現在のページのリソースを取り出して、リソースオブジェクトの配列としてシリアライズします。
それだけでなく、[[yii\rest\Serializer]] は次の HTTP ヘッダを使ってページネーション情報もレスポンスに含めます。

* `X-Pagination-Total-Count`: リソースの総数
* `X-Pagination-Page-Count`: ページ数
* `X-Pagination-Current-Page`: 現在のページ (1 から始まる)
* `X-Pagination-Per-Page`: 各ページのリソース数
* `Link`: クライアントがリソースをページごとにたどることが出来るようにするための一群のナビゲーションリンク

その一例を [クイックスタート](rest-quick-start.md#trying-it-out) の節で見ることが出来ます。