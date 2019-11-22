プラクティス
===

ここでは、これまでの自分の経験と書籍で学んだプラクティスを紹介していく。

1.ルート例外を使わない
---

* ルートの例外(`Exception`/`RuntimeException`)で失敗を通知すると、メソッドの利用者にはどのような問題が発生したか判別がつかない

メソッド

```java 
Beer pourBeer(Mug mug) throws Exception {
  MugOfBeer mugOfBeer = new MugOfBeer(mug.size());
  while(tank.remains()) {
    BeerBuffer buffer = tank.bufferOfBeer();
    mugOfBeer.add(buffer);
    if (nugOfBeer.isFilled()) return mugOfBeer.finish(tank.bubbles());
  }
  throw new Exception("out of beer");
}
```

メソッドの呼び出し元メソッド

```java 
try {
  // RuntimeException のサブクラス NoMoreMugException で通知する
  Mug mug = mugHolder.head();
  // Exception で通知する
  Beer beer = beerServer.pourBeer(mug);
  // NoChangeException で通知する
  Change change = changeReserve.prepareChange(payment);
  return Contract.success(beer, change);
} catch (Exception e) {
  // BeerServer#pourBeer から通知された例外のハンドリング
}
```

* `MugHolder#head` で発生した `NoMoreMugException` はこのメソッド内で例外処理をしたくないのだが、
`Exception` を捕捉してしまうため、従来の意図に反して例外処理されてしまう
* `ChangeReserve#prepareChange` から `NoChangeException` で通知された例外のハンドリングがないため
機能落ちしているが、そのバグ(事後条件違反)をコンパイラーで検出できない

---

2.例外のデフォルトコンストラクターを使わない - あるいは例外オブジェクトを生成する場合は必ず詳細メッセージを設定する
---

* 例外を通知するメソッドのコードを書いている人には自明な状態でも、そのメソッドのユーザーにとっては自明ではない
* 運用フェーズのエンジニア(エラーを分析してソフトウェアを改善する)にとっては、エラーメッセージだけが唯一の問題に関する情報なので、改善の機会の芽を摘まない

##### Slf4J - logback のログ出力 1

クイズ : 以下のログから例外が発生した原因は何か分析せよ

```text
2019/05/21 21:16:11.871, main, INFO, com.example.web.OrderController, unexpected error
com.example.beer.vendor.BeerLeakingException: null
  at com.example.beer.vendor.BeerValve.closeValve(BeerValve.java:1034)
  at com.example.beer.vendor.BeerServer.pourBeer(BeerServer.java:429)
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/java.util.AbstractList$RandomAccessSpliterator.forEachRemaining(A
...
  at com.example.beer.BeerService.newMugOfBeer(BeerService.java:643)
  at com.example.beer.OrderController.newMugOfBeer(OrderController.java:311)
```

答え : わからん

##### Slf4J - logback のログ出力 2

クイズ : 以下のログから例外が発生した原因は何か分析せよ

```text
2019/05/21 21:16:11.871, main, INFO, com.example.web.OrderController, unexpected error
com.example.beer.vendor.BeerLeakingException: valve not closed completely,
beer-temperature: 4C, outside-temperature: 33C, usage-count: 3045
  at com.example.beer.vendor.BeerValve.closeValve(BeerValve.java:1034)
  at com.example.beer.vendor.BeerServer.pourBeer(BeerServer.java:429)
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/jva.util.stream.ReferencePipleline$3$1.accept(Referencepipeline.j
  at java.base/java.util.AbstractList$RandomAccessSpliterator.forEachRemaining(A
...
  at com.example.beer.BeerService.newMugOfBeer(BeerService.java:643)
  at com.example.beer.OrderController.newMugOfBeer(OrderController.java:311)
```

答え

* バルブが完全に閉じていない
  * ビールの温度 4 ℃
  * 外気温 33 ℃
  * バルブの開閉回数は `3045` 回の時に発生

##### 回避方法

* 独自例外クラスを作る場合は、デフォルトコンストラクターを利用できないようにする

```java
class BeerLeakingException extends Exception {

  final int beerTemperature;
  final int outsideTemperature;
  final int usageCount;

  BeerLeakingException(String message, int beerTemperature,
      int outsideTemperature, int usageCount) {
    super(message);
    this.beerTemperature = beerTemperature;
    this.outsideTemperature = outsideTemperature;
    this.usageCount = usageCount;
  }
  // BeerLakingException() デフォルトコンストラクターを作らない 
}
```

