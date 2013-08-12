クイックスタート
================

簡単な例
--------

プロパティ(性質)の定義の簡単な例は

.. code-block:: fsharp

  let revRevIsOrig (xs:list<int>) = List.rev(List.rev xs) = xs

この性質は、リストを逆順にして逆順にしたものは元のリスト自身になるということを主張します。この性質を確かめるために、F# インタラクティブ上にこの定義をロードして、起動してみましょう。

.. code-block:: fsharp

  > Check.Quick revRevIsOrig;;
  Ok, passed 100 tests.

性質が失敗すると、FsCheck は反例を表示します。例えば、もし

.. code-block:: fsharp

  let revIsOrig (xs:list<int>) = List.rev xs = xs

を定義すると、

.. code-block:: fsharp

  > Check.Quick revIsOrig;;
  Falsifiable, after 2 tests (2 shrinks) (StdGen (884019159,295727999)):
  [1; 0]

という結果を確認できるでしょう。

FsCheck は反例のシュリンク [#]_ も行うので、テストケースを失敗させるものが最小の反例となるのです。上の例では反例は確かに最小、つまりリストは少なくとも2つの異なる要素を持たなくてはならないのです。FsCheck はより小さな反例を(何らかの方法で)何回見つけてシュリンクを進めたかを表示します。

FsCheck を使う
--------------

FsCheck を使うために、最新の FsCheck のソースコードかバイナリをダウンロードします。スペックやテストデータ生成器を含むすべてのプロジェクトにあるアセンブリをビルドし参照します。性質を定義したモジュールを F# インタラクティブにロードし、

.. code-block:: fsharp

  Check.Quick <propertyName>

を呼び出すか、Check 関数を呼び出す小さなコンソールアプリを書いて実行することで性質をテストできます。xUnit や NUnit のようなテストランナーとの統合も同様に可能です。例は使い方のヒントを御覧ください。

性質をまとめる
--------------

大抵は、複数のテストすべき性質を書くことになります。FsCheck はクラスの静的メンバーとして性質をひとまとめにすることができます:

.. code-block:: fsharp

  type ListProperties =
      static member ``reverse of reverse is original`` xs = RevRevIsOrig xs
      static member ``reverse is original`` xs = RevIsOrig xs

これらは Check.QuickAll 関数を使うことで一度にチェックできます:

.. code-block:: fsharp

  > Check.QuickAll<ListProperties>();;
  --- Checking ListProperties ---
  ListProperties.reverse of reverse is original-Ok, passed 100 tests.
  ListProperties.reverse is original-Falsifiable, after 3 tests (3 shrinks) (StdGen (885249229,295727999)):
  [1; 0]

FsCheck はそれぞれのテスト名も出力します。モジュールにあるすべてのトップレベル関数はそのモジュール名をもつクラスの静的メンバーとしてコンパイルされるので、あるモジュールにあるすべてのトップレベル関数をテストするために Check.QuickAll を使うこともできます。しかし、モジュールの型は F# から直接アクセスできないので、次のようなトリックを使いましょう:

.. code-block:: fsharp

  > Check.QuickAll typeof<ListProperties>.DeclaringType;;
  --- Checking QuickStart ---
  QuickStart.revRevIsOrig-Ok, passed 100 tests.
  QuickStart.revIsOrig-Falsifiable, after 6 tests (7 shrinks) (StdGen (885549247,295727999)):
  [1; 0]
  QuickStart.revRevIsOrigFloat-Falsifiable, after 10 tests (4 shrinks) (StdGen (885679254,295727999)):
  [nan]

もしテストがループしたりエラーに出くわしたら何をする？
------------------------------------------------------

性質が有効ではありませんが、Check.Quick が反例を表示しないという場合があります。このような場合のために、別のテスト関数があります。テストを実行する前にそれぞれのテストケースを表示する

.. code-block:: fsharp

  Check.Verbose <property_name>

を使ってもう一度テストしてみましょう。つまり、最後に表示されたテストケースがループしてるかエラーが発生しているものだということです。Check.VerboseAll は性質のグループをくどくどとチェックするために型やモジュールにも使えます。

警告
----

上記の性質(逆順の逆順のリストは元のリスト自身)は常に正しいとは限りません。infinity や nan (非数(not a number))を含んだ浮動小数点数のリストを考えてみましょう。infinity <> infinity であり、nan <> nan なので、もし単純に要素同士の比較を用いるなら ``[nan, nan]`` の逆順の逆順は実際 ``[nan, nan]`` と等しくありません。FsCheck はこの手のスペック問題を見つけ出すコツを備えています。しかし、この振る舞いはめったに思ったとおりにならないので、型多相性(今のところ、unit, bool, char および string 値)を残しているなら FsCheck は「上手く」比較できる値だけを生成します。このエラーを実際に見るために、FsCheck に浮動小数点数のリストを生成させてみましょう:

.. code-block:: fsharp

  let revRevIsOrigFloat (xs:list<float>) = List.rev(List.rev xs) = xs

  > Check.Quick revRevIsOrigFloat;;
  Falsifiable, after 19 tests (12 shrinks) (StdGen (886719313,295727999)):
  [nan]

.. [#] 推定による絞り込みのこと。
