[@molpako](https://twitter.com/molpako) です！

Pythonを勉強していて並行処理あたりが難しいと感じたので、Golangと比較しながらまとめていきます。

**並行処理 (concurrency)**

複数のタスクを **見かけ上** 同じ時間に実行すること。OSが1コア上で実行するプロセスを切り替えている。他のタスクを待たせないのが目的。

**並列処理(parallelism)**

見かけ上ではなく、**実際に** 複数のタスクを同じに時間に実行すること。複数のコアが別々の仕事を実行している。早くするのが目的。

**ブロッキングI/O**

I/O処理中は待機するようなI/Oのこと。同期I/O。

## モジュール

「Python 並行処理」などで検索するといくつかのモジュールがでます。基本的には以下と思います。

- [threading](https://docs.python.jp/3/library/threading.html)
- [multiprocessing](https://docs.python.jp/3/library/multiprocessing.html)
- [concurrent](https://docs.python.jp/3/library/concurrent.html)
- [subprocess](https://docs.python.jp/3/library/subprocess.html)
- [queue](https://docs.python.jp/3/library/queue.html)
- [asyncio](https://docs.python.jp/3/library/asyncio.html)

まずはこれらのモジュールの使い方から勉強していきます。

### queue

threadingモジュールとmultiprocessingモジュールにもQueueクラスは提供されていますが、queueモジュールというものあってそれにもQueueクラスがあります。。。
// スレッドやプロセス間のデータの受け渡しを主にかく。

### concurrent

// 勉強中


### asyncio

// 勉強中

## 参考文献

[https://docs.python.jp/3/library/concurrency.html/:embed:cite]
[https://www.oreilly.co.jp/books/9784873117560/:embed:cite]
[http://ascii.jp/elem/000/001/440/1440099/:embed:cite]
