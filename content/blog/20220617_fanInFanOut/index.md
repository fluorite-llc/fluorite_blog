---
title: "[golang][並行処理] fanInFanOut"
date: "2022-06-17T09:25"
description: Go言語による並行処理に記載されている fanIn fanOutを可読性向上 + Context対応させてみました
---

golangが好きな fです。こんにちは。
まだまだにわかgopherです。はやく一人前になりたいですね。
(一人前の定義って、、、)

今回は並行処理に便利なfanInFanOut処理を実装したいと思います。
参考書も購入しております。

[Go言語による並行処理](https://www.amazon.co.jp/dp/4873118468)

(それにしても、リンクをカードにする仕組が欲しいですね、、、)


## fanIn, fanOutとは

正確な認識ではないかもしれませんが、
xargs で標準入力を受けて複数のプロセスを起動させる処理を考えてみれば
ちょうどそれに相当する処理だという認識です。

複数のプロセスを起動させる処理が fanOut。
起動させたプロセスの標準出力を一つにまとめて受けとるのを fanIn。

一時流行したMapReduceと似たような処理ですよね。

それでは実装です(--;)。


今回参考書に載っているor-doneチャネルを使って実装したいと思います。

## or-doneチャネルとは

参考書の説明によると、

```
for val := range myChan {
  // valに対して何かする
}
```
`Go言語による並行処理より引用`

と簡潔に記述できるはずのコードが
キャンセル処理を導入するだけで

```
loop:
for {
  select {
    case <-done:
      break loop
    case maybeVal, ok := <-myChan:
      if ok == false {
        return
      }
      // valに対して何かする
    }
  }
}
```
`Go言語による並行処理より引用`

という形で可読性が低下してしまう(慣れればどうってことないですが)
リーダブルコード的にはインデントが深すぎって言われる典型的なケースとなってしまうので
それを避けるためのテクニックと紹介されています。


```
“早すぎる最適化ではなく、並行処理のコードをよりすっきりと書くためにゴルーチンを使うという点をさらに進めると、ゴルーチンを1つ使うことでこの煩雑なコードを直すことができます。この冗長な状況をカプセル化して、他の人が触らないで済むようにするのです。”
```
`Go言語による並行処理より引用`


これは正直目からウロコでした。
すばらしい書籍ですね。
or-doneチャネルの実装は実際に書籍を取って参照ください。


## fanIn の実装(or-doneチャネル付き)

書籍に記載されているfanInの処理はキャンセレーションの導入がされているものの、or-doneチャネルが導入されていません。
contextも導入されていませんね。
contextとor-doneチャネルを導入した実装をここに乗せておきます。


```
func fanIn(ctx context, channels ...<-chan interface{}) <-chan interface{}{
  var wg sync.WaitGroup
  multiplexedStream := make(chan interface{})

  mutiplex := func(c <-chan interface{}) {
    defer wg.Done()
    for i := range orDone(ctx, c) {
      multiplexedStream <- i:
    }
  }

  wg.Add(len(channels))
  for _, c := range channels {
    go multiplex(c)
  }

  go func() {
    wg.Wait()
    close(multiplexedStream)
  }()

  return multiplexedStream
}
```


この処理を使う側はfanInが返すmultiplexedチャネルをfor + range ループで扱うため、チャネルがクローズされるまで待つということになります。
チャネルのクローズはwg.Wait()が完了しない限り呼ばれないので、結果として使う側は同期処理のように記述することができるということになりますね。
キャンセルが実行された場合にはorDoneチャネルがcloseし、問題なくwg.Done()が呼ばれますね。
すばらしい。

そんなに複雑な処理ではないのですが、wg.Wait() を goroutine内で呼びだしている発想が自分にはなかったですね。
goroutine内で実行するのはアンチパターンだという認識でした。とはいえ、流れが理解できない以上は避けた方がよいパターンだと思います。


## fanInFanOut

fanOutの処理はシンプルです。

```
type Worker func(context.Context, int, <-chan interface) <-chan interface

func FanOut(ctx context.Context, numWorkers int, worker Worker, in <-chan interface{}) []<-chan interface{} {
  outputs := make([]<-chan interface, numWorkers)
  for i := 0; i < numWorkers; i++ {
    outputs[i] = worker(ctx, i, in)
  }
  return outputs
}
```

チャネルのスライスを返すので、これをfanInに渡してあげるればOK。

```
func FanInFanOut(ctx context.Context, numWorkers int, worker Worker, in <-chan interface{}) <-chan interface{} {
  return FanIn(ctx, FanOut(ctx, numWorkers, worker, in)...)
}
```

## まとめ

今回はGo言語による並行処理に記載されている fanIn, fanOut処理を or-doneチャネルを使って context対応させてみました。
goroutine内でwg.Wait() + close(ch) するというトリックと or-done のまるで言語を拡張したかのような可読性向上トリックがすばらしいとこの書籍を購入して良かったとしみじみ感じました。

or-done実装のcontext対応が重要なんじゃないかと指摘を受けるかとは思いますが、また次回機会があれば。
