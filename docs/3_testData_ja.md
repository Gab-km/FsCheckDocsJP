テストデータ: ジェネレータ、シュリンカ、 Arbitrary インスタンス
===============================================================

テストデータはテストデータジェネレータによって作成されます。 FsCheck には、いくつかのよく使用される型については既定のジェネレータが定義されていますが、自分で定義することもできますし、導入した新しい型については自身でジェネレータを定義する必要があります。ジェネレータは Gen<'a> という形式の型を持ちます。これは型 a の値のためのジェネレータです。型 Gen の値を扱うために、 FsCheck は gen と呼ばれるコンピュテーション式を提供します。これにより Gen モジュール内のすべての関数が自由に利用できます。シュリンカは 'a -> seq<'a> という型を持ちます。値を一つ受け取ると、ある方法により受け取った値より小さな値からなるシーケンスを生成します。 FsCheck は、与えられた性質を満たさない値セットを見つけると、値のシュリンクを取得して、それぞれを順番に試して性質が満たされないことを確認することにより、元の（ランダムな）値より小さな値を作ろうとします。もし存在すれば、見つかった小さな値が新しい反例となり、その値とともにシュリンク過程が継続します。シュリンカに FsCheck からの特別なサポートはありません。言い換えればこれは必要ないということです。なぜならば、必要なものはすべて seq コンピュテーション式と Seq モジュールにあるからです。最後に、 Arbitrary<'a> インスタンスは、これまでの利用される 2 つの型をプロパティとしてまとめます。 FsCheck では Arbitrary インスタンスを、 Type から Arbitrary インスタンスの辞書に登録することもできます。この辞書は、主張のタイプに基づき、主張を持つ性質の任意のインスタンスを見つけるために使われます。 Arb モジュールには Arbitrary インスタンスのためのヘルパ関数があります。

ジェネレータ
------------

ジェネレータは、関数

```fsharp
val choose : (int * int -> Gen<int>)
```

により構築されます。これは一様分布を利用した区間から値のランダム抽出を行います。例えばリストからの要素のランダム抽出を行いたい場合、

```fsharp
let chooseFromList xs = 
    gen { let! i = Gen.choose (0, List.length xs-1) 
          return (List.nth xs i) }
```

を使用します。

選択肢から選ぶ
--------------

ジェネレータは、リスト内のジェネレータから等確率で選ぶ Gen.oneof のような形式をとるかもしれません。例えば、

```fsharp
Gen.oneof [ gen { return true }; gen { return false } ]
```

は確率 1/2 で true になるランダムな bool 値を生成します。あるいは、関数

```fsharp
val frequency: seq<int * Gen<'a>> -> Gen<'a>
```

を使用して、結果の分布を制御することができます。 frequency はリストからジェネレータをランダムに選びますが、それぞれの選択肢が選ばれる確率は与えられた要素により重み付けされます。例えば、

```fsharp
Gen.frequency [ (2, gen { return true }); (1, gen { return false })]
```

は 2/3 の確率で true を生成します。

テストデータサイズ
------------------

テストデータジェネレータは暗黙のサイズパラメータを持ちます。 FsCheck は少しのテストケースを作ることから始め、テストが進むにつれて徐々にサイズを増加させます。テストデータジェネレータによってサイズパラメータの解釈の方法が異なります。すなわち、無視するものもあれば、例えばリストジェネレータは生成されるリストの長さの上限にします。使用するかどうかは、テストデータジェネレータを制御したいかどうかによります。サイズパラメータの値は

```fsharp
val sized : ((int -> Gen<'a>) -> Gen<'a>)
```

を使用することで取得できます。 sized g は、現在のサイズをパラメータとして g を呼びます。例えば、 0 から大きさまでの自然数を生成するには、

```fsharp
Gen.sized <| fun s -> Gen.choose (0,s)
```

を使用します。

サイズ制御の目的は、エラーを見つけるのに充分多く、しかもテストを素早く行うのに充分少ないだけのテストケースを確保するためにあります。既定のサイズ制御ではこれを達成できない場合があります。例えば、 1 回のテストランの終わりまでに任意のリストは 50 要素まで作られますが、これはつまりリストのリストは 2,500 要素となり、効率的なテストには多すぎます。このようなケースでは、明示的にサイズパラメータを変更するのが良いでしょう。そのようにするには

```fsharp
val resize : (int -> Gen<'a> -> Gen<'a>)
```

