# チュートリアル: GraphQL APIのデザイン

本チュートリアルはもともと[Shopify](https://www.shopify.ca/)が社内利用のために作ったものですが、
GrahphQL APIを作る人すべてに便利だと思い今回公開することにしました。

本チュートリアルは3年以上にわたって、Shopifyでプロダクションのスキーマを
作成・拡張してきた中で得た教訓に基づいています。
本チュートリアルは改変され続けてきましたし、今後も変わり続けます。

大抵の場合このデザインガイドラインは適用できるとは信じていますが、
全部が全部使えるわけではないかもしれません。
社内でもガイドラインのルールのほとんどが100%そのまま使えるわけではないので、
議論したり例外を設けたりしています。
ですから、ただ盲目にここに書かれているルールを全てそのまま真似しないでください。
ユースケースにはまるものだけを使ってください。

## イントロダクション

ようこそ! このドキュメントでは新しい(あるいは既存のものに追加する場合の)GraphQL APIの設計について一つづつ段階を踏みながら説明していきます。
API設計は繰り返し・実験・ビジネスドメインの深い理解が強く求められる難しいタスクです。

## ステップ0: 背景

このチュートリアルを進めるに当たって、あなたはEコマースの会社で働いているとします。
現状、既存のGraphQL APIは商品情報の他にはほとんど情報を公開していません。
しかし、あなたのチームはバックエンドの"コレクション"を実装するプロジェクトをちょうど実装し終えたので、
APIで公開したいと考えています。

コレクションとは、商品をグループ化するための新たな便利な方法です。
たとえば、Tシャツのコレクションを持っているかもしれません。
コレクションは、ウェブサイトを閲覧する際の表示方法であったり、プログラム化されたタスクに使うことができます。
（たとえば、ある特定のコレクションの商品にのみ割引を適用したいことがあるかもしれません）。

バックエンドでは、次のように新しい機能が実装されています。
- すべてのコレクションは、タイトル、説明文 (HTMLフォーマットを含む)、画像のような幾つかのシンプルな属性を持っています。
- 2種類のコレクションがあります。コレクションに含まれる製品をリストアップする「マニュアル」コレクションと、ルールに従って商品をリストアップする「自動」コレクションです。
- 商品とコレクションの関係は多対多であるため、`CollectionMembership`というジョインテーブルを真ん中に持たせます。
- コレクションは、これまでの商品と同様に、公開されている（店頭に表示されている）か、公開されていないかのどちらかです。

このような背景がわかったので、API設計について考え始める準備が整いました。

## ステップ1: 俯瞰して見る

スキーマをナイーブに考える、以下のようになるかもしれません(`Product` のような既存の型は省略)。

```graphql
interface Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollection implements Collection {
  id: ID!
  rules: [AutomaticCollectionRule!]!
  rulesApplyDisjunctively: Bool!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type ManualCollection implements Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollectionRule {
  column: String!
  relation: String!
  condition: String!
}

type CollectionMembership {
  collectionId: ID!
  productId: ID!
}
```

4つのオブジェクトと1つのインターフェースだけとは言え、一見しただけではすでにかなり複雑になっています。
また、例えばモバイルアプリのコレクション機能を実装する場合に必要なすべてのAPIを実装しているわけではないことは自明です。
この時点ですべての機能を実装しているわけではないことははっきりわかります。

一歩前に戻ってみましょう。非常に複雑なGraphQL APIでは、沢山のオブジェクトが複数のパスや数多くのフィールドで関連づけられています。
このようなものを一度にデザインしようとすると、混乱や間違いのもとになります。
まず、特定のフィールドやmutationのことは考えるのではなく、
型とその関係だけに焦点を当て、よりハイレベルの視点から始めてください。
基本的にGraphQL固有の要素を持った、Entity-Relationshipモデルを想像してください。
ナイーブなスキーマを縮小すると次のようになります。

```graphql
interface Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [CollectionMembership]
}

type ManualCollection implements Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollectionRule { }

type CollectionMembership {
  Collection
  Product
}
```

簡略化するためにのスカラーフィールド、フィールド名、およびのNULL値をとるかどうかの情報などをすべてを省略しました。
簡略化の結果残ったのは、GraphQLのようなものですが、よりハイレベルのタイプとそれらの関係に集中できるようになっています。

*ルール1: 特定のフィールドを扱う前に、常にオブジェクトとその関係のハイレベルなビューから始めてください。

## ステップ2: 白紙の状態から始める

デザインを簡略化したので、このデザインの大きな欠陥に取り組むことができるようになりました。

前述のように、実装では、マニュアルおよび自動のコレクションの存在、およびジョインテーブルの使用が定義されています。
私たちのナイーブなAPI設計は、実装をもとに明確に構成されていましたが、これが間違いでした。

このアプローチの根本的な問題は、APIが実装とは異なる目的のために動作し、しばしば異なる抽象レベルで動作することです。
このようなケースでは、実装が私たちをさまざまな面で迷わせました。

### `CollectionMembership` を表現する

すでに目立っていて、かなりはっきりしている問題は、`CollectionMembership` 型がスキーマに含まれていることです。
コレクションの所属関係を表すテーブルは製品とコレクションの多対多の関係を表すために使用されます。
最後の文をもう一度読んでください。このテーブルが表すのは**製品とコレクションの間の関係**です。
セマンティック、ビジネスドメインの観点からみれば、コレクションの所属関係は
何とも関係ない実装の詳細です。

これは、このテーブルがAPIに属していないことを意味します。
代わりに、APIでは製品と実際のビジネスドメインの関係を直接公開するべきです。
コレクションの所属関係をスキーマから取り除くと、ハイレベルなデザインは次のようになります。

```graphql
interface Collection {
  Image
  [Product]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [Product]
}

type ManualCollection implements Collection {
  Image
  [Product]
}

type AutomaticCollectionRule { }
```

もとのデザインよりもかなり良いですね。

**ルール2: APIデザインに実装の詳細は決して含めないこと**

### コレクションを表現する

このAPIデザインにはまだ大きな欠陥が1つありますが、ビジネスドメインを完全に理解していなければ、それほど明白ではないでしょう。
既存の設計では、`AutomaticCollections`と`ManualCollections`はそれぞれが共通の`Collection`インタフェースを実装していますが、
2つの別の型としてモデル化されています。
これは直観的にはかなり理にかなっています。２つの型は共通のフィールドがたくさんありますが、2つの関係（`AutomaticCollections`はルールを持つ）や挙動を見ると依然として異なっていることがわかります。

しかし、ビジネスモデルの観点から見ると、これらの違いは基本的に実装の詳細です。
コレクションの挙動の定義は、製品をグループ化することであり、製品を選択する方法は補足的なものです。
ある時点で実装を拡張して3つ目の製品選択の方法（機械学習とか?）や選択方法の混ぜ合わせる（ルールや手動で追加された製品）などを許可するようにするかもしれませんが、**それらも依然としてコレクションである**ということが重要です。
もし仮に今すぐ選択方法をミックスできないのであれば、実装の失敗であると主張することさえできます。
要するにAPIの形は実際には次のようにすべきです。

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

これは素晴らしいですね。
現時点で`ManualCollections`にルールがあるように見えて気持ち悪いと思う方もいるかもしれませんが、この関係はリストであることに注意してください。
新しいAPIのデザインでは、 "マニュアルなコレクション"は単なるルールのリストが空であるコレクションと考えればよいのです。

### まとめ

この抽象的なレベルで最善のAPIデザインを選択するには、モデリングしている問題のドメインを非常に深く理解する必要があります。
チュートリアルの設定では、特定のトピックのコンテキストの深さを提供するのは難しいですが、
説明が理解できるくらいコレクションの設計が簡単だと思ってもらえれば幸いです。
たとえあなたがコレクションに対する深い理解を持っていなくても、実際にモデリングしているどんなドメインであれ、
このステップの話は絶対に必要です。 APIを設計する際には、これらの難しい質問を自分自身に尋ねることが非常に重要であり、
何でもかんでも実装に従うべきではありません。

今回の話と密接に関連した補足をしておきますが、良いAPIはユーザーインターフェイスをモデル化しません。
実装とUIは、インスピレーションとAPIデザインへの判断材料の両方になり得ますが、
最終的な意思決定要因は常にビジネスドメインでなければなりません。

もっと重要なのは、既存のREST APIの内容に必ずしも従う必要はないということです。
RESTとGraphQLの背後にある設計原則は全く異なる選択につながる可能性がありますので、
REST APIでうまくいくものがGraphQLの良い選択であるとは考えないでください。

可能な限り今持っている荷物を捨てて、一からスタートしてください。

**ルール3: 実装、ユーザーインターフェース、レガシーAPIではなく、ビジネスドメインを中心としたAPI設計をしてください。**

## ステップ3: 詳細を詰める


型をモデル化するクリーンな構造ができたので、ここまでは省略していたフィールドを戻して、
詳細なレベルで再びデザインを考えられるようになりました。

詳細を追加する前に、現時点で本当に必要かどうかを自問してください。
データベースのカラム、モデルのプロパティ、またはRESTの属性が存在するかもしれないというだけで、
必ずしもそれらをGraphQLのスキーマに追加する必要はありません。

スキーマ要素（フィールド、引数、型など）を公開するかどうかは実際のニーズとユースケースをもとに考えるべきです。
GraphQLのスキーマは要素を追加することで簡単に拡張することができますが、スキーマの変更や削除は重大な変更であり、はるかに困難です。

**ルール4: フィールドは削除するより追加するほうが簡単である**

### 出発点

元々あったフィールドを新しい構造に合わせて、もとに戻すと次のようになります。

```graphql
type Collection {
  id: ID!
  rules: [CollectionRule!]!
  rulesApplyDisjunctively: Bool!
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

解決しなければならない全く新しい設計の問題が出てきました。
フィールドを上から下へと順に見ていきながら修正していきましょう。

### IDと`Node` インターフェース

コレクション型の最初のフィールドはIDフィールドです。いたって普通ですね。
このIDは、特に変更や削除などの操作を実行する際に、APIを通じてコレクションを識別するのに必要となります。
しかし、一つ欠落しているものが1つあります。それは`Node` インターフェースです。
これは、非常に一般的に使用されるインターフェイスで、ほとんどのスキーマには既に存在します。
次のような構造です。

```graphql
interface Node {
  id: ID!
}
```
クライアントには、このオブジェクトが永続化され、指定されたIDで取得可能であることがわかり、
クライアントがローカルキャッシュなどの方法を正確かつ効率的に管理できるようになります。
識別可能な主なビジネスオブジェクト（製品、コレクションなど）のほとんどは、 `Node`を実装するべきです。


型の定義は次のように始まります。
```graphql
type Collection implements Node {
  id: ID!
  ...
}
```

**ルール5: 主なビジネスオブジェクトの型は常に`Node`を実装するべきである**

### ルールとサブオブジェクト

Collection型の次の2つのフィールド `rules`と`rulesApplyDisjunctively` は一緒にして考えます。前者はルールのリストであり、かなり簡単です。
リスト自体とリストの要素の両方がnullでないとマークされていることに注意してください。
GraphQLは`null`と`[]`と`[null]`を区別するため、これは問題ありません。
マニュアルなコレクションの場合、このリストは空でもかまいませんが、nullにすることも、nullを含めることもできません。

*Protip: リスト型フィールドは、ほとんどの場合、非nullであり、要素も非nullです。nullなリストを必要とする場合は、空のリストとnullのリストを区別できることに意味があることを確認してください。*


2番目のフィールドはちょっと変わっていて、ルールが論理的に適用されるかどうかを示すブール値のフィールドです。
これもnullではありませんが、一つの問題に直面します。マニュアルなコレクションではこのフィールドはどのような値を取るべきですか?
falseまたはtrueにするのは誤解を招く恐れがありますが、フィールドをnull可能にすると、自動のコレクションを扱うときには奇妙な3状態を表すフラグになります。
私たちはこれに疑問を呈していますが、伝えておくべきことがもう1つあります。
これらの2つのフィールドは明らかに複雑に関係しています。
これは意味的にそうであり、共有なプレフィックスを持つ名前にしたことからも見て取れます。
何らかの形でこの関係をスキーマに示す方法はあるのでしょうか。

実際には、根本的な実装からさらに逸脱し、直接のモデルには対応しない新しいGraphQL型である`CollectionRuleSet`を導入することで、これらの問題のすべてを解決することができます。
値と動作がリンクされている密接に関連したフィールドがある場合、これはしばしば正当なものです。
APIレベルで2つのフィールドを独自のタイプにグループ化することで、明確で意味のある指標を提供し、
null許容性に関するすべての問題を解決します。
マニュアルコレクションの場合はルールセットはnullを取ります。
ブール値フィールドはnull以外のままにできます。以上から次のような設計になります。

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Protip: リストと同様、ブール値フィールドはほとんどの場合nullではありません。 null可能なブール値を必要とする場合は、3つの状態（null / false / true）を区別することに、本当に意味的価値があるかどうかを確認し、より大きな設計上の欠陥ではないことを確かめてください*

*ルール6: 密接に関連するフィールドをサブオブジェクトにグループ化する。*

### リストとページネーション

次に考慮すべき塊は`products`フィールドです。
`CollectionMembership`型を削除したときに、この関係をすでに「固定」したので、
このフィールドは特に問題ないと思うかもしれません。
しかし、実は一つ問題があります。

現在のところこのフィールドは製品を要素とする配列を返すように定義されていますが、
コレクションは何十万もの製品を持っているかもしれません。
なので、これらの製品すべてを一つの配列に集めるのは非常に高価で非効率です。
このようなケースを解決するために、GraphQLにはリストのページネーションがあります。

オブジェクトを複数返すフィールドや関係を実装する際には、そのフィールドは何個のオブジェクトを持ちうるか、どの量まで達すると病的かなど、
常にそのフィールドをページネーションにするべきか自問してください。

フィールドをページネーションにするということは、ページネーションをまず実装する必要があるということです。
このチュートリアルでは、[Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm)で定義されている[Connections](https://graphql.org/learn/pagination/#complete-connection-model)を使います。

この場合、製品フィールドをページネーションにするには、フィールドの定義を`products: ProductConnection!`にするだけで済みます。
connectionsが実装済みであると仮定すると、型は次のようになるでしょう。

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}
```
*ルール7: フィールドをページネーションにするべきか常に確認してください。*

### Strings

Next up is the `title` field. This one is legitimately fine the way it is. It's
a simple string, and it's marked non-null because all collections must have a
title.

*Protip: As with booleans and lists, it's worth noting that GraphQL does
distinguish between empty strings (`""`) and nulls (`null`), so if you need a
nullable string make sure there is a legitimate semantic difference between
not-present (`null`) and present-but-empty (`""`). You can often think of empty
strings as meaning "applicable, but not populated", and null strings meaning
"not applicable".*

### IDs and Relations

Now we come to the `imageId` field. This field is a classic example of what
happens when you try and apply REST designs to GraphQL. In REST APIs it's
pretty common to include the IDs of other objects in your response as a way to
link together those objects, but this is a major anti-pattern in GraphQL.
Instead of providing an ID, and forcing the client to do another round-trip to
get any information on the object, we should just include the object directly
into the graph - that's what GraphQL is for after all. In REST APIs this pattern
often isn't practical, since it inflates the size of the response significantly
when the included objects are large. However, this works fine in GraphQL because
every field must be explicitly queried or the server won't return it.

As a general rule, the only ID fields in your design should be the IDs of the
object itself. Any time you have some other ID field, it should probably be an
object reference instead. Applying this to our schema so far, we get:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  bodyHtml: String
}

