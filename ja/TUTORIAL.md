# チュートリアル: GraphQL APIのデザイン

本チュートリアルはもともと[Shopify](https://www.shopify.ca/)が社内利用のために作ったものですが、
GrahphQL APIを作る人すべてに便利だと思い今回公開することにしました。

3年以上にわたって、Shopifyでプロダクションのスキーマを
作成・拡張してきた中で得た教訓にもとに本チュートリアルを作成しました。
本チュートリアルは今後も更新されていく予定です。

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
コレクションは、ウェブサイトを閲覧する際の表示方法であったり、プログラム化されたルールに使うことができます[^1]
[^1]: ここはよくわからなかったです!すいません!
（たとえば、ある特定のコレクションの商品にのみ割引を適用したいことがあるかもしれません）。

バックエンドでは、次のように新しい機能が実装されています。

- すべてのコレクションは、タイトル、説明文 (HTMLフォーマットを含む)、画像のような幾つかのシンプルな属性を持っています。
- 2種類のコレクションがあります。コレクションに含まれる商品をリストアップする「マニュアル」コレクションと、ルールに従って商品をリストアップする「自動」コレクションです。
- 商品とコレクションの関係は多対多であるため、`CollectionMembership`というジョインテーブルを真ん中に持たせます。
- コレクションは、これまでの商品と同様に、公開されている（店頭に表示されている）か、公開されていないかのどちらかです。

このような背景がわかったので、API設計について考え始める準備が整いました。

## ステップ1: 俯瞰して見る

スキーマをナイーブに考えると、以下のようになるかもしれません(`Product` のような既存の型は省略)。

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
ナイーブなスキーマを簡略化すると次のようになります。

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

*ルール1: 特定のフィールドを扱う前に、常にオブジェクトとその関係のハイレベルなビューから始めてください。*

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

*ルール2: APIデザインに実装の詳細は決して含めないようにしてください*

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

*ルール3: 実装、ユーザーインターフェース、レガシーAPIではなく、ビジネスドメインを中心としたAPI設計をしてください。*

## ステップ3: 詳細を詰める

型をモデル化するクリーンな構造ができたので、ここまでは省略していたフィールドを戻して、
詳細なレベルで再びデザインを考えられるようになりました。

詳細を追加する前に、現時点で本当に必要かどうかを自問してください。
データベースのカラム、モデルのプロパティ、またはRESTの属性が存在するかもしれないというだけで、
必ずしもそれらをGraphQLのスキーマに追加する必要はありません。

スキーマ要素（フィールド、引数、型など）を公開するかどうかは実際のニーズとユースケースをもとに考えるべきです。
GraphQLのスキーマは要素を追加することで簡単に拡張することができますが、スキーマの変更や削除は重大な変更であり、はるかに困難です。

*ルール4: フィールドは削除するより追加するほうが簡単です*

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

*ルール5: 主なビジネスオブジェクトの型は常に`Node`を実装すべきです*

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

型に関してですが、`field`と` relation`はおそらくどちらも内部的に列挙型として実装されています（使用している言語が列挙型をサポートしていると仮定します）。
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
ただ、今回設計したコレクションには対応できていない詳細はたくさんあります。
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
クライアントも必要であればビジネスロジックの実装を自分で行うことができなければなりません。
クライアントが必要とするすべてのロジックを予測することはできません。
また、クライアントが追加のフィールドを簡単に要求できる方法は必ずしもありません。（ただ、できるだけこのような方法を提供できるようする必要はありますが）。

*ルール13: ビジネスロジックがあったとしても、生データも提供してください。*

最後に、API全体の形がビジネスロジックのフィールドに影響されることはないようにしてください。
ビジネスドメインのデータはコアとなるモデルにはあります。ビジネスロジックが現状の設計に合わないのであれば、
それはベースとなるモデルが間違っていることを示唆しているのかもしれません。

## ステップ5: ミューテーション

GraphQLのスキーマ設計で最後に残っているのは実際に値を変える機能、つまりは、コレクションや関連するオブジェクトの作成、更新、削除です。
わかりやすい部分から始めるために、今回のケースでは、特定の入出力は気にせず実装したいと思う色々ミューテーションをハイレベルな視点から実装し始めるべきです。
愚直に行くのであれば、CRUDのパラダイムに従って`create`、`delete`、`update`というミューテーションだけを持たせるかもしれません。
それなりに良いスタート地点ではあるのですが、適切なGraphQL APIというには不十分です。

### ロジカルなアクションの分離

単純にCRUDにこだわれば`update`ミューテーションがすぐに肥大化し,、
タイトルのような単純なスカラー値を更新するだけではなく、
コレクション内の商品の公開・非公開、追加・削除・順序替え、自動コレクションのルールの変更などのような
複雑なアクションを行う責務までをも持ってしまうことはすぐに気づくでしょう。
これはサーバー上での実装とクライアント側でのAPIの使い方を難しくしてしまいます。
代わりに、GraphQLをうまく利用して一つの大きなミューテーションをより細かい、ロジカルなアクションに分けることができます。
最初にできることとして、`update`に含まれた公開・非公開を切り出すことで、ミューテーションの一覧は次のようになります。

- create
- delete
- update
- publish
- unpublish

*ルール14: リソースにおいて、別のロジカルなアクションには別のミューテーションを書いてください*

### リレーションの操作

`update`ミューテーションはまだ多くの責務を持ちすぎているので、引き続き別のアクションに切り出したほうがいいです。
ただ、これから切り出すアクションは、別の視点から考えてみる価値があります。[^2]
[^2]: 結構異訳しました。
別の視点というのは、オブジェクトのリレーション(例: 一対多、多対多)を操作するという視点です。
これまですでに読み込み用のAPIで、「ID」対「 オブジェクト埋め込み」、「ページネーションの使用」対「配列」を考えてきましたが、
これらのリレーションをミューテーションする際にも読み込み用のAPIの時と似たような問題があります。

商品とコレクションの関係については、広く考慮できるスタイルが幾つかあります。
- 更新のミューテーションにリレーション全体 (例: `products: [ProductInput!]!`)を埋め込むのはCRUD形式のデフォルトですが、当然ながらリストが大きければすぐに効率は悪くなります。
- 更新のミューテーションに"差分"のフィールド (例: `productsToAdd: [ID!]!`)を埋め込むと、リスト全体ではなく変更があるIDのみを明記すればよいので、より効率的になります。ただ、アクションは密接に結びついたままです。
- 別のミューテーション(`addProduct`, `removeProduct`など)に完全に切り分けるの方法が、一番強力かつ柔軟であるだけではなく、一番うまく行きます。

最後の選択肢が一般的に最も安全です。結局このようなミューテーションがたいてい明確なロジカルアクションになることが大きな理由です。
ただ、考慮すべき要素がたくさんあります。
- リレーション先は大きくてページネーションされているか。もしそうなら、リスト全体を埋め込むのは実用的でないです。
  ただ、差分フィールドや別のミューテーションを設ければ大丈夫かもしれません。
  リレーション先が常に小さいのであれば (特に一対一であれば)、埋め込みが一番シンプルな選択かもしれません。
- リレーションはオーダリングされているか。もし商品とコレクションのリレーションがオーダリングされており、マニュアルのオーダリングも許可されています。
  オーダリングは埋め込まれたリストや別のミューテーション (`reorderProducts`ミューテーションを追加すれば良いです) を使えば自然に提供できますが、差分フィールドでは無理です。
- リレーションは必須か。商品とコレクションはどちらも作成・削除のライフサイクルをもっており、リレーションとは関係なく独立して存在することができます。
  もしリレーションが必須であれば (例: 商品はコレクションに必ず含まれなければならない)、アクションはリレーションを更新するだけではなく、
  実際に商品を*作成*するため別のミューテーションに分けるのを強く勧めます。
- どちらもIDを持っているか。コレクションとルールの関係は必須ですが (ルールはコレクションなしでは存在しえません)、
  ルールはIDを持っていません。ルールがコレクションの配下にあることは明らかです。
  このケースでは、リレーションが小さいのでリストを埋め込むのは悪い選択ではありません。
  他の方法を取るのであれば個々のルールが特定可能でなければならず、それはやり過ぎな感じがします。

*ルール15: リレーションのミューテーションは本当に複雑なのでキレイなルールにまとめるのは簡単ではないです*

これら全てを集めると、コレクションに関しては次のようなミューテーションのリストが出来上がります。

- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

商品に関しては別のミューテーションにしました。リレーションが大きくて、オーダリングされているからです。
それに対してルールはインラインのままです。リレーションは小さく、IDを持つほど大したものではないからです。

最後に1つ、`addProduct`ではなく`addProducts`のようなミューテーションになっていることに気づいた方もいるかもしれません。
これはこうした方が単純にクライアント側で便利だからです。
なぜなら、リレーションを操作する際に、一度に複数の商品を追加、削除、順序替えをするケースが多いからです。

*ルール16: リレーションに対して別のミューテーションを記述する際は、一度に複数の要素を操作できたほうが便利かを考慮してください。*

### 入力: 構造 パート1

記述したいミューテーションを洗い出せたので、入力の構造をどういった形にするかを考えてきましょう。
公開されている実際のプロダクションのスキーマのどれを見ても、多くのミューテーションが単一のグローバルな`Input`型を定義してミューテーションの全ての引数を持たせているのに気づくかもしれません。このパターンはレガシーなクライアントで見られる要件ですが、新しいコードにはこのパターンは必要ありませんので、無視して構いません。

単純なミューテーションの多くは、単一のIDか幾つかのIDだけが必要なものですので、このステップはかなり簡単です。
コレクションのうち、次のミューテーションの引数は簡単に洗い出すことができます。

- `delete`、`publish`、`unpublish` は全て単純に一つのコレクションIDが必要です。
- `addProducts`と`removeProducts`はどちらもコレクションIDと商品のID一覧が必要です。

これで設計すべき入力は次の3つの"複雑な"ものだけに絞られました。

- create
- update
- reorderProducts

createから見ていきましょう。かなり愚直に考えると入力はもとの愚直なコレクションモデルのように見えるかもれませんが、
そのモデルよりはすでにうまく設計できるようになっています。最終的なコレクションモデルと上述のリレーションに関する議論をベースにすると、
以下のようなものから始めることができます。

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
まずネーミングに関して一つ述べておくと、全てのミューテーションは、`<action>Collection`の形のほうが英語ではより自然なのに、あえて`collection<Action>`の形の名前になっていることに気づいた方もいるでしょう。残念ながら、GraphQLはミューテーションをグループ化したり整理する方法を提供していないため、回避策としてアルファベット順を使用します。コアな型を最初に置くことで、すべての関連するミューテーションが最終的なリストにまとめられます。

*ルール17: アルファベット順でまとめるために、ミューテーションの名前の先頭にミューテーションを施すオブジェクトの名前を付けてください (例:`cancelOrder`ではなく`orderCancel`)。*

### 入力: スカラー

これまでで得られたスキーマのドラフトは完全にナイーブなアプローチよりもかなり良くなっていますが、
まだ完璧ではありません。特に、入力フィールド`descripton`には幾つかの問題があります。
`HTML`がnull不可能というのは、コレクションの説明文が出力値である場合には理にかなっていますが、
入力の場合には次の理由から、あまりうまくいきません。まず、 `!`は出力ではnull不可能性を意味しているのですが、
入力では同じことは意味しません。どちらかというと、フィールドが「必須」であるかどうかの概念を示していると言って良いでしょう。
必須フィールドはリクエストが先に進むためにクライアントが提供しなければならないものですが、`description`は必須ではありません。
説明文を提供しないという理由だけでクライアントがコレクションを作れないという事態はあってほしくありません (もしくは、クライアントに意味のない`""`の提供を余儀なくさせるようなことはしたくありません)。なので、`description`を必須でなくすべきです。

*ルール18: 入力フィールドは、ミューテーションの処理に意味的に必要な場合のみ、必須フィールドにしてください。*

`description`のもう一つの問題は型にあります。すでに強く型付けされていますし、(`String`ではなく`HTML`)、このチュートリアルではここまでずっと強い型付けでやってきたので、これは直感的でないように見えるかもしれません。
しかし、もう一度いいますが、入力は出力とは挙動が少し違います。
入力における強い型付けのバリデーションは「ユーザースペース」のコードが実行される前にGraphQLのレイヤーで行われます。
つまり、現実的にはクライアントは２つのレイヤーでのエラーを扱わなければならないということです。
1つは、GraphQLレイヤーのバリデーションエラー。もう一つは、ビジネスレイヤーのバリデーションエラー(例えば、現在のストレージで作成できるコレクションの上限に達しました、みたいな)です。このプロセスを簡略化するために、クライアントが前のレイヤーでバリデーションするのが難しい場合は、
入力フィールドを意図的に弱く型付けします。これによって、ビジネスロジック側で全てのバリデーションができ、
クライアントは一箇所からのエラーのみを扱えばよいようになります。

*ルール19: フォーマットが曖昧でクライアント側のバリデーションが複雑な場合、入力にはより弱い型を使用します（例えば `Email`ではなく` String`）。これにより、サーバーはすべての複雑なバリデーションを一度に実行し、エラーを単一の形式で一箇所に返すことができ、クライアントを簡素化します。*

注意したいのは、全ての入力を弱く型付けしたほうが良いわけではないということです。
ルールの入力値である`field`や`relation`には強い型である列挙型を使いますし、
仮に本チュートリアルで他の入力に`DateTime`のようなものあれば強い型を使うでしょう。

The key
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

## おわりに

チュートリアルを読んでいただきありがとうございました! 現時点でGraphQL APIの設計方法に対して十分に理解できているのであれば嬉しいです。
満足のできるAPIが設計できれば、あとは実装するだけです!
