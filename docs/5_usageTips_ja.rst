使い方のヒント
==============

関数の性質
----------

FsCheck はランダムな関数値を生成できるので、関数の性質を検査できます。例えば、次のように関数合成の結合性を検査できます。

.. code-block:: fsharp

  let associativity (x:Tree) (f:Tree->float,g:float->char,h:char->int) = ((f >> g) >> h) x = (f >> (g >> h)) x

  > Check.Quick associativity;;
  Ok, passed 100 tests.

Tree -> *任意の型* の関数を生成できます。反例が見つかった場合は、関数値が "func" として表示されます。しかしながら、 FsCheck は Function 型を使用することで、より詳細に生成された関数を表示することができます。もしそれを使えば、 FsCheck は関数をシュリンクことさえ可能です。例は以下の通りです。

.. code-block:: fsharp

  let mapRec (F (_,f)) (l:list<int>) =
      not l.IsEmpty ==>
          lazy (List.map f l = ((*f <|*) List.head l) :: List.map f (List.tail l))

  > Check.Quick mapRec;;
  Falsifiable, after 1 test (1 shrink) (StdGen (1028557426,295727999)):
  { 0->0; 1->0; 2->0 }
  [1]

``type Function<'a,'b> = F of ref<list<('a*'b)>> * ('a ->'b)`` は呼ばれていたすべての引数、および生成した結果の写像を記録します。性質では、例のように、パターンマッチによって実際の関数を抽出することができます。 Function は関数を出力し、また、関数をシュリンクするために使用されます。

カスタムジェネレータを使用する forAll の代わりにパターンマッチを使用する
----------------------------------------------------------------------------

.. code-block:: fsharp

  type EvenInt = EvenInt of int with
      static member op_Explicit(EvenInt i) = i

  type ArbitraryModifiers =
      static member EvenInt() = 
          Arb.from<int> 
          |> Arb.filter (fun i -> i % 2 = 0) 
          |> Arb.convert EvenInt int

  Arb.register<ArbitraryModifiers>()

  let ``generated even ints should be even`` (EvenInt i) = i % 2 = 0
  Check.Quick ``generated even ints should be even``

特にカスタムシュリンク関数と同様に定義できるので、これは性質をはるかに読みやすくします。 FsCheck は、例えば NonNegativeInt 、 PositiveInt 、 StringWithoutNullChars など、 FsCheck はこれらのうちかなりの数を持っています。 Arbitrary のデフォルトインスタンスである Arb.Default 型を見てみましょう。そしてそのうえ、あなた自身で定義したい場合は、 Arb.filter 、 Arb.convert 、Arb.mapFilter が役立つでしょう。

モジュールのみを使ったテスト実行
--------------------------------

Arbitrary インスタンスはクラスの静的メンバとして与えられ、性質はクラスの静的メンバとして一緒にグループ化することができるので、また、トップレベルの let 関数はそれらを囲むモジュール(クラスとしてコンパイルされる)の静的メンバとしてコンパイルされるので、単に性質やジェネレータをトップレベルの let-束縛関数として定義でき、次のトリックを使用してすべてのジェネレータと性質を登録することができます。

.. code-block:: fsharp

      let myprop =....
          let mygen =...
          let helper = "a string"
          let private helper' = true

      type Marker = class end
          Arb.register (typeof<Marker>.DeclaringType)
          Check.All (typeof<Marker>.DeclaringType)

Marker 型は、モジュールの Type が取得できるように、モジュール内に定義されているだけの任意の型です。 F# は直接モジュールの型を取得する方法を提供していません。 FsCheck は、戻り値の型に基づいて関数の意図を決定しています。すなわち、

- 性質： unit、bool、Property、これらの型への任意の引数の関数、もしくはこれらの型のいずれかの Lazy 値を返します
- Arbitrary インスタンス： Arbitrary<_> を返します

他の全ての関数は丁重に無視されます。もし FsCheck が何かを行うという型を返す関数をトップレベルに持っていて、しかしそれらをチェックしたり登録したくない場合は、単にそれらを private にするだけです。 FsCheck はこれらの関数を無視します。

あなたのテストのために働く NUnit Addin を取得するための一時修正
---------------------------------------------------------------

テスト(FsCheck.NUnit の [Property] でマークされたメソッド)を含むプロジェクトでは、以下のことを行ってください。

- FsCheck.Nunit と FsCheck.NUnit.Addin を参照に追加
- アドインを実装するプロジェクトに public クラスを追加

ここに、 F# の例を示します。

.. code-block:: fsharp

  open NUnit.Core.Extensibility
  open FsCheck.NUnit
  open FsCheck.NUnit.Addin
  [<NUnitAddin(Description = "FsCheck addin")>]
  type FsCheckAddin() =        
      interface IAddin with
          override x.Install host = 
              let tcBuilder = new FsCheckTestCaseBuider()
              host.GetExtensionPoint("TestCaseBuilders").Install(tcBuilder)
              true

これは C# プロジェクトでも使用できます。

.. code-block:: csharp

  using FsCheck.NUnit.Addin;
  using NUnit.Core.Extensibility;
  namespace FsCheck.NUnit.CSharpExamples
  {
      [NUnitAddin(Description = "FsCheck addin")]
      public class FsCheckNunitAddin : IAddin
      {
          public bool Install(IExtensionHost host)
          {
              var tcBuilder = new FsCheckTestCaseBuider();
              host.GetExtensionPoint("TestCaseBuilders").Install(tcBuilder);
              return true;
          }
      }
  }