type Image {
  id: ID!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Rule #8: Always use object references instead of ID fields.*

### Naming and Scalars

The last field in our simple `Collection` type is `bodyHtml`. To a user who is
unfamiliar with the way that collections were implemented, it's not entirely
obvious what this field is for; it's the body description of the specific
collection. The first thing we can do to make this API better is just to rename
it to `description`, which is a much clearer name.

*Rule #9: Choose field names based on what makes sense, not based on the
implementation or what the field is called in legacy APIs.*

Next, we can make it non-nullable. As we talked about with the title field, it
doesn't make sense to distinguish between the field being null and simply being
an empty string, so we don't expose that in the API. Even if your database
schema does allow records to have a null value for this column, we can hide that
at the implementation layer.

Finally, we need to consider if `String` is actually the right type for this
field. GraphQL provides a decent set of built-in scalar types (`String`, `Int`,
`Boolean`, etc) but it also lets you define your own, and this is a prime use
case for that feature. Most schemas define their own set of additional scalars
depending on their use cases. These provide additional context and semantic
value for clients. In this case, it probably makes sense to define a custom
`HTML` scalar for use here (and potentially elsewhere) when the string in
question must be valid HTML.

Whenever you're adding a scalar field, it's worth checking your existing list of
custom scalars to see if one of them would be a better fit. If you're adding a
field and you think a new custom scalar would be appropriate, it's worth talking
it over with your team to make sure you're capturing the right concept.

*Rule #10: Use custom scalar types when you're exposing something with specific
semantic value.*

### Pagination Again

That covers all of the fields in our core `Collection` type. The next object is
`CollectionRuleSet`, which is quite simple. The only question here is whether or
not the list of rules should be paginated. In this case the existing array
actually makes sense; paginating the list of rules would be overkill. Most
collections will only have a handful of rules, and there isn't a good use case
for a collection to have a large rule set. Even a dozen rules is probably an
indicator that you need to rethink that collection, or should just be manually
adding products.

### Enums

This brings us to the final type in our schema, `CollectionRule`. Each rule
consists of a column to match on (e.g. product title), a type of relation (e.g.
equality) and an actual value to use (e.g. "Boots") which is confusingly called
`condition`. That last field can be renamed, and so should `column`; column is
very database-specific terminology, and we're working in GraphQL. `field` is
probably a better choice.

As far as types go, both `field` and `relation` are probably implemented
internally as enumerations (assuming your language of choice even has
enumerations). Fortunately GraphQL has enums as well, so we can convert those
two fields to enums. Our completed schema design now looks like this:

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  description: HTML!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Bool!
}

