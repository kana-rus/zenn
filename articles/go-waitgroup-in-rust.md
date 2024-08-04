---
title:     "Rust で Go の WaitGroup っぽいものを手軽に用意する"
emoji:     "📢"
type:      "tech"
topics:    ["async", "rust", "go"]
published: true
---

現状 tokio 環境でしか試してないので以下 tokio 前提になりますが、tokio なしでも ( [`futures-channel`](https://github.com/rust-lang/futures-rs/tree/master/futures-channel) など使ったり自作したりで ) 工夫すれば同じようにできるとは思います。


## 概要

これです

https://twitter.com/kana_rus/status/1819756492201083265

つまり、

```rust:channel.rs
let (tx, rx) = tokio::sync::watch::channel(());

for _ in 0..3 {
    let rx = rx.clone();
    tokio::spawn(async {
        // do something
        drop(rx)
    })
}

drop(rx);
tx.closed().await;
```

みたいなのが Go の

```go:waitgroup.go
var wg sync.WaitGroup

for i:=0; i<3; i++ {
    wg.Add(1)
    go func() {
        // do something
        wg.Done()
    }()
}

wg.Wait()
```

に対応します。


## 仕組み

`tokio::sync::watch` の `Sender::closed` は対応する `Reciever` が全部 drop されるまで `Pending` な `Future` を返すので、

1. 各 task に対して `let rx = rx.clone()` して終わったら `drop(rx)`
2. 中央で `drop(rx); tx.closed().await`

が

1. 各 goroutine に対して `wg.Add(1)` して終わったら `wg.Done()`
2. 中央で `wg.Wait()`

に対応します。

`closed()` の前に最初に作った `Reciever` を drop する必要がある点が `WaitGroup` と感覚違いますが、いい感じに `watch::channel` をラップしたりしてもっと `WaitGroup` に寄せたものを作ることもできそうです。


## 応用

冒頭で挙げたように graceful shutdown をきれいに実装できるのが応用例の１つになります。

:::details 余談: この類の「になります」について
> 店員「500円になります」
> 客「500円になる?! いつなるの?!」

https://youtube.com/shorts/bJTY1lqKU6E

というのが稀によくありますが、個人的にはこの「になる」は「である」と同じ (「五百円なり」の「なり」みたいな (？) ) 語源からくるものであって「に + 成る」とは関係ない同音異義語だと思っているので、この手の言説は的外れに感じるのですが、どうなんでしょう？ @有識者
:::

せっかくなので先日実装した [`Ohkami`](https://github.com/ohkami-rs/ohkami) の graceful shutdown を紹介しておきます。

https://github.com/ohkami-rs/ohkami/blob/fd0eb16f9d91c383855f6054640b10d2bb0bf78d/ohkami/src/ohkami/mod.rs#L271-L306

要素を抜粋すると

```rust
let ctrl_c = tokio::signal::ctrl_c();

let (ctrl_c_tx, ctrl_c_rx) = tokio::sync::watch::channel(());
tokio::spawn(async {
    ctrl_c.await.unwrap();
    drop(ctrl_c_rx);
});

let (close_tx, close_rx) = tokio::sync::watch::channel(());
loop {
    tokio::select! {
        Ok((connection, _)) = listener.accept() => {
            let close_rx = close_rx.clone();
            tokio::spawn(async {
                /* handle `connection` */
                drop(close_rx)
            });
        },
        _ = ctrl_c_tx.closed() => {
            drop(listener);
            break
        }
    }
}

drop(close_rx);
close_tx.closed().await;
```

という感じで、簡潔に実装できました。( もしまずいところがあればぜひコメントかプルリクで教えてください @有識者 )