を使用します。 resize n g はサイズパラメータ n とともに g を実行します。サイズパラメータは負になってはいけません。例えば、ランダム行列を生成するために元のサイズの平方根をとるのは適切でしょう。

```fsharp
let matrix gen = Gen.sized <| fun s -> Gen.resize (s|>float|>sqrt|>int) gen
```

再帰的データ型の生成
--------------------

再帰的データ型のためのジェネレータは、 `oneof` あるいは `frequency` をコンストラクタの選択に使用し、 F# の標準のコンピュテーション式の文法でジェネレータを組み立てることで、容易に表現することができます。また、 6 個までのアリティをコンストラクタや関数を Gen 型にマップする関数もあります。例えば、木の型が

```fsharp
type Tree = Leaf of int | Branch of Tree * Tree
```

で定義されているとすると、木のためのジェネレータは

```fsharp
let rec unsafeTree() = 
    Gen.oneof [ Gen.map Leaf Arb.generate<int> 
                Gen.map2 (fun x y -> Branch (x,y)) (unsafeTree()) (unsafeTree())]
```

で定義できるでしょう。しかしながら、このような再帰ジェネレータはおそらく StackOverflowException とともに失敗するか、とても大きな結果を生じるでしょう。これを防ぐために、再帰ジェネレータは常にサイズ制御機序を使用するべきです。例えば、

```fsharp
let tree =
    let rec tree' s = 
        match s with
        | 0 -> Gen.map Leaf Arb.generate<int>
        | n when n>0 -> 
            let subtree = tree' (n/2)
            Gen.oneof [ Gen.map Leaf Arb.generate<int> 
                        Gen.map2 (fun x y -> Branch (x,y)) subtree subtree]
        | _ -> invalidArg "s" "Only positive arguments are allowed"
    Gen.sized tree'
```

以下に注意してください。 

* サイズが 0 であるときに結果を葉で強制することで終了を保証しています。
* 再帰ごとにサイズを半分にします。これによりサイズは木におけるノード数の上限をとなります。望む限りにおいて、サイズについて考える必要はありません。
* 2 つの枝間で 1 つの部分木を共有しているという事実は、ケースごとに同じ木を生成するということを意味しません。

便利なジェネレータコンビネータ
------------------------------

g を型 t に対するジェネレータとすると、

* `two g` は t の 2 つ組を生成します。
* `three g` は t の 3 つ組を生成します。
* `four g` は t の 4 つ組を生成します。
* xs をリストとすると、 `elements xs` は xs の任意の要素を生成します。
* `listOfLength n g` はちょうど n 個の t のリストを生成します。
* `listOf g` は長さがサイズパラメータによって決められる t のリストを生成します。
* `nonEmptyListOf g` は長さがサイズパラメータによって決められる t の空でないリストを生成します。
* `constant v` は値 v を生成します。
* `suchThat p g` は述語 p を満足する t を生成します。述語を満たす確率が高いようにしてください。
* `suchThatOption p g` は述語 p を満足するならば Some t を、見つからなければ None を生成します（「一生懸命」やった後で）。

これらすべてのジェネレータコンビネータは Gen モジュールの関数です。

型に基づく既定のジェネレータとシュリンカ
----------------------------------------

FsCheck には、よく使用される型（unit、 bool、 byte、 int、 float、 char、 string、 DateTime、リスト、一次元か二次元の配列、 Set、 Map、オブジェクト、以上の型から型への関数）については既定のテストデータジェネレータとシュリンカが定義されています。さらに、リフレクションを用いて、 FsCheck はあらゆるプリミティブ型に関して定義された（FsCheck 内に定義されているかユーザー定義かに関わらず）レコード型、判別共用体、タプル、列挙型の既定の実装を得ることができます。これらについては性質ごとに明示的に定義する必要はありません。 FsCheck は、すべての性質の主張について、知りうる、得られる限りにおいて、適切なジェネレータとシュリンカを指定したプロパティを提供することができます。たいていの場合、性質に基づいたこれらの型を見つけ出す仕事を型推論に任せることができます。しかしながら FsCheck に特定のジェネレータとシュリンカを強制したい場合は、適切な型注釈を与えることによって可能になります。導入で述べたように、 FsCheck は特定の型に対するジェネレータとシュリンカを Arbitrary 型としてまとめることができます。 Arbitrary<'a> を継承したクラスのインスタンスを返すようなクラスの静的メンバを定義することで、独自型に対する Arbitrary インスタンスを FsCheck に与えることができます。