type CollectionRule {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}

enum CollectionRuleField {
  TAG
  TITLE
  TYPE
  INVENTORY
  PRICE
  VENDOR
}

enum CollectionRuleRelation {
  CONTAINS
  ENDS_WITH
  EQUALS
  GREATER_THAN
  LESS_THAN
  NOT_CONTAINS
  NOT_EQUALS
  STARTS_WITH
}
```

*Rule #11: Use enums for fields which can only take a specific set of values.*

## Step Four: Business Logic

We now have a minimal but well-designed GraphQL API for collections. There is a
lot of detail to collections that we haven't dealt with - any real
implementation of this feature would need a lot more fields to deal with things
like product sort order, publishing, etc. - but as a rule those fields will all
follow the same design patterns layed out here. However, there are still a few
things which bear looking at in more detail.

For this section, it is most convenient to start with a motivating use case from
the hypothetical client of our API. Let us therefore imagine that the client
developer we have been working with needs to know something very specific:
whether a given product is a member of a collection or not. Of course this is
something that the client can already answer with our existing API: we expose
the complete set of products in a collection, so the client simply has to
iterate through looking for the product they care about.

This solution has two problems though. The first, obvious problem is that it's
inefficient; collections can contain millions of products, and having the client
fetch and iterate through them all would be extremely slow. The second, bigger
problem, is that it requires the client to write code. This last point is a
critical piece of design philosophy: the server should always be the single
source of truth for any business logic. An API almost always exists to serve
more than one client, and if each of those clients has to implement the same
logic then you've effectively got code duplication, with all the extra work and
room for error which that entails.

*Rule #12: The API should provide business logic, not just data. Complex
calculations should be done on the server, in one place, not on the client, in
many places.*

Back to our client use-case, the best answer here is to provide a new field
specifically dedicated to solving this problem. Practically, this looks like:
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Bool!
}
```
This field takes the ID of a product and returns a boolean based on the server
determining if a product is in the collection or not. The fact that this sort-of
duplicates the data from the existing `products` field is irrelevant. GraphQL
returns only what clients explicitly ask for, so unlike REST it does not cost us
anything to add a bunch of secondary fields. The client doesn't have to write
any code beyond querying an additional field, and the total bandwidth used is a
single ID plus a single boolean.

