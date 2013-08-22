性質
====

性質は F# の関数定義として表現されます。性質はそのパラメータにわたって例外なく定量化されるので、

.. code-block:: fsharp

  let revRevIsOrig xs = List.rev(List.rev xs) = xs

は等式が全てのリスト xs で成り立つことを意味します。性質は決して単相型をもってはいけません。上記のような「多相的な」性質は、まるでジェネリックな引数が object 型であるかのようにテストされます。これは、様々な単純型(bool, char, string, ...)が生成されることを意味します。それは1つの生成されるリストに複数の型が含まれる場合さえあるかもしれないということで、例えば {['r', "1a", true]} は上記の性質をチェックするために使うことができるリストになりえます。ですが生成された値は型をベースにしているので、推論により、あるいは明示的に、xs に異なる型を与えることで、この振る舞いを簡単に変更することもできます。つまり、

.. code-block:: fsharp

  let revRevIsOrigInt (xs:list<int>) = List.rev(List.rev xs) = xs

は int のリストについてのみチェックされます。FsCheck は様々な形式の性質をチェックすることができます―これらの形式はテスト可能であると呼ばれ、'Testable というジェネリック型によって API において示されます。'Testable は bool または unit を返す任意個のパラメータを取る関数となるでしょう。後者の場合、もし(例外を)送出しなければテストはパスします。

条件付き性質
------------

性質は ==> という形式をとることがあります。例えば、

.. code-block:: fsharp

  let rec private ordered xs = 
     match xs with
     | [] -> true
     | [x] -> true
     | x::y::ys ->  (x <= y) && ordered (y::ys)

  let rec private insert x xs = 
     match xs with
     | [] -> [x]
     | c::cs -> if x <= c then x::xs else c::(insert x cs)

  let Insert (x:int) xs = ordered xs ==> ordered (insert x xs)

条件が成り立つときはいつでも ==> の後の性質が成り立つなら、このような性質は成り立ちます。テストは条件を満たさないテストケースを切り捨てます。条件を満たすケースが100件見つかるまで、あるいはテストケース数の限度に達する(条件が決して成り立たないループを避けるため)まで、テストケースの生成は続けられます。この場合、

.. code-block:: fsharp

  Arguments exhausted after 97 tests.

というようなメッセージは、条件を満たすテストケースが97件見つかり、その97件に対して性質が成り立った、ということを示します。この場合、生成された値は int 型に制限されなければならなかったことに注意してください。なぜなら生成された値は比較可能である必要がありましたが、これは型に反映されていません。ですから、明示的な制限がなければ、FsCheck は異なる型(object 型の派生型)を含むリストを生成したかもしれず、そしてこれらは互いに比較可能ではありません。

遅延性質
--------

F# はデフォルトでは正則評価ですが、上記の性質は必要以上に働いてしまいます。つまり、左側の条件の結果がどうであれ、条件の右側にある性質を評価します。上の例におけるパフォーマンス面の考察だけではありますが、性質の表現力を制限する可能性があります―考えてみましょう:

