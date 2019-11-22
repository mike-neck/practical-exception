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

クイズ : 以下のログが発生した原因は何か分析せよ

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

クイズ : 以下のログが発生した原因は何か分析せよ

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