One follow-up warning though: just because we're providing business logic in a
situation does not mean we don't have to provide the raw data too. Clients
should be able to do the business logic themselves, if they have to. You can’t
predict all of the logic a client is going to want, and there isn't always an
easy channel for clients to ask for additional fields (though you should strive
to ensure such a channel exists as much as possible).

*Rule #13: Provide the raw data too, even when there's business logic around it.*

Finally, don't let business-logic fields affect the overall shape of the API.
The business domain data is still the core model. If you're finding the business
logic doesn't really fit, then that's a sign that maybe your underlying model
isn't right.

## Step Five: Mutations

The final missing piece of our GraphQL schema design is the ability to actually
change values: creating, updating, and deleting collections and related pieces.
As with the readable portion of the schema we should start with a high-level
view: in this case, of just the various mutations we will want to implement,
without worrying about their specific inputs or outputs. Naively we might follow
the CRUD paradigm and have just `create`, `delete`, and `update` mutations.
While this is a decent starting place, it is insufficient for a proper GraphQL
API.

### Separate Logical Actions

The first thing we might notice if we were to stick to just CRUD is that our
`update` mutation quickly becomes massive, responsible not just for updating
simple scalar values like title but also for performing complex actions like
publishing/unpublishing, adding/removing/reordering the products in the
collection, changing the rules for automatic collections, etc. This makes it
hard to implement on the server and hard to reason about for the client.
Instead, we can take advantage of GraphQL to split it apart into more granular,
logical actions. As a very first pass, we can split out publish/unpublish
resulting in the following mutation list:
- create
- delete
- update
- publish
- unpublish

