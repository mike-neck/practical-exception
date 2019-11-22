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