.. code-block:: fsharp

  let Eager a = a <> 0 ==> (1/a = 1/a)

  > Check.Quick Eager;;
  Falsifiable, after 1 test (0 shrinks) (StdGen (886889323,295727999)):
  0
  with exception:
  System.DivideByZeroException: Attempted to divide by zero.
     at Properties.Eager(Int32 a) in C:\Users\Kurt\Projects\FsCheck\FsCheck\FsCheck.Documentation\Properties.fs:line 24
     at DocumentationGen.fsCheckDocGen@130-3.Invoke(Int32 a) in C:\Users\Kurt\Projects\FsCheck\FsCheck\FsCheck.Documentation\Program.fs:line 130
     at FsCheck.Testable.evaluate[a,b](FSharpFunc`2 body, a a) in C:\Users\Kurt\Projects\FsCheck\FsCheck\FsCheck\Property.fs:line 168

ここで、性質が正しくチェックされているかどうかを確かめるために、遅延評価が必要になります:

.. code-block:: fsharp

  let Lazy a = a <> 0 ==> (lazy (1/a = 1/a))

  > Check.Quick Lazy;;
  Ok, passed 100 tests.

定量化された性質
----------------

性質は

.. code-block:: fsharp

  forAll <arbitrary>  (fun <args> -> <property>)

という形式を取る場合があります。例えば、

.. code-block:: fsharp

  let InsertWithArb x = forAll orderedList (fun xs -> ordered(insert x xs))

forAll の第一引数は IArbitrary インスタンスです。このようなインスタンスはテストデータのジェネレータとシュリンカ(詳しくは後ほど)をカプセル化します。その型のデフォルトであるジェネレータを使う代わりに、お手製のジェネレータを与えることで、テストデータの分布をコントロールすることができます。この例では、整序済みリスト用の自作ジェネレータを与えることで、整序されていないテストケースをフィルタリングするというよりも、むしろテストケースの総合的な限界に達することなく100のテストケースを生成できることを保証します。ジェネレータを定義するためのコンビネータを後ほど説明します。

例外の予測
----------

ある状況下で関数やメソッドが例外をスローするというテストをしたいと思うかもしれません。次のコンビネータが役に立ちます:

.. code-block:: fsharp

  throws<'e :> exn,'a> Lazy<'a>

例:

.. code-block:: fsharp

  let ExpectDivideByZero() = throws<DivideByZeroException,_> (lazy (raise <| DivideByZeroException()))

  > Check.Quick ExpectDivideByZero;;
  Ok, passed 100 tests.

時間制限のある性質
------------------

性質は

.. code-block:: fsharp

  within <timeout in ms> <Lazy<property>>

という形式をとることがあります。例えば、

.. code-block:: fsharp

  let timesOut (a:int) = 
      lazy
          if a>10 then
              while true do System.Threading.Thread.Sleep(1000)
              true
          else 
              true
      |> within 2000

  > Check.Quick timesOut;;
  Timeout of 2000 milliseconds exceeded, after 37 tests (0 shrinks) (StdGen (945192658,295727999)):
  11

第一引数は与えられた遅延性質が実行してよい最大の時間です。もしそれよりも長く実行していると、FsCheck はそのテストが失敗したと見做します。そうでなければ、遅延性質の結果は within の結果です。within が性質が実行されているスレッドをキャンセルしようとしても、それはきっと上手くいかないでしょうし、そのスレッドはプロセスが終了するまで実際に実行し続けるであろうということに注意してください。

テストケース分布の観測
----------------------

テストケースの分布を意識しておくのは重要なことです。つまり、もしテストデータがよく分布していなかったら、テスト結果から導き出される結論は正しくないかもしれません。特に、与えられた性質を満足するテストデータだけが使われるので、==> 演算子はテストデータの分布を不適切に歪めます。FsCheck はテストデータの分布を観測する手段を幾つか提供します。観測するためのコードは性質の宣言に含まれ、性質が実際にテストされる度に観測され、収集した観測結果はテストが完了した時に集約されます。

自明なケースの計数
------------------

性質は

.. code-block:: fsharp

  trivial <condition> <property>

という形式をとることがあります。例えば、

.. code-block:: fsharp

  let insertTrivial (x:int) xs = 
      ordered xs ==> (ordered (insert x xs))
      |> trivial (List.length xs = 0)

この条件が真になるテストケースは自明であると分類され、全体における自明なテストケースの比率が報告されます。この例では、テストすることで

.. code-block:: fsharp

  > Check.Quick insertTrivial;;
  Arguments exhausted after 55 tests (36% trivial).

という結果になります。

テストケースの分類
------------------

性質は classify という形式をとることがあります。例えば、

.. code-block:: fsharp

  let insertClassify (x:int) xs = 
     ordered xs ==> (ordered (insert x xs))
     |> classify (ordered (x::xs)) "at-head"
     |> classify (ordered (xs @ [x])) "at-tail" 

条件を満足するテストケースは与えられた分類に割り当てられ、分類の分布はテスト後に報告されます。この場合、結果は

.. code-block:: fsharp

  > Check.Quick insertClassify;;
  Arguments exhausted after 54 tests.
  44% at-tail, at-head.
  24% at-head.
  22% at-tail.

1つのテストケースは複数の分類に当てはまる場合があることに注意してください。

データの値の収集
----------------

性質は

.. code-block:: fsharp

  collect <expression> <property>

という形式をとることがあります。例えば、

.. code-block:: fsharp

  let insertCollect (x:int) xs = 
      ordered xs ==> (ordered (insert x xs))
      |> collect (List.length xs)

collect の引数はテストケース毎に評価され、値の分布が報告されます。この引数の型は sprintf "%A" を用いて出力されます。上記の例では、出力は

.. code-block:: fsharp

  > Check.Quick insertCollect;;
  Arguments exhausted after 70 tests.
  50% 0.
  32% 1.
  11% 2.
  4% 3.
  1% 4.

です。

観測の連結
----------

ここで説明した観測は何らかの方法で連結されるかもしれません。テストケースそれぞれにおける全ての観測は連結されており、これらの連結の分布は報告されます。例えば、

.. code-block:: fsharp

  let insertCombined (x:int) xs = 
      ordered xs ==> (ordered (insert x xs))
      |> classify (ordered (x::xs)) "at-head"
      |> classify (ordered (xs @ [x])) "at-tail"
      |> collect (List.length xs)

という性質をテストすると、

.. code-block:: fsharp

  > Check.Quick insertCombined;;
  Arguments exhausted after 53 tests.
  24% 0, at-tail, at-head.
  18% 1, at-tail, at-head.
  18% 1, at-head.
  16% 1, at-tail.
  7% 2, at-head.
  5% 2, at-tail.
  3% 2.
  1% 5, at-tail.
  1% 4.

となります。

And、Or および従属性質へのラベル付け
------------------------------------------

性質は

.. code-block:: fsharp

  <property> .&. <property>

  <property> .|. <property>

という形式をとることがあります。

p1 .&. p2 は両方成功した場合に成功し、性質のどちらか一方が失敗した場合に失敗し、両方ともに棄却された場合に棄却されます。p1 .|. p2 は性質のどちらかが成功した場合に成功し、両方の性質が失敗した場合に失敗し、両方とも棄却された場合に棄却されます。.&. コンビネータは、ジェネレータを共有する複雑な性質を記述するために一般に最も使われます。この場合、失敗時にどのサブプロパティ(従属性質)が原因で失敗したか正確に知るのが難しいことがあるかもしれません。そのために従属性質にラベルを付けることができて、FsCheck が反例を見つけると、失敗した従属性質のラベルを表示します。これはこのような形になります:

.. code-block:: fsharp

  <string> @| <property>

  <property> |@ <string>

例えば、

.. code-block:: fsharp

  let complex (m: int) (n: int) =
      let res = n + m
      (res >= m)    |@ "result > #1" .&.
      (res >= n)    |@ "result > #2" .&.
      (res < m + n) |@ "result not sum"

はこのようになります:

.. code-block:: fsharp

  > Check.Quick complex;;
  Falsifiable, after 1 test (0 shrinks) (StdGen (995775551,295727999)):
  Label of failing property: result not sum
  0
  0

1つの性質に複数のラベルを適用することは一向に構いません。FsCheck は適用可能なすべてのラベルを表示します。これは途中の結果を表示するのに便利で、例えば:

.. code-block:: fsharp

  let multiply (n: int, m: int) =
      let res = n*m
      sprintf "evidence = %i" res @| (
      "div1" @| (m <> 0 ==> lazy (res / m = n)),
      "div2" @| (n <> 0 ==> lazy (res / n = m)),
      "lt1"  @| (res > m),
      "lt2"  @| (res > n))

  > Check.Quick multiply;;
  Falsifiable, after 1 test (0 shrinks) (StdGen (996145572,295727999)):
  Labels of failing property: evidence = 0, lt1
  (0, 0)

上記の性質は従属性質をタプルにすることで連結していることに注意しましょう。これは長さ 6 のタプルまで上手くいきます。リストに対しても上手くいきます。一般的な形式

.. code-block:: fsharp

  (<property1>,<property2>,...,<property6>) means <property1> .&. <property2> .&.... .&.<property6>

  [property1;property2,...,propertyN] means <property1> .&. <property2> .&.... .&.<propertyN>

リストとして記述した例:

.. code-block:: fsharp

  let multiplyAsList (n: int, m: int) =
      let res = n*m
      sprintf "evidence = %i" res @| [
      "div1" @| (m <> 0 ==> lazy (res / m = n));
      "div2" @| (n <> 0 ==> lazy (res / n = m));
      "lt1"  @| (res > m);
      "lt2"  @| (res > n)]

同じ結果となります。