*Rule #14: Write separate mutations for separate logical actions on a resource.*

### Manipulating Relationships

The `update` mutation still has far too many responsibilities so it makes sense
to continue splitting it up, but we will deal with these actions separately
since they're worth thinking about from another dimension as well: the
manipulation of object relationships (e.g. one-to-many, many-to-many). We've
already considered the use of IDs vs embedding, and the use of pagination vs
arrays in the read API, and there are some similar issues to deal with when
mutating these relationships.

For the relationship between products and collections, there are a couple of
styles we could broadly consider:
- Embedding the entire relationship (e.g. `products: [ProductInput!]!`) into the
  update mutation is the CRUD-style default, but of course it quickly becomes
  inefficient when the list is large.
- Embedding "delta" fields (e.g. `productsToAdd: [ID!]!` and
  `productsToRemove: [ID!]!`) into the update mutation is more efficient since
  only the changed IDs need to be specified instead of the entire list, but it
  still keeps the actions tied together.
- Splitting it up entirely into separate mutations (`addProduct`,
  `removeProduct`, etc.) is the most powerful and flexible but also the most
  work.

The last option is generally the safest call, especially since mutations like
this will usually be distinct logical actions anyway. However, there are a lot
of factors to consider:
- Is the relationship large or paginated? If so, embedding the entire list is
  definitely impractical, however either delta fields or separate mutations
  could still work. If the relationship is always small though (especially if
  it's one-to-one), embedding may be the simplest choice.
- Is the relationship ordered? The product-collection relationship is ordered,
  and permits manual reordering. Order is naturally supported by the embedded
  list or by separate mutations (you can add a `reorderProducts` mutation)
  but isn't an option for delta fields.
- Is the relationship mandatory? Products and collections can both exist on
  their own outside of the relationship, with their own create/delete lifecycle.
  If the relationship were mandatory (i.e. products must be in a collection)
  then this would strongly suggest separate mutations because the action would
  actually be to *create* a product, not just to update the relationship.
- Do both sides have IDs? The collection-rule relationship is mandatory (rules
  can't exist without collections) but rules don't even have IDs; they are
  clearly subservient to their collection, and since the relationship is also
  small, embedding the list is actually not a bad choice here. Anything else
  would require rules to be individually identifiable and that feels like
  overkill.

*Rule #15: Mutating relationships is really complicated and not easily
 summarized into a snappy rule.*

 If you stir all of this together, for collections we end up with the following
 list of mutations:
- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

Products we split into their own mutations, because the relationship is large,
and ordered. Rules we left inline because the relationship is small, and rules
are sufficiently minor to not have IDs.

Finally, you may note that we have mutations like `addProducts` and not
`addProduct`. This is simply a convenience for the client, since the common use
case when manipulating this relationship will be to add, remove, or reorder more
than one product at a time.

*Rule #16: When writing separate mutations for relationships, consider whether
 it would be useful for the mutations to operate on multiple elements at once.*

### Input: Structure, Part 1

Now that we know which mutations we want to write, we get to figure out what
their input structures look like. If you've been browsing any of the real
production schemas that are publicly available, you may have noticed that many
mutations define a single global `Input` type to hold all of their arguments:
this pattern was a requirement of some legacy clients but is no longer needed
for new code; we can ignore it.

For many simple mutations, an ID or a handful of IDs are all that is needed,
making this step quite simple. Among collections, we can quickly knock out the
following mutation arguments:
- `delete`, `publish` and `unpublish` all simply need a single collection ID
- `addProducts` and `removeProducts` both need the collection ID as well as a
  list of product IDs
This leaves us with only three remaining "complicated" inputs to design:
- create
- update
- reorderProducts

Let's start with create. A very naive input might look kind of like our original
naive collection model when we started, but we can already do better than that.
Based on our final collection model and the discussion of relationships above,
we can start with something like this:

```graphql
type Mutation {
  collectionDelete(id: ID!)
  collectionPublish(collectionId: ID!)
  collectionUnpublish(collectionId: ID!)
  collectionAddProducts(collectionId: ID!, productIds: [ID!]!)
  collectionRemoveProducts(collectionId: ID!, productIds: [ID!])
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: HTML!)
}

input CollectionRuleSetInput {
  rules: [CollectionRuleInput!]!
  appliesDisjunctively: Bool!
}

input CollectionRuleInput {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}
```

First a quick note on naming: you'll notice that we named all of our mutations
in the form `collection<Action>` rather than the more naturally-English
`<action>Collection`. Unfortunately, GraphQL does not provide a method for
grouping or otherwise organizing mutations, so we are forced into
alphabetization as a workaround. Putting the core type first ensures that all of
the related mutations group together in the final list.

*Rule #17: Prefix mutation names with the object they are mutating for
 alphabetical grouping (e.g. use `orderCancel` instead of `cancelOrder`).*

### Input: Scalars

This draft is a lot better than a completely naive approach, but it still isn't
perfect. In particular, the `description` input field has a couple of issues. A
non-null `HTML` field makes sense for the output of a collection's description,
but it doesn't work as well for input for a couple of reasons. First-off, while
`!` denotes non-nullability on output, it doesn't mean quite the same thing on
input; instead it denotes more the concept of whether a field is "required". A
required field is one the client must provide in order for the request to
proceed, and this isn't true for `description`. We don't want to prevent clients
from creating collections if they don't provide a description (or equivalently,
we don't want to force them to provide a useless `""`), so we should make
`description` non-required.

*Rule #18: Only make input fields required if they're actually semantically
 required for the mutation to proceed.*

The other issue with `description` is its type; this may seem counter-intuitive
since it is already strongly-typed (`HTML` instead of `String`) and we've been
all about strong typing so far. But again, inputs behave a little differently.
Validation of strong typing on input happens at the GraphQL layer before any
"userspace" code gets run, which means that realistically clients have to deal
with two layers of errors: GraphQL-layer validation errors, and business-layer
validation errors (for example something like: you've reached the limit of
collections you can create with your current storage). In order to simplify this
process, we intentionally weakly type input fields when it might be difficult
for the client to validate up-front. This lets the business-logic side handle
all of the validation, and lets the client only deal with errors from one spot.

*Rule #19: Use weaker types for inputs (e.g. `String` instead of `Email`) when
 the format is unambiguous and client-side validation is complex. This lets the
 server run all non-trivial validations at once and return the errors in a
 single place in a single format, simplifying the client.*

It is important to note, though, that this is not an invitation to weakly-type
all your inputs. We still use strongly-typed enums for the `field` and
`relation` values on our rule input, and we would still use strong typing for
certain other inputs like `DateTime`s if we had any in this example. The key
differentiating factors are the complexity of client-side validation, and the
ambiguity of the format. HTML is a well-defined, unambiguous specification, but
is quite complex to validate. On the other hand, there are hundreds of ways to
represent a date or time as a string, all of them reasonably simple, so it
benefits from a strong scalar type to specify which format we expect.

*Rule #20: Use stronger types for inputs (e.g. `DateTime` instead of `String`)
 when the format may be ambiguous and client-side validation is simple. This
 provides clarity and encourages clients to use stricter input controls (e.g. a
 date-picker widget instead of a free-text field).*

### Input: Structure, Part 2

Continuing on to the update mutation, it might look something like this:

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(id: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

You'll note that this is very similar to our create mutation, with two
differences: an `id` argument was added which determines which collection to
update, and `title` is no longer required since the collection must already have
one.  Ignoring the title's required status for a moment our example mutations
have four duplicate arguments, and a complete collections model would include
quite a few more.

While there are some arguments for leaving these mutations as-is, we have
decided that situations like this call for DRYing up the common portions of the
arguments, even at the cost of requiredness. This has a couple of advantages:
- We end up with a single input object representing the concept of a collection
  and mirroring the single `Collection` type our schema already has.
- Clients can share code between their create and update forms (a common
  pattern) because they end up manipulating the same kind of input object.
- Mutations remain slim and readable with only a couple of top-level arguments.

The primary cost, of course, is that it's no longer clear from the schema that
the title is required on creation. Our schema ends up looking like this:

```graphql
type Mutation {
  # ...
  collectionCreate(collection: CollectionInput!)
  collectionUpdate(id: ID!, collection: CollectionInput!)
}

input CollectionInput {
  title: String
  ruleSet: CollectionRuleSetInput
  image: ImageInput
  description: String
}
```

*Rule #21: Structure mutation inputs to reduce duplication, even if this
 requires relaxing requiredness constraints on certain fields.*

### Output

The final design question we need to deal with is the return value of our
mutations. Typically mutations can succeed or fail, and while GraphQL does
include explicit support for query-level errors, these are not ideal for
business-level mutation failures. Instead, we reserve these top-level errors for
failures of the client (e.g. requesting a non-existant field) rather than of the
user. As such, each mutation should define a "payload" type which includes a
user-errors field in addition to any other values that might be useful. For
create, that might look like this:

```graphql
type CollectionCreatePayload {
  userErrors: [UserError!]!
  collection: Collection
}

type UserError {
  message: String!

  # Path to input field which caused the error.
  field: [String!]
}
```

Here, a successful mutation would return an empty list for `userErrors` and
would return the newly-created collection for the `collection` field. An
unsuccessful one would return one or more `UserError` objects, and null for the
collection.

*Rule #22: Mutations should provide user/business-level errors via a
 `userErrors` field on the mutation payload. The top-level query errors entry is
 reserved for client and server-level errors.*

In many implementations, much of this structure is provided automatically and
all you will have to define is the `collection` return field.

For the update mutation, we follow exactly the same pattern:

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

It's worth noting that `collection` is still nullable even here, since if the
provided ID doesn't represent a valid collection there is no collection to
return.

*Rule #23: Most payload fields for a mutation should be nullable, unless there
 is really a value to return in every possible error case.*

## Conclusion

Thank you for reading our tutorial! Hopefully by this point you have a solid
idea of how to design a good GraphQL API.

Once you've designed an API you're happy with, it's time to implement it!
