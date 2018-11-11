[@molpako](https://twitter.com/molpako) です！

[前回](https://molpako.hatenablog.com/entry/2018/11/09/015636) では、multiprocessingモジュールを勉強しました。
今回は、concurrentパッケージを勉強していきます！

concurrentパッケージには、一つだけモジュールがあります。それが、並列実行のための [concurrent.futures](https://docs.python.org/ja/3/library/concurrent.futures.html#module-concurrent.futures) です。

## concurrent.futures

concurrent.futuresは、主に二つのクラスを提供しています。

- ThreadPoolExecutor
- ProcessPoolExecutor

この二つのクラスは前回紹介したthreadingとmultiprocessingを呼び出していて、スレッドやプロセスのプールを使用して非同期に実行します。また、両方ともExecutorのサブクラスで同じインターフェースを実装しているので、同じメソッドを提供しています。

では、 前回と前々回で書いた処理をconcurrent.futuresを使用し実装していきます！

### ThreadPoolExecutor

まずは並列に実行する関数の作成。引数によって待つ時間を変えれるようにしています。

```python
import select
import socket

def slow_syscall(timeout=1):
    """遅いシステムコールを実行する関数"""
    select.select([socket.socket()], [], [], timeout)
```

ThreadPoolExecutorを使用して、slow_syscall()を並列に実行します。並列処理の終了を同期するために、withステートメントを使用しています。これは、内部で `shutdown(wait=True)` を呼び出していて、実行されたプール内のスレッドが全て終わるまで待機させています。

```python
from time import time
from concurrent.futures import ThreadPoolExecutor

start = time()
# with を使うことで、pool内の実行がすべて終わるまで待つ
with ThreadPoolExecutor(max_workers=10) as pool:
    for _ in range(10):
        pool.submit(slow_syscall)

print('Took %.3f seconds' % (time() - start))

>>>
Took 1.006 seconds
```


### ProcessPoolExecutor

関数の作成。

```python
def factorize(number):
    """素因数分解する関数"""
    for i in range(1, number + 1):
        if number % i == 0:
            yield i

def call_factorize(number):
    """イテレーターをリストに変換する"""
    return list(factorize(number))
```

ThreadPoolExecutorと同じインタフェースを実装しているため、同じように扱うことができます。ついでに計算した結果も出力しておきます。

```python
from concurrent.futures import ProcessPoolExecutor

numbers = [53541233, 21235343, 11421443, 5423123]
start = time()

with ProcessPoolExecutor(max_workers=2) as pool:
    # mapは呼び出す関数をイテラブルな要素それぞれに対して実行する。
    results = pool.map(call_factorize, numbers)

    for result in results:
        print(result)

print('Took %.3f seconds' % (time() - start))

>>>
[1, 5501, 9733, 53541233]
[1, 21235343]
[1, 11, 383, 2711, 4213, 29821, 1038313, 11421443]
[1, 5423123]
Took 4.070 seconds
```

## まとめ

- threadingやmultiprocessingを直接扱わずとも、concurrent.futuresを使用して並列処理ができる。
- 両方とも簡単なインターフェースを実装していてとても扱いやすい。


次回は、データのやり取りを安全に行うための queueモジュールについて勉強しますー！

## 参考文献

- [Python 標準ライブラリ 17. 並行実行](https://docs.python.jp/3/library/concurrency.html)
- [Effective Python --Pythonプログラムを改良する59項目](https://www.oreilly.co.jp/books/9784873117560)
