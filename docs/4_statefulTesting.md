# 状態遷移を伴うテストの実行

FsCheckでは通常は内部状態が一連のメソッドによってカプセル化されているようなオブジェクトを対象にしたテストを実行することもできます。
非常に小さな拡張ではありますが、FsCheckではテスト対象となるクラスの仕様をモデルベースで定義することができます。
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

このクラスをテストするモデルとしては、オブジェクトの内部状態を1つのint値として抽象化したものが適切でしょう。
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

仕様はISpecificatin<'typeUnderTest,'modelType>を実装したオブジェクトとして定義します。
このインターフェイスを実装するには、初期状態のオブジェクトと、オブジェクトに対する初期状態のモデルを返すメソッドを実装します。
また、ICommandオブジェクトのジェネレーターを返すメソッドを実装する必要もあります。
それぞれのICommandオブジェクトでは、一般的にはテスト対象のオブジェクトに対する1つのメソッド呼び出しに対応するようにして、
コマンドを実行することでモデルとオブジェクトに対して起こることを定義します。
さらに、ICommandオブジェクトではコマンドを実行する前の時点で、一致すべき前提条件がチェックされます。
もしも前提条件が一致しないのであればFsCheckはそのコマンドを実行しません。
コマンドの実行後は一致すべき事後条件もチェックされます。
事後条件が一致しない場合、FsCheckではテストが失敗したものと判断されます。
なお反例が表示されるようにToStringをオーバーライドしておくことが推奨されます。
上記の仕様は以下のようにしてチェックできます：

```fsharp
> Check.Quick (asProperty spec);;
Falsifiable, after 6 tests (2 shrinks) (StdGen (1020916989,295727999)):
[inc; inc; inc; dec]
```

このように、「バグ」が発見されただけでなく、バグが発生する最低限の手順もFsCheckから出力されていることに注目してください。