##### 注意

* 例外のコンストラクターに設定するメッセージは、必ずしもエンドユーザーに見せるメッセージではない
  * `ResourceBundle` などの仕組みで解決する

```java 
try {

} catch (BeerLeakingException e) {
  logger.info("unexpected error", e);
  return Response
      .temporaryUnavailable(
          Map.of("message", e.getMessage()));
}
```

---


3.例外を無視しない
---

* 事後条件を保証しないバグ
* 不変条件も壊している可能性があり、後で想定もできないような不具合に発展する可能性がある

### 例外無視の3大パターン

* 何もしてない
* 翻訳した例外が連鎖していない
* `finally` で `return`/`throw` している

##### 何もしてない

* 例外の発生に対して回復もせずに、成功のように振る舞うやつ
* 問題の発生を検知できないため、発見が遅れて、その結果問題をややこしくしてしまう

```java
void doPost(Request request, Buffer buffer) {
  try {
    Result result = service.doSomeService(request.getParam());
    buffer.put(result.bytes());
  } catch (SomeException e) {
  }
}
```

* コンパイラーをうまく使えば発見・修正できるかもしれない
  * 例外処理を忘れたらコンパイルエラーになるプログラミングモデルにする
* 具体的には `void` のメソッドをなくす
* 弱点は例外を無視される場合、 `null` を返される

```java
// void をやめて、 Result を返すようなメソッドにする
Result doPost(Request request) {
  try {
    return service.doSomeService(request.getParam());
  } catch (SomeException e) {
    return Result.defaultError(e.errorType());
  }
}
```

##### 翻訳した例外が連鎖していない

* 例外翻訳した際に発生する
* ログ等から翻訳した場所はわかるものの、根本原因にたどり着けない

```java 
void doPost(Request request, Buffer buffer) {
  try {
    Result result = service.doSomeService(request.getParam());
    buffer.put(result.bytes());
  } catch (SomeException e) { // e が無視されている
    // 連鎖していない
    throw new ApplicationException("some exception”);
  }
}
```

* IntelliJ IDEA なら対策はできそう
  * Preferences -> Inspections -> Java -> Error Handling -> Catch block may ignore exception にチェックする
    * Do not warn when 'catch' block is not empty にチェックする
* Eclipse? ごめん...
* spotbugs でできないか調べたが、ちょっとできなさそうだった(詳しい人教えて)

##### `finally` で `return`/`throw` している

* `finally {}` での `return`/`throw` は `try {}` および `catch(){}` での  `return`/`throw` を上書きしてしまう
* `try {}` の中を適切に記述したのに、挙動がおかしくなる

```java
Result doPost(Request request) {
  try {
    return service.doSomeService(request.getParam());
  } catch (SomeException e) {
    throw new ApplicationException(e);
  } finally {
    return Result.getDefault(); // 優先される
  }
}
```

* 一応、コンパイルエラーで防げる

```shellsession
$ javac -Xlint:finally -Werror BadFainallyReturn.java
BadFainallyReturn.java:13: 警告:[finally] finally節が正常に完了できません
    }
    ^
エラー: 警告が見つかり-Werrorが指定されました
エラー1個
警告1個
```

* `AutoCloseable` を使って try-with-resources を徹底する
* `finally` を使わせない

4.概念に適切な例外を用いる
---

### 適切なレイヤーの例外を返すようにする。業務ロジックに実装の詳細がわかってしまうような例外を通過させない

* ビジネスロジック(業務専門のロジックが記述されている)とインフラ(データベースや外部のAPIと通信するレイヤー)の
境界にあたるインターフェースには実装がどのようであっても実装技術の例外を宣言しない
* また、そのような例外を素通しさせない

```java 
interface FileStorage {
  ReservedFileSpace reserveFileSpace(
      UserId userId, AttachmentFile file);
}
```

上記のインターフェースに対して、 AWS を使っていてどうしても避けられないといって、次のような `throws` をつけない

```java
interface FileStorage {
  ReservedFileSpace reserveFileSpace(
      UserId userId, AttachmentFile file)
          throws AmazonS3Exception;
}
```

