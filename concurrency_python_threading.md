[@molpako](https://twitter.com/molpako) です！

Pythonを勉強していて並行処理あたりが難しいと感じたので、Golangと比較しながらまとめていきます。

## 並行処理で使用する標準ライブラリ

「Python 並行処理 [^1] 」などで検索するといくつかのモジュールがでます。基本的には以下が出てくると思います。

- [threading](https://docs.python.jp/3/library/threading.html)
- [multiprocessing](https://docs.python.jp/3/library/multiprocessing.html)
- [concurrent](https://docs.python.jp/3/library/concurrent.html)
- [queue](https://docs.python.jp/3/library/queue.html)
- [asyncio](https://docs.python.jp/3/library/asyncio.html)

このモジュールたちの使い方、使うべきタイミングなど勉強して行きましょうー！まずは、threadingから！


## threading

threadingはスレッドを扱うモジュールです。正直スレッドに対しては「プロセスより軽い物でマルチスレッドとかで並行か並列処理できるんだろう」という認識しかなく、全然具体的なイメージを持っていませんでした。Pythonは並列処理 [^2] は向いていないということをネットでみたりしていたので「スレッド扱えるなら並列処理できるんじゃないの」と思っていましたがその曖昧な認識が間違っていたことに勉強してやっと気づけました。「実行がマルチスレッド」＝「CPUのマルチコアを活用できる」＝「並列処理できる」という認識は、実際C++やJavaのような言語では間違ってないみたいですが、Pythonではそうではないみたい。。。それはなぜかと言うと、PythonのGIL(グローバルインタプリタロック) [^3] という仕組みが同時に一つのスレッドしか進行できないようにしているからです。

つまりPythonではマルチスレッドを使用してもマルチコアの恩恵を受けられず並列処理でスピードアップできないとのことです！スピードアップできないならいつ使うの！？と思っていたのですが、使うべきタイミングはドキュメントに以下のように書いていました。

>I/Oバウンドなタスクを並行して複数走らせたい場合においては、 マルチスレッドは正しい選択肢です。

と記載されている通りI/Oバウンドタスクをスレッドで実行しプログラムから隔離することによって、ブロッキングI/O [^4] の処理を行いながら必要な処理ができます。それでは、 [select](https://docs.python.jp/3/library/select.html#select.select) を使用して0.1秒のI/Oイベントを発生させる関数slow_syscall()を作成し、実験してみます。

```python
from time import time
import select
import socket

def slow_syscall():
    """遅いシステムコールを実行する関数"""
    select.select([socket.socket()], [], [], 0.1)


# メインの実行スレッドが 1秒(=0.1 * 10) ブロックされる
start = time()
for _ in range(10):
    slow_syscall()
print('Took %.3f seconds' % (time() - start))

>>>
Took 1.024 seconds
```

複数のシステムコールを別々のスレッドで実行します。
```python
start = time()
threads = []
for _ in range(10):
    thread = threading.Thread(target=slow_syscall)
    thread.start()
    threads.append(thread)

# join()で全てのスレッドの処理が終了するまで待機する
for thread in threads:
    thread.join()
print('Took %.3f seconds' % (time() - start))

>>>
Took 0.103 seconds
```

スレッドにブロッキングI/Oを処理させることで並列に実行され処理時間が約1/10になりました。ちなみにGolangでは、goroutineという軽量スレッドと使用し複数のgoroutineにそれぞれブロッキングI/Oの処理をさせることができます。下の例では順番に slowSyscall() を実行したので約1秒かかりました。

```go
func slowSyscall() {
	fd, _ := syscall.Socket(
		syscall.AF_INET,
		syscall.SOCK_DGRAM,
		syscall.IPPROTO_UDP,
	)

	// 0.1秒かかるようにする
	timeout := &syscall.Timeval{Sec: 0, Usec: 100000}
	syscall.Select(fd, nil, nil, nil, timeout)
}

func Successive() int {

	// 逐次的に関数を実行させる
	for i := 0; i < 10; i++ {
		slowSyscall()
	}
	return 0
}

// --- PASS: TestSuccessive/#00 (1.02s)
```

slowSyscall() をそれぞれgoroutineに実行してもらうと、並列に実行され処理時間が0.1秒とpythonのthreadingと同じ結果になりました。

```go
func Concurrency() int {
	wg := &sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		wg.Add(1)

		// goroutineを立ち上げ関数を実行させる
		go func() {
			slowSyscall()
			wg.Done()
		}()
	}

	// 複数のgoroutineの処理が終わるまで待機する
	wg.Wait()
	return 0
}
// --- PASS: TestCunccurency/#00 (0.10s)
```

スレッドは、メモリ空間を共有するので複数のスレッドがグローバルなオブジェクトを扱うときは危険です。ロックを使って回避します。10個のスレッドを並列に実行し、スレッドでカウンタを上げていくプログラムで、データ競合を起こしてみます。

まずは、変数とカウンタクラスを用意します。
```python
thread_number = 10

# スレッドでカウンタをあげる回数
call_number = 10**5

class Counter(object):
    """カウントするクラス。スレッドにこのクラスのオブジェクトを渡す"""
    def __init__(self):
        self.count = 0
    
    def increment(self):
        self.count += 1
```

ここで、データ競合を起こしやすくするため [Barrier](https://docs.python.jp/3/library/threading.html#barrier-objects)を使用します。Barrierはブロックのような働きをしてくれて、Barrierオブジェクトを生成する時に指定された数分だけの wait() が呼び出されると、同時にブロックから解放されます。これにより以下ではスレッド立ち上げのオーバーヘッドのせいでデータ競合が発生しにくくなるのを防ぎます。

```python
b = threading.Barrier(thread_number)

def syscall_worker(i, counter):
    """i回システムコールを実行し、その度にカウントを１あげる"""
    b.wait()
    for _ in range(i):
        # 実行しない方が競合が起きやすいので実際にシステムコールは実行しない
        # slow_syscall()
        counter.increment()


# スレッドの開始
threads = []
counter = Counter()
for _ in range(thread_number):
    thread = threading.Thread(
        target=syscall_worker, args=(call_number, counter))

    thread.start()
    threads.append(thread)

for thread in threads:
    thread.join()

print('want: {}, got: {}'.format(
    thread_number * call_number, counter.count))

>>>
want: 1000000, got: 466988
```

出力を見ると、カウンタの数がおかしくなっていました。これはスレッド同士が処理結果を上書きしあいデータの不整合が起きているからみたいでした。threadingではそうのようなデータ競合が起きさせないように、Lockクラスが用意されています。。インクリメントする時に、ロックをかけるようにしてみてもう一度実行してみます。

```python
class LockCounter(object):
    def __init__(self):
        self.lock = threading.Lock()
        self.count = 0
    
    def increment(self):
        with self.lock:
            self.count += 1

# もう一度スレッドの実行をする
...
>>>
want: 1000000, got: 1000000
```

できました！ロックをとった分、実行時間は遅くなりましたがデータ競合は起きてなく求める値が取得されました！

## まとめ

- PythonのGILが、マルチスレッドを使ってもマルチコアの恩恵を受けれないようにしている。
- I/Oバウンドなタスクを扱うときはthreadingモジュールを使う。
- 複数スレッドで同じオブジェクトを扱うときは、threading.Lockクラスを使う。


次回は、multiprocessingを勉強していきますー！

## 参考文献 

- [Python 標準ライブラリ 17. 並行実行](https://docs.python.jp/3/library/concurrency.html)
- [Effective Python --Pythonプログラムを改良する59項目](https://www.oreilly.co.jp/books/9784873117560)
- [Goならわかるシステムプログラミング ― 第12回 ファイルシステムと、その上のGo言語の関数たち（3）](http://ascii.jp/elem/000/001/440/1440099)

[^1]: 複数のタスクを **見かけ上** 同じ時間に実行すること。OSが1コア上で実行するプロセスを切り替えている。他のタスクを待たせないのが目的。
[^2]: 見かけ上ではなく、**実際に** 複数のタスクを同じに時間に実行すること。複数のコアが別々の仕事を実行している。早くするのが目的。
[^3]: スレッド同士がデータ競合しないようにするロック機構のこと。
[^4]: I/O処理中は待機するようなI/Oのこと。同期I/O。