```fsharp
type MyGenerators =
    static member Tree() =
        {new Arbitrary<Tree>() with
            override x.Generator = tree
            override x.Shrinker t = Seq.empty }
```

'a を Arbitrary インスタンスを定義したい特定の型に置き換えてください。 Generator メソッドのみ定義する必要があります。 Shrinker は既定では空のシーケンスを返します（すなわち、この型に対してはシュリンクが起こらない）。そして、このクラスのすべての Arbitrary インスタンスを登録するために、次のようにします。

```fsharp
Arb.register<MyGenerators>()
```

これで FsCheck は Tree 型のことを知るようになりました。そして、 Tree 値のみならず、例えば Tree を含むリストやタプル、オプション値についても生成できます。

```fsharp
let RevRevTree (xs:list<Tree>) = 
List.rev(List.rev xs) = xs

> Check.Quick RevRevTree;;
Ok, passed 100 tests.
```

ジェネリック型パラメータを生成するために、例えば、

```fsharp
type Box<'a> = Whitebox of 'a | Blackbox of 'a
```

について、同様の原理があてはまります。したがって MyGenerators 型は次のように記述できます。

```fsharp
let boxGen<'a> : Gen<Box<'a>> = 
    gen { let! a = Arb.generate<'a>
          return! Gen.elements [ Whitebox a; Blackbox a] }

type MyGenerators =
    static member Tree() =
        {new Arbitrary<Tree>() with
            override x.Generator = tree
            override x.Shrinker t = Seq.empty }
    static member Box() = Arb.fromGen boxGen
```

Arb モジュールの関数 'val generate<'a> : Gen<'a>' が Box の型パラメータのジェネレータを取得するために使用していることに注目してください。これによりジェネレータを再帰的に定義することができます。同じように、 shrink<'a> 関数があります。このような Arbitrary インスタンスの記述法の感覚をつかむために、 FsCheck のソースコードの既定の Arbitrary の実装の例を参照してください。 Arb モジュールも同様に役立つでしょう。さあ、次の性質を確認してみましょう。

```fsharp
let RevRevBox (xs:list<Box<int>>) = 
    List.rev(List.rev xs) = xs
    |> Prop.collect xs

> Check.Quick RevRevBox;;
Ok, passed 100 tests.
11% [].
2% [Blackbox 0].
1% [Whitebox 9; Blackbox 3; Whitebox -2; Blackbox -8; Whitebox -13; Blackbox -19;
(etc)
```

クラスに属性によるタグ付けが必要ないことに注意してください。 FsCheck はジェネレータの型を静的メンバの戻す型により決定します。また、このケースではジェネレータやシュリンカの記述が必要ないことに注意してください。 FsCheck は、リフレクションにより判別共用体、レコード型、列挙型に対する適切なジェネレータを作ることができます。

いくつかの便利な Arb モジュールのメソッド
-----------------------------------------

* `Arb.from<'a>` は与えられた型 'a に対する登録された Arbitrary インスタンスを返します。
* `Arb.fromGen` は与えられたジェネレータから新しい Arbitrary インスタンスを作成します。
  シュリンカは空のシーケンスを返します。
* `Arb.fromGenShrink` は与えられたジェネレータとシュリンカから新しい Arbitrary インスタンスを作成します。
  これは Arbitrary を自身で実装するのと等価ですが、短くなるでしょう。
* `Arb.generate<'a>` は与えられた型 'a に対する登録された Arbitrary インスタンスのジェネレータを返します。
* `Arb.shrink` は与えられた値に対して登録されたインスタンスの即時シュリンクを返します。
* `Arb.convert` は、変換関数 to ('a ->'b) と from ('b ->'a) が与えられると、 Arbitrary<'a> インスタンスを Arbitrary<'b> に変換します。
* `Arb.filter` は与えられた Arbitrary インスタンスのジェネレータとシュリンカを、与えられたフィルター関数にマッチする値のみを含むように、フィルターします。
* `Arb.mapFilter` は与えられた Arbitrary インスタンスのジェネレータをマップし、シュリンカをフィルターします。
  ジェネレータのマッピングは高速になります。例えば PositiveInt に対して負の値をフィルターするより絶対値をとった方が高速です。
* `Arb.Default` は、 FsCheck によりあらかじめ公開登録されたすべての既定の Arbitrary インスタンスを含む型です。
  これは既定のジェネレータをオーバーライドするのに便利です。
  典型的には、ある値をフィルターして、なおかつオーバーライドしたジェネレータから既定のジェネレータが参照できるようにしたい場合です。
