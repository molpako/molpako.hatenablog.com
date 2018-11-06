[@molpako](https://twitter.com/molpako) です！

Pythonを勉強していて並行処理あたりが難しいと感じたので、Golangと比較しながらまとめていきます。

[前回](https://molpako.hatenablog.com/entry/2018/11/05/235541) では、threadingモジュールを勉強しました。
今回は、multiprocessingを勉強していきます！

## multiprocessing

multiprocessingのサンプルコードをみると start() や join() というメソッドがあるしthreadingと同じじゃん！マルチスレッドとマルチプロセスはどっちを使えばいいんだ！と感じましたが、ドキュメントをみると答えが書いてありました。multiprocessingモジュールの目的は **並列処理** ということです。

threadingのところで説明しましたが、PythonはGILという仕組みがあって、それがスレッドを同時に一つのスレッドしか動かさないようにしています。multiprocessingはその問題を解決するモジュールらしく、名の通り複数のプロセスを使いマルチコアの恩恵を受け、並列処理ができるみたいです。早速CPUバウンドな処理を並列にして高速化してみましょう。まずは、順番に実行します。

```python
def factorize(number):
    """素因数分解する関数"""
    for i in range(1, number + 1):
        if number % i == 0:
            yield i

numbers = [53541233, 21235343, 11421443, 5423123]

from time import time
start = time()
for number in numbers:
    list(factorize(number))

print('Took %.3f seconds' % (time() - start))

>>>
Took 7.344 seconds
```

処理時間は約7秒かかりました。次に、プロセスクラスを作成し並列に実行していきます。複数プロセスの終了を待機するには、Threadクラスと同じように join() を使います。

```python
import multiprocessing

class FactorizeProcess(multiprocessing.Process):
    """計算するプロセスの各処理を表すクラス"""
    def __init__(self, number):
        super().__init__()
        self.number = number
    
    def run(self):
        self.factors = list(factorize(self.number))

# プロセスの開始
start = time()
procs = []
for number in numbers:
    proc = FactorizeProcess(number)
    proc.start()
    procs.append(proc)

for proc in procs:
    proc.join()

print('Took %.3f seconds' % (time() - start))

>>>
Took 4.885 seconds
```

並列実行の場合の処理時間は約4.9秒！正直パフォーマンス的にもっと早くなるものかと思っていましたが、CPUバウンドな処理でも並列実行され時間が短縮できたのが確認できました。これは、多分、おそらく、予想ですが、プロセスはスレッドより重くオーバーヘッドがありメモリ使用量も多いからと考えられます。

ちなみに、multiprocessingのPoolクラスを使用すると上記よりも少ないコード量でワーカープロセスのプールを制御し複数のプロセスを並列に動かすことができます。factorize()を使用して試してみましょう。

```python
def call_factorize(number):
    """ジェネレーターを呼ぶための関数"""
    return list(factorize(number))


start = time()

# 計算する要素分プロセスを立ち上げる
with multiprocessing.Pool(len(numbers)) as pool:
    results = pool.map(call_factorize, numbers)

    for result in results:
        print(result)

print('Took %.3f seconds' % (time() - start))

>>>
[1, 5501, 9733, 53541233]
[1, 21235343]
[1, 11, 383, 2711, 4213, 29821, 1038313, 11421443]
[1, 5423123]
Took 5.252 seconds
```

## まとめ

- start()やjoin()などthreadingとAPIが似ている。（ので、移行がしやすい）
- threadingと違い、マルチコア実行ができる。
- Poolクラスにより複数プロセスの管理が簡単になる。