実装クラスで発生を完全に防げない実装技術に関する例外が発生してしまう場合、
例外翻訳をおこなって、実装技術の情報が業務ロジックに現れないようにする

```java 
@Override
public ReservedFileSpace reserveFileSpace(
      UserId userId, AttachmentFile file)
          throws TechnicalException {
  try {
    ......
  } catch (AmazonS3Exception e) {
    throw new TechnicalException("URLの生成に失敗しました", e);
  }
}
```

5.検査例外の使い分け
---

特に事情がなければ検査例外を使わない

### 検査例外を使って良い条件

Effective Java 71 より

* API の適切な使用では例外状態を防げない
* API を使っているプログラマが有用な例外処理ができる
  * API の利用者がモジュールの状態に関する知識を持っており、それを制御する API も提供している
  * API の利用者の不変条件がモジュールの状態を含んでいる

```java 
try {
  dao.beginTransaction();
  dao.insert(userId, messageId, message);
  dao.commit();
} catch (DatabaseException e) {
  dao.rollback();
}
```

6.本当に例外が必要なところだけに例外を使う
---

* Effective Java 69
  * 例外は、その名が示す通り、例外的条件に対してのみ使うべきです。通常の制御フローに対しては、使うべきではありません。
* 達人プログラマー 24
  * 「すべての例外ハンドラを除去しても、このプログラムは動作可能だろうか？」と自問してください。答えが「ノー」であれば、例外ではない状況下で例外が使われているはずです。

```java 
try {
  Iterator<Item> iterator = items.iterator();
  while (true) {
    Item item = iterator.next();
    if (item.hasStock()) {
      throw new ItemFoundException(item);
    }
  }
} catch (ItemFoundException e) {
  return Optional.of(e.getItem());
} catch (NoSuchElementException e) {
  return Optional.empty();
}
```

* 不適切な用途に用いた例外は、読む人の注意を逸らせてしまう
* 例外に基づくループは動作を保証できない
  * ループ内で同じ例外を発生させる箇所があった場合、ループ終了の例外であるかを判定できない
* 結果、バグの存在を隠蔽してしまう

##### 意外なところに潜んでいる通常の制御フローにひそむ例外

```java 
class UserInputReport {
  final String reportDate = …;
  final String reportTitle = …;
  final String reportContent = …;
  
  void validate() {
    if (isInvalidDateFormat(reportDate)) {
      throw new BadUserInputException("report_date", "invalid format");
    }
    if (!isLessThan120BytesInUtf8(reportTitle)) {
      throw new BadUserInputException("report_title", “too long");
    }
    if (!isLessThan6000BytesInUtf8(reportTitle)) {
      throw new BadUserInputException("report_content", "too long");
    }
  }
}
```

利用クラス

```java 
UserInputReport userInputReport = …;
try {
  userInputReport.validate();
} catch (BadUserInputException e){
  return Response
      .badRequest()
      .body(Map.of(e.getField(),e.getErrorMessage()));
}
```

martinfowler.com より

* Exceptions signal something outside the expected bounds of behavior of the code in question. But if you’re running some checks on outside input, this is because you expect some messages to fail - and if a failure is expected behavior, then you shouldn’t be using exceptions.
(要約:ユーザー入力が間違えているのは予測の範囲内なので例外は使うべきでない)

---

修正方法

* Notification パターンを使う

```java 
class UserInputReport {
  final String reportDate = …;
  final String reportTitle = …;
  final String reportContent = …;
  
  ErrorNotification validate() {
    List<InputError> inputErrors = new ArrayList<>();
    if (isInvalidDateFormat(reportDate)) {
      inputErrors.add(new InputError("report_date", "invalid format"));
    }
    if (!isLessThan120BytesInUtf8(reportTitle)) {
      inputErrors.add(new InputError("report_title", “too long"));
    }
    if (!isLessThan6000BytesInUtf8(reportTitle)) {
      inputErrors.add(new InputError("report_content", "too long"));
    }
    return new ErrorNotification(inputErrors);
  }
}
```

利用クラス

```java
UserInputReport userInputReport = …;
InputErrorNotification notification =  userInputReport.validate();
if (notification.hasError()){
  return Response
      .badRequest()
      .body(notification.errorMessages());
}
```

制御フローから例外がいなくなったのがわかる
