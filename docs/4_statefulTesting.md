# 状態遷移を伴うテストの実行

たいていの場合オブジェクトは一連のメソッドによって内部状態がカプセル化されているわけですが、FsCheckではこのようなオブジェクトを対象にしたテストを実行することもできます。
FsCheckではほんの少し手を加えるだけで、テスト対象となるクラスに対してモデルを基準とするような仕様を定義できます。
たとえば以下のような人為的なバグが含まれているクラスがあるとします：

```fsharp
type Counter() =
    let mutable n = 0
    member x.Inc() = n <- n + 1
    member x.Dec() = if n > 2 then n <- n - 2 else n <- n - 1
    member x.Get = n
    member x.Reset() = n <- 0
    override x.ToString() = n.ToString()
```

このクラスをテストするためのモデルとしては、オブジェクトの内部状態を表すのに役立ちそうなint値1つがふさわしいでしょう。
それを踏まえると、以下のような仕様が作成できます：

```fsharp
let spec =
    let inc =
        { new ICommand<Counter,int>() with
            member x.RunActual c = c.Inc(); c
            member x.RunModel m = m + 1
            member x.Post (c,m) = m = c.Get
            override x.ToString() = "inc" }
    let dec =
        { new ICommand<Counter,int>() with
            member x.RunActual c = c.Dec(); c
            member x.RunModel m = m - 1
            member x.Post (c,m) = m = c.Get
            override x.ToString() = "dec" }
    { new ISpecification<Counter,int> with
        member x.Initial() = (new Counter(),0)
        member x.GenCommand _ = Gen.elements [inc;dec] }
```

仕様はISpecification<'typeUnderTest,'modelType>を実装したオブジェクトです。
仕様は、初期状態のオブジェクトと、オブジェクトに対する初期状態のモデルを返さなくてはいけません。
また、ICommandオブジェクトのジェネレーターも返す必要があります。
それぞれのICommandオブジェクトでは、一般的にはテスト対象のオブジェクトに対する1つのメソッド呼び出しに対応するようにして、
コマンドを実行することでモデルとオブジェクトに対して起こることを定義します。
さらに、ICommandオブジェクトではコマンドを実行する前の時点で、一致すべき前提条件がチェックされます。
もしも前提条件が一致しないのであればFsCheckはそのコマンドを実行しません。
コマンドの実行後は一致すべき事後条件もチェックされます。
事後条件が一致しない場合、FsCheckではテストが失敗したものと判断されます。
なお反例が表示できるようにToStringをオーバーライドしておくとよいでしょう。
上記の仕様は以下のようにしてチェックできます：

```fsharp
> Check.Quick (asProperty spec);;
Falsifiable, after 6 tests (2 shrinks) (StdGen (1020916989,295727999)):
[inc; inc; inc; dec]
```

このように、「バグ」が発見されただけでなく、バグが発生する最低限の手順もFsCheckから出力されているのがポイントです。
