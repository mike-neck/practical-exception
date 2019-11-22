Java における例外
===

1.例外はオブジェクト
---

* Java における例外は実行中のメソッドを失敗させ、呼び出し元に通知する特別なクラスのインスタンスオブジェクト
* すべての例外は `Exception` クラスの直接または間接のサブクラスのインスタンス
* オブジェクトにする以外にも以下の方法があるが、オブジェクトが拡張性とシンプルさでよさそう
  * `int` 値を用いる方法
    * メリット
      * シンプル
      * C 言語などの API と同様な扱い方ができる
    * デメリット
      * 関数・ライブラリーごとにエラー値の体系が異なりミスを多発しやすい
      * 詳細なエラーの情報を添付できない
  * `String` を用いる
    * メリット
      * 例外の表現として柔軟・シンプル
      * 詳細情報が付与できる
    * デメリット
      * 例外ごとに情報を取り出すためのパーサーが必要になる

2.独自例外を作る
---

* 独自の例外を作ると、きめ細かい例外処理が行える

### 独自例外の作り方

* `Exception` クラスあるいは `RuntimeException` クラスを継承したクラスを作る

```java
class NoBeerException extends Exception {
  NoBeerException(String message) {
    super(message);
  }
}
```

* 独自例外クラスを用いると `Exception` クラスのインスタンスだけでは実現できない、豊富な情報を伝えられる

```java
class ItemNotFoundException extends Exception {

  final int itemId;
  final int categoryId;

  ItemNotFoundException(int itemId, int categoryId) {
    super(String.format(
      "item not found, id: %d, category: %d",
          itemId, categoryId));
    this.itemId = itemId;
    this.categoryId = categoryId;
  }
}
```

* 呼び出し元に例外処理を強制する場合は `Exception` クラスのサブクラスを作る
  * ただし `RuntimeException` のサブクラスにしない
* 呼び出し元に例外処理を強制しない場合は `RuntimeException` クラスのサブクラスを作る

---

3.例外にメッセージ、原因を設定する
---

`Exception` クラスのコンストラクターには以下のものがあり、例外の原因を伝えられる

* `Exception(String message)`
* `Exception(Exception cause)`
* `Exception(String message, Exception cause)`

`Exception` クラスから情報を取得する方法

* `getMessage()` メソッドによって、例外に設定されたメッセージを取得できる
* `getCause()` メソッドによって、例外の原因の例外を取得できる

```java
Exception outOfGas = new NoBeerException("out of gas");
Exception exception = new NoBeerException("Beer has gone", outOfGas);

exception.getMessage(); // Beer has gone
exception.getCause(); // -> NoBeerException : out of gas
```

---

4.発生箇所を特定する
---

* `Exception` クラスには、呼び出しの階層を記録する機能がついている
* `getStackTrace()` メソッドにより、呼び出しの階層の記録 `StackTraceElement` の配列を取得できる
* `StackTraceElement` には次の情報が含まれている
  * クラス名
  * メソッド名
  * ファイル名
  * 行番号

### 例外発生箇所を特定する

```text
java.lang.NullPointerException
  at com.example.Library.nameOf(Library.java:22)
  at com.example.UserCode$LibraryExecutor.execute(UserCode.java:49)
  at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Me
  at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMeth
  at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(Delega
  at java.base/java.lang.reflect.Method.invoke(Method.java:567)
  at com.example.UserCode.reflectiveInvoke(UserCode.java:63)
  at com.example.UserCode.main(UserCode.java:31)
```

このスタックトレースからは...

* `Library.java` というファイルの `22` 行目
* `Library` というクラスにある `nameOf` というメソッドの中で
* `NullPointerException` が発生した

5.失敗を呼び出し元に通知する方法
---

* `return` の代わりに `throw` キーワードに例外オブジェクトを指定することでメソッドの実行を終了し、失敗したと呼び出し元に通知できる

```java
if (beer.isPresent()) return Pair.of(beer.get(), change);

// メソッドの実行を終了して失敗を通知
throw new NoBeerException("beer has gone");
```

* なお、通知する例外が `Exception` クラスのサブクラスで、 `RuntimeException` のサブクラスではない場合、
メソッドのシグニチャー(メソッドの型情報)に `throws` キーワードと通知する例外クラスを記述する必要がある

```java
public Pair<Beer, Change> pouringBeer()
      throws NoBeerException, NoChangeException {
  ...
  if (beer.isPresent()) return Pair.of(beer.get(), change);

  // メソッドの実行を終了して失敗を通知
  throw new NoBeerException("beer has gone");
}
```

6.例外処理

### 例外処理をおこなう場合...

* 例外が発生しうる箇所を `try {}` ブロックで囲む
* 処理対象の例外型を `catch()` で指定し、例外パラメーターを受け取る
* 続くブロックに例外処理を記述する

```java
try {
  Beer beer = beerServer.pourBeer(1);
  Change change = changeReserve.prepareChange(payment);
  return Pair.of(beer, change);
} catch (NoBeerException e) {
  // ビールがない場合の例外処理を記述する
} catch (NoChangeException e) {
  // お釣りがない場合の例外処理を記述する
}
```

### `catch` のマッチング

* `catch()` でのマッチングは上から順番に行われる
* `Exception` や `RuntimeException` などのルートになる例外を上の方に指定すると、処理できないルートが発生し、コンパイルエラーになる

```java
try {
  Beer beer = beerServer.pourBeer(1);
  Change change = changeReserve.prepareChange(payment);
  return Pair.of(beer, change);
} catch (Exception e) {
  ...
} catch (NoBeerException e) { // !!コンパイルエラーになる!!
  ...
}
```

### 最後にかならず行いたい処理がある場合

* `finally {}` ブロックに処理を記述する
* `AutoCloseable` の実装クラスを作り、 `close()` メソッドに処理を記述した上で、 `try() {}` のカッコ内に指定する

```java
BeerValve valve = ...;
try {
  valve.open();
  MugOfBeer mugOfBeer = pourBeer(mugCup);
  return mugOfBeer;
} finally {
  valve.close();
}
```

```java 
BeerValve valve = ...;
AutoCloseable closingValve = new AutoCloseable() {
  @Override public void close() throws Exception {
    valve.close();
  }
};
try(closingValve) {
  valve.open();
  MugOfBeer mugOfBeer = pourBeer(mugCup);
  return mugOfBeer;
}
```

* `AutoCloseable` は複数個指定できる

```java 
BeerValve valve = ...;
AutoCloseable closingValve = valve::close;

try(closingValve; MugHolder mugHolder = openMugHolder()) {
  MugCup mugCup = mugHolder.cupAtHead();
  valve.open();
  MugOfBeer mugOfBeer = pourBeer(mugCup);
  return mugOfBeer;
} // mugHolder -> closingValve の順に実行される

// try() の中にて取得したオブジェクトのスコープは try {} ブロック内
```
