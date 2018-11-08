[@molpako](https://twitter.com/molpako) です！

Pythonを勉強していて並行処理あたりが難しいと感じたので、Golangと比較しながらまとめていきます。

[前回](https://molpako.hatenablog.com/entry/2018/11/05/235541) では、threadingモジュールを勉強しました。
今回は、multiprocessingを勉強していきます！

## multiprocessing

multiprocessingのサンプルコードをみると start() や join() というメソッドがあるしthreadingと同じじゃん！マルチスレッドとマルチプロセスはどっちを使えばいいんだ！と感じましたが、ドキュメントを見ると答えが書いてありました。multiprocessingモジュールの目的は **並列処理** ということです。

threadingの所で説明しましたが、PythonはGILという仕組みがあって、それがスレッドを同時に一つのスレッドしか動かさないようにしています。multiprocessingはその問題を解決するモジュールらしく、名の通り複数のプロセスを使いマルチコアの恩恵を受け、並列処理ができるみたいです。早速CPUバウンドな処理を並列にして高速化してみましょう。まずは並列ではなく、順番に実行します。

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

処理時間は約7秒でした。次に、プロセスクラスを作成し並列に実行していきます。複数プロセスの終了を待機するには、Threadクラスと同じように join() を使います。

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

並列実行の場合の処理時間は約4.9秒！正直パフォーマンス的にもっと早くなるものかと思っていましたが、CPUバウンドな処理でも並列実行され時間が短縮できたのが確認できました。これは、多分、おそらく、予想ですが、プロセスはスレッドより重くオーバーヘッドがありメモリ使用量も多いから？と思います。

ちなみに、multiprocessingのPoolクラスを使用すると上記よりも少ないコード量でワーカープロセスのプールを制御し複数のプロセスを並列に動かすことができます。作成した素因数分解する関数 `factorize()` を使用して試してみましょう。

```python
def call_factorize(number):
    """イテレーターをリストに変換する"""
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

ちなみにgolangでは、前回と同じようにgoroutine [^1] を使えば計算処理もはやくなります。

### メモリの共有

2つのプロセス間でデータのやり取りをするためには、Pipeクラスを使用します。（Queueクラスもありますが、別の記事で紹介します！）[^2]

Pipe()が返すコネクションオブジェクトは send() , recv() などのメソッドがあり、socketオブジェクトに似ていますね。早速Pipeクラスを使用して2つのプロセス噛んでデータをやり取りしてみましょう。

`FactorizeProcess` を少し変更して、コネクションオブジェクトを扱えるようにします。プロセスが開始されると、計算をし、結果をパイプ先のプロセスへと送信します。

```python
class PipeFactorizeProcess(multiprocessing.Process):
    """計算するプロセスの各処理を表すクラス
    結果をパイプ先のプロセスに送信する"""

    def __init__(self, numbers, conn):
        super().__init__()
        self.numbers = numbers
        self.conn = conn

    def run(self):
        for number in self.numbers:
            self.conn.send(list(factorize(number)))
        self.conn.close()
```

受信用のプロセスは、5秒間データが受信できるか確認します。確認ができたら受信をして、データの受信がなければコネクションを閉じます。

```python
# Pipe()は、パイプの両端を表すConnectionオブジェクトのペアを返す！
parent_conn, child_conn = multiprocessing.Pipe()
p = PipeFactorizeProcess(numbers, child_conn)
p.start()
while True:
    if parent_conn.poll(5):
        print('receive: {}'.format(parent_conn.recv()))
    else:
        parent_conn.close()
        break
p.join()

>>>
receive: [1, 5501, 9733, 53541233]
receive: [1, 21235343]
receive: [1, 11, 383, 2711, 4213, 29821, 1038313, 11421443]
receive: [1, 5423123]
```

ちなみにgolangでは、プロセスやスレッドを扱わず、goroutineを扱います。goroutine間でのデータのやり取りはチャネルの通信によってデータのやり取りを行います。

```go
// 素因数分解する関数
func factorize(numbers []int, c chan<- []int) {
	for _, number := range numbers {
		var a []int
		for i := 1; i < number+1; i++ {
			if number%i == 0 {
				a = append(a, i)
			}
		}
		c <- a
	}
	// 送信側がチャネルをクローズする
	close(c)
}


func main() {
	numbers := []int{53541233, 21235343, 11421443, 5423123}
	c := make(chan []int)
	go factorize(numbers, c)

	for i := range c {
		fmt.Printf("receive: %d\n", i)
	}
}

>>>
receive: [1 5501 9733 53541233]
receive: [1 21235343]
receive: [1 11 383 2711 4213 29821 1038313 11421443]
receive: [1 5423123]
```
## まとめ

- start()やjoin()などthreadingとAPIが似ている。（ので、移行がしやすい）
- threadingと違い、マルチコア実行ができる。
- Poolクラスにより複数プロセスの管理が簡単になる。
- Pipeクラスにより二つのプロセスでデータをやり取りできる。

次回は **並列性** のための councurrent モジュールについて勉強しますー！

## 参考文献 

- [Python 標準ライブラリ 17. 並行実行](https://docs.python.jp/3/library/concurrency.html)
- [Effective Python --Pythonプログラムを改良する59項目](https://www.oreilly.co.jp/books/9784873117560)

[^1]: スレッドより小さい軽量スレッド。Golangで並行・並列処理する場合には、goroutineを扱う。
[^2]: データをやり取りするためにメモリを共有する方法があるが、デフォルトではメモリを共有しない。プロセス同士でメモリを共有したい場合は、ValueクラスやArrayクラスもしくは [multiprocessing.sharedctypes](https://docs.python.jp/3/library/multiprocessing.html#module-multiprocessing.sharedctypes) を使用する。
