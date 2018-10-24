# チュートリアル: GraphQL APIのデザイン

本チュートリアルはもともと[Shopify](https://www.shopify.ca/)が社内利用のために作ったものですが、
GrahphQL APIを作る人すべてに便利だと思い今回公開することにしました。

3年以上にわたって、Shopifyでプロダクションのスキーマを
作成・拡張してきた中で得た教訓にもとに本チュートリアルを作成しました。
本チュートリアルは改変され続けてきましたし、今後も変わり続けます。

大抵の場合このデザインガイドラインは適用できるとは信じていますが、
全部が全部使えるわけではないかもしれません。
社内でもガイドラインのルールのほとんどが100%そのまま使えるわけではないので、
議論したり例外を設けたりしています。
ですから、ただ盲目的にここに書かれているルールを全てそのまま真似しないでください。
ご自身のユースケースにはまるものだけを使ってください。

## イントロダクション

ようこそ! このドキュメントでは新しい(あるいは既存のものに追加する場合の)GraphQL APIの設計について一つずつ段階を踏みながら説明していきます。
API設計は繰り返し・実験・ビジネスドメインの深い理解が強く求められる難しいタスクです。

## ステップ0: 背景

このチュートリアルを進めるに当たって、あなたはEコマースの会社で働いているとします。
現状、既存のGraphQL APIは商品情報の他にはほとんど情報を公開していません。
しかし、あなたのチームはバックエンドの"コレクション"を実装するプロジェクトをちょうど実装し終えたので、
APIで公開したいと考えています。

コレクションとは、商品をグループ化するための新たな便利な方法です。
たとえば、Tシャツのコレクションを持っているかもしれません。
コレクションは、ウェブサイトを閲覧する際の表示方法であったり、プログラム化されたルールに使うことができます。
（たとえば、ある特定のコレクションの商品にのみ割引を適用したいことがあるかもしれません）。

バックエンドでは、次のように新しい機能が実装されています。
- すべてのコレクションは、タイトル、説明文 (HTMLフォーマットを含む)、画像のような幾つかのシンプルな属性を持っています。
- 2種類のコレクションがあります。コレクションに含まれる商品をリストアップする「マニュアル」コレクションと、ルールに従って商品をリストアップする「自動」コレクションです。
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
コレクションの所属関係を表すテーブルは商品とコレクションの多対多の関係を表すために使用されます。
最後の文をもう一度読んでください。このテーブルが表すのは**商品とコレクションの間の関係**です。
セマンティック、ビジネスドメインの観点からみれば、コレクションの所属関係は
何とも関係ない実装の詳細です。

これは、このテーブルがAPIに属していないことを意味します。
代わりに、APIでは商品と実際のビジネスドメインの関係を直接公開するべきです。
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
コレクションの挙動の定義は、商品をグループ化することであり、商品を選択する方法は補足的なものです。
ある時点で実装を拡張して3つ目の商品選択の方法（機械学習とか?）や選択方法の混ぜ合わせる（ルールや手動で追加された商品）などを許可するようにするかもしれませんが、**それらも依然としてコレクションである**ということが重要です。
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
識別可能な主なビジネスオブジェクト（商品、コレクションなど）のほとんどは、 `Node`を実装するべきです。


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

現在のところこのフィールドは商品を要素とする配列を返すように定義されていますが、
コレクションは何十万もの商品を持っているかもしれません。
なので、これらの商品すべてを一つの配列に集めるのは非常に高価で非効率です。
このようなケースを解決するために、GraphQLにはリストのページネーションがあります。

オブジェクトを複数返すフィールドや関係を実装する際には、そのフィールドは何個のオブジェクトを持ちうるか、どの量まで達すると病的かなど、
常にそのフィールドをページネーションにするべきか自問してください。

フィールドをページネーションにするということは、ページネーションをまず実装する必要があるということです。
このチュートリアルでは、[Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm)で定義されている[Connections](https://graphql.org/learn/pagination/#complete-connection-model)を使います。

この場合、商品フィールドをページネーションにするには、フィールドの定義を`products: ProductConnection!`にするだけで済みます。
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

次は `title`フィールドです。これは見ればわかる通り問題ないです。
単純な文字列であり、すべてのコレクションにはタイトルが必要なので非nullとなっています。

*Protip: ブール値やリストと同様に、GraphQLは空文字列（`""`）とnull（`null`）を区別するので注意しましょう。
なので、null可能な文字列が必要な場合は、 「存在しない」 (`null`）と「存在するが空」（``)との間にちゃんとした意味の違いがあることを確認してください。
空文字列は、「適用可能だが値が入ってない」とみなし、空の文字列は「適用外」を意味すると考えられるケースは多々あると思います。*

### IDとリレーション
やっと`imageId`フィールドに来ました。このフィールドは、RESTデザインをGraphQLに適用しようとするときに起きることの典型的な例です。
REST APIでは、他のオブジェクトと紐付けるために、それらのIDをレスポンスに含めるのが一般的ですが、これはGraphQLではメジャーなアンチパターンです。
IDを使ってクライアントがオブジェクトに関する情報を取得するために別のラウンドトリップを実行するのではなく、オブジェクトをグラフに直接含める必要があります。これこそがGraphQLの本来のな目的です。REST APIにおいてこのパターンは、含まれているオブジェクトが大きい場合にレスポンスサイズが大きくなってしまうので実用的ではないことが多いです。しかし、サーバーはクエリに含まれないフィールドは返さないため、この方法はGraphQLでは問題ありません。

原則として、IDフィールドはオブジェクト自体のIDにとどめるようにデザインすべきです。
他のIDフィールドがあるということは、おそらくオブジェクトの参照になってしまっているはずです。
以上の考え方を今までのスキーマに適用すると、次のようになります。

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

*ルール8: IDフィールドの代わりに常にオブジェクト参照を使ってください*

### ネーミングとスカラー

単純な`Collection`型の最後のフィールドは`bodyHtml`です。
コレクションの実装方法になじみのないユーザーには、このフィールドの目的は全くはっきりしませんが、
特定のコレクションの内容説明のためのフィールドです。
このAPIを改善するために、まず名前を`description`に変更すべきです。これではるかにわかりやすいな名前になりました。

*ルール9: 実装やレガシーAPIでの名前に頼らずに、わかりやすい名前を選んでください。*

次に、`description`フィールドnull不可能にします。`title`フィールドのときに説明したように、
このフィールドがnullであることと単に空の文字列であることを区別することには意味がありません。
なので、APIでそのフィールドをnull可能として公開しないでください。
データベーススキーマでレコードがこの列のnull値を持つようにしたとしても、実装レイヤーで対応できます。

最後に、`String`が本当にこのフィールドの適切な型であるかどうかを考えましょう。
GraphQLにはビルトインのスカラー型（`String`、`Int`、`Boolean`など）がありますが、独自のスカラー型を定義することもできます。
これはその機能の主要な使用例です。ユースケースに応じて独自のスカラーセットを追加で定義しているスキーマも少なくありません。
独自のスカラー型はクライアントに追加のコンテキストと意味的価値を提供します。
今回のケースでは、文字列が有効なHTMLでなければならないのでカスタムの`HTML`スカラーを定義する価値はおそらくあるでしょう。(もしかしたら別の場所でも使えるかもしれません）

スカラーフィールドを追加するときは、既存のカスタムスカラフィールドの一覧により適したものがあるかを調べたほうがいいです。
フィールドを追加する際に新しいカスタムスカラーが適切だと思うなら、適切なコンセプトを捉えているかどうかをチームメンバーと話し合ったほうがいいです。

*ルール10: 意味的に価値を持つものを公開するときに、カスタムスカラー型を使用してください。*

### 再びページネーション

これで`Collection`型の主要なフィールドは全て見ました。次のオブジェクトは`CollectionRuleSet`です。これはかなり簡単です。
ここでの唯一の質問は、ルールのリストをページネーションするかどうかですが、リストを配列にするのは理にかなっています。
というんも、ルールのリストをページネーションするのはやり過ぎだからです。
コレクションは、限られた数のルールしかありません場合がほとんどですし、
コレクションが大きなルールセットを持つのはよいユースケースとは言えません。
おそらくルールが十数もあれば、コレクションを再考するか、商品を一つずつリストアップするほうが良いことを示唆しているのかもしれません

### 列挙型

スキーマの最後の型は、`CollectionRule`です。各ルールは、ルールの対象列（例: 商品タイトル）、リレーションのタイプ（例: 等価）、
使用する実際の値（例:「ブーツ」）で構成されます。値は`condition`という分かりにくい名前になっているので、名前は変更してもいいかもしれません。
カラムはデータベース特有の用語で、GraphQL上ではおそらく `field` のほうが良いでしょう

型に関してですが、`field`と` relation`ははおそらくどちらも内部的に列挙型として実装されています（使用している言語が列挙型をサポートしていると仮定します）。
幸いなことにGraphQLにも列挙型があるので、これら2つのフィールドを列挙型に変換できます。完成したスキーマは次のようになります。

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

*ルール11: フィールドが特定の集合の値のどれかを取る場合のみ、列挙型を使ってください*

## ステップ4: ビジネスロジック

コレクション用のGraphQL APIが最小限ですが、うまく設計できました。
ただ、今回設計したコレクションには対応できていない詳細ははたくさんあります。
詳細に実装するには、商品のソート順やパブリッシュなどの処理にもっと多くのフィールドが必要ですが、
原則としてそれらのフィールドはすべてこのチュートリアルで説明したデザインパターンに従ってください。
しかし、もう少し踏み込んで見ていきましょう。

このセクションでは、説明をわかりやすくするために、APIのあるクライアントを仮定したユースケースから初めましょう。
なので、一緒に仕事をしているクライアント開発者は、特定の製品がコレクションのメンバーであるかどうか、という非常に具体的なことを
知る必要があるとしましょう。
もちろんこれは、今のAPIで実現可能なことです。
コレクション内で完全な一連の製品が公開されているため、クライアントは気になる製品を探すだけで済みます。

この解決方法には2つの問題があります。
１つ目の問題ははっきりしていて、非効率的であることです。
コレクションには何百万もの製品が含まれている可能性があり、クライアントが製品のリストをフェッチしてループするとなると、処理が非常に遅くなります。 
2つ目の問題は1つ目よりも深刻な問題で、クライアント側でコードを書く必要があることです。
この最後のポイントは、設計理念で重要な部分です。サーバーは、常にビジネスロジックのただ一つの真実の源であるべきです。
APIは複数のクライアントにサービスを提供することがほとんどですが、それぞれのクライアントが同じロジックを実装する必要があるのであれば、コードの重複が発生し余分な作業とエラーが発生します。

*ルール12: APIはデータだけではなくビジネスロジックも提供すべきです。複雑な処理は複数のクライアントでは行わず、サーバー側で行うようにすべきです。*

クライアントのユースケースに話を戻しますが、ここでの最も良い方法はこの問題を解決することに特化した新しいフィールドを提供することです。
実際には次のようになります。

```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Bool!
}
```
このフィールドは、商品のIDを取得し、商品がコレクションに含まれているかどうかをサーバーが判断してブール値を返します。
このフィールドが、既存の`products`フィールドと被っているいうというのは関係ありません。
GraphQLは明示的にクライアントが要求するものだけを返すので、RESTとは異なり、余分なフィールドを追加することはありません。
クライアントは追加のフィールドに対してクエリを投げる以外にコードを書く必要はなく、消費される合計帯域幅はIDとブール値だけです。

ただし、補足しておきますが、ビジネスロジックを提供しているからといって、生データを提供する必要はがないわけではありません。
クライアントも必要であればビジネスロジックの実装をを自分で行うことができなければなりません。
クライアントが必要とするすべてのロジックを予測することはできません。
また、クライアントが追加のフィールドを簡単に要求できる方法は必ずしもありません。（ただ、できるだけこのような方法を提供できるようする必要はありますが）。

*ルール13: ビジネスロジックがあったとしても、生データも提供してください。*

最後に、API全体の形がビジネスロジックのフィールドに影響されることはないようにしてください。
ビジネスドメインのデータはコアとなるモデルにはあります。ビジネスロジックが現状の設計に合わないのであれば、
それはベースとなるモデルが間違っていることを示唆しているのかもしれません。

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