その後は、このように [Test] の代わりに [Property] を使用してテストにフラグを宣言することができます。

.. code-block:: fsharp

  [<Property>]
  let maxLe (x:float) y = 
      (x <= y) ==> (lazy (max  x y = y))

アドインは、 Check.One を使ってこれを実行し、 テストに失敗した場合に、そのフラグを立てるために Assert.Fail を実行します。

FsCheck と mb|x|N|cs|Unit を統合するための IRunner 実装
--------------------------------------------------------

Check.One もしくは Check.All メソッドに渡すことができる Config 型は、引数として IRunner をとります。このインターフェイスは次のメソッドを持っています。

- OnStartfixture は、FsCheck がその型のすべてのメソッドをテストするときに、任意のテストを開始する前に呼び出されます。
- OnArguments は、テスト番号、引数、あらゆる関数の実装を渡して、全てのテストの後に呼び出されます。
- OnShrink は全ての成功したシュリンクに対して呼び出されます。
- OnFinished は、テストの名前と、全体的なテスト実行の結果を伴って呼び出されます。これは以下の例のように、外側のユニットテストフレームワークから Assert 文を呼び出すために使われます - FsCheck は複数のユニットテストフレームワークを統合することができます。あなたは、 setup や tear down、素敵なグラフィカルランナーなどを持つ他のユニットテスティングフレームワークの能力に影響力を行使することができます。

.. code-block:: fsharp

  let xUnitRunner =
      { new IRunner with
          member x.OnStartFixture t = ()
          member x.OnArguments (ntest,args, every) = ()
          member x.OnShrink(args, everyShrink) = ()
          member x.OnFinished(name,testResult) = 
              match testResult with 
              | TestResult.True _ -> Assert.True(true)
              | _ -> Assert.True(false, Runner.onFinishedToString name result) 
      }

  let withxUnitConfig = { Config.Default with Runner = xUnitRunner }

生成された引数の出力をカスタマイズするための IRunner 実装
---------------------------------------------------------

デフォルトでは、 FsCheck は sprintf "%A"、すなわち構造化フォーマットを使用して生成された引数を出力します。これは通常、あなたの期待通りのことを行います。すなわち、プリミティブ型は値を出力し、オブジェクトは ToString オーバーライドを出力するなどです。もしそうしなければ（動機のあるケースは、 COM オブジェクトをテストすることです - オーバーライドされた ToString という選択肢はなく、構造化フォーマットはそれに役立つようなことは何もしません）、これを性質毎に解決するためにラベルコンビネータを使用することができますが、より構造化した解決策は IRunner を実装することで達成することができます。例は以下の通りです。

.. code-block:: fsharp

  let formatterRunner =
      { new IRunner with
          member x.OnStartFixture t =
              printf "%s" (Runner.onStartFixtureToString t)
          member x.OnArguments (ntest,args, every) =
              printf "%s" (every ntest (args |> List.map formatter))
          member x.OnShrink(args, everyShrink) =
              printf "%s" (everyShrink (args |> List.map formatter))
          member x.OnFinished(name,testResult) = 
              let testResult' = match testResult with 
                                  | TestResult.False (testData,origArgs,shrunkArgs,outCome,seed) -> 
                                      TestResult.False (testData,origArgs |> List.map formatter, 
                                                        shrunkArgs |> List.map formatter,outCome,seed)
                                  | t -> t
              printf "%s" (Runner.onFinishedToString name testResult') 
      }

等式の左辺と右辺を出力する等価比較
----------------------------------

性質は、一般に等価性をチェックします。テストケースが失敗した場合、FsCheck は反例を出力しますが、時に、最初に生成された引数でいくつかの複雑な計算を行う場合は特に、比較の左辺と右辺も出力すると便利です。これを簡単にするために、独自のラベル表示等価コンビネータを定義できます。

.. code-block:: fsharp

  let (.=.) left right = left = right |@ sprintf "%A = %A" left right

  let testCompare (i:int) (j:int) = 2*i+1  .=. 2*j-1

  > Check.Quick testCompare;;
  Falsifiable, after 1 test (0 shrinks) (StdGen (1029127459,295727999)):
  Label of failing property: 1 = -1
  0
  0

もちろん、あなたがよく使用する任意の演算子や関数のためにこれを行うことができます。

FsCheck を使用するためのいくつかの方法
--------------------------------------

- あなたのプロジェクトの fsx ファイルに性質やジェネレータを追加することで。実行することは簡単で、 ctrl-a を入力してから alt-enter を入力すると、結果は F# Interactive に表示されます。ソリューションに組み込まれている dll を参照する時は注意してください。 F# interactive はセッションの残りの間はずっとそれらをロックし、あなたがセッションを終了するまでビルドすることができません。一つの解決策は、 dll の代わりにソースファイルを(ソリューションに)含めることですが、それは処理が遅くなります。小規模なプロジェクトに有用です。私が知る限りでは、デバッグは困難です。
- 別のコンソールアプリケーションを作成することで。アセンブリに迷惑なロックを行わないので、デバッグは簡単です。 テストのために FsCheck のみをを使用し、性質が複数のアセンブリをまたぐような場合は、最良の選択肢です。
- 別のユニットテストフレームワークを使用することで。 FsCheck / ユニットテストの手法が混在し(いくつかのものはユニットテストを使用して調べた方が簡単であり、逆もまた然り)、グラフィカルランナーを好む場合に便利です。あなたが使用しているユニットテストフレームワーク次第では、無料で Visual Studio と上手く統合できるでしょう。このシナリオで FsCheck をカスタマイズする方法は、上記を参照してください。

