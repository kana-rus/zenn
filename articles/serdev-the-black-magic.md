---
title: "SerdeV - Serde with Validation の紹介と黒魔術解説"
emoji: "🧙🏻‍♀️"
type: "tech"
topics: ["Rust", "Serde", "validation"]
published: true
---

Rust で

```rust
#[derive(Deserialize, Debug)]
#[serde(validate = "Self::validate")]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn validate(&self) -> Result<(), impl std::fmt::Display> {
        if self.x < 0 || self.y < 0 {
            return Err("x and y must not be negative")
        }
        Ok(())
    }
}

fn main() {
    let point = serde_json::from_str::<Point>(r#"
        { "x" : 1, "y" : 2 }
    "#).unwrap();

    // Prints point = Point { x: 1, y: 2 }
    println!("point = {point:?}");

    let error = serde_json::from_str::<Point>(r#"
        { "x" : -10, "y" : 2 }
    "#).unwrap_err();

    // Prints error = x and y must not be negative
    println!("error = {error}");
}
```
*( [README](https://github.com/ohkami-rs/serdev/blob/main/README.md) より引用 )*

みたいなこと ( validation on deserializing without boilerplate ) できないのかなーということに数年前から興味があり、その後 Rust 力をつけた結果できそうな気がしてきたので [SerdeV](https://crates.io/crates/serdev) というクレートを開発してみたらできました。紹介を兼ねて研究レポート的な意味で中身の解説を置いておきます。


## 前提

- Serde を fork して改造するのは嫌
  - それでよければ割と簡単にできそうだが、その後本家の issue や PR に追従するのが面倒
  - あくまで Serde のラッパーとして、proc macro の力でなんとか実現する

- 現実的に需要があるかはとりあえず気にしない
  - [Parse, dont' validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate) という有名 (？) な標語の通り、パースした時点で型システムによって valid であることが保証されるような型設計ができれば、明示的なバリデーションというものはいらない
    - そして Rust は ( 多くの場合 ) それができる言語である
  - しかし、時に Serde の中で気軽にバリデーションを入れたい場面も存在する気はする
    - 特に Web 開発で (？)


## 準備

### 準備１：ディレクトリ構成

例によって proc macro crate は proc macro しか export できなかったりするので、

```
.
├── serdev
│   └── src
└── serdev_derive
    └── src
        └── internal
```

として、`serdev_derive` で `Serialize`, `Deserialize` の derive 実装を提供し、`serdev` で

```rust:lib.rs
pub use serdev_derive::{Serialize/* macro */, Deserialize/* macro */};
pub use ::serde::ser::{self, Serialize/* trait */, Serializer};
pub use ::serde::de::{self, Deserialize/* trait */, Deserializer};
```

とします。

### 準備２：Serialize

SerdeV は ( `validate` 以外において ) Serde と _100% compatible_ を謳っているので、普通に

```rust
use serdev::Serialize;

#[derive(Serialize)]
struct Point {
    x: i32,
    y: i32,
}
```

のように書ける必要があります。そこで `#[derive(serde::Deserialize)] の impl 部分` みたいなものを生成すればいいので、

```rust:serdev/src/lib.rs
#[doc(hidden)]
pub mod __private__ {
    pub use ::serde;
}
```

として derive 実装で

```rust
quote! {
    #[derive(::serdev::__private__::serde::Serialize)]
    #[serde(crate = "::serdev::__private__::serde")]
    #input
}
```

を返すのが素朴なアイデアです。

:::message
`serde::Serialize` や `serde::Deserialize` を impl する部分を自前でちゃんと生成しようとすると `#[serde(borrow)]` ( field ) や `#[serde(bound = "...")]` ( container, field ) を見るなどする必要があり、気づけば serde_derive の再実装が始まってしまうので、fork を避けた意味がなくなります。そこは本家の derive macro に丸投げしましょう
:::

が、これだけではうまくいきません。というのも、`#[derive(〜)] #input` というものは `#input` を受け取って **#input そのもの + derive macroが返したもの** を返す

:::message
本来、これによって derive macro ではもとの struct や enum は保存されるので実装者は単に `impl 〜` 部分を生成して返せばいいようになっています
:::

ので、上記の実装だと

1. もとの構造体は保存される

2. `derive(serdev::Serialize)` が

    ```rust
    #[derive(::serdev::__private__::serde::Serialize)]
    #[serde(crate = "::serdev::__private__::serde")]
    #input
    ```

    を返す

3. それが本家の `derive(serde::Serialize)` によって `#input` そのもの + `Serialize` impl に展開されるので、最終的に

    ```rust
    struct Point {
        x: i32,
        y: i32,
    }

    struct Point {
        x: i32,
        y: i32,
    }

    /* Serialize impl */
    ```

    となってコンパイルエラー

ということになります。これの解決にあたって黒魔術が必要になります。これです：

https://zenn.dev/kanal/articles/rust-use-external-derive-in-my-derive

:::message
この記事のときには気づいていなかったのですが１つの attribute が影響を及ぼせるのは１つの item までのようなので、明示的にパース処理を書かずいきなり

```rust:serdev_derive/src/lib.rs
#[proc_macro_attribute]
pub fn consume(_: proc_macro::TokenStream, _: proc_macro::TokenStream) -> proc_macro::TokenStream {
    proc_macro::TokenStream::new()
}
```

で問題ないです
:::

つまり、この `consume` という attribute を `serdev::__private__` から export しておき、

```rust
quote! {
    #[derive(::serdev::__private__::serde::Serialize)]
    #[serde(crate = "::serdev::__private__::serde")]
    #[::serdev::__private__::consume]
    #input
}
```

を返すようにするとうまくいきます。

ただし、このままだと、

```rust
use serdev::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
#[serde(validate = "...")]
struct Point {
    x: i32,
    y: i32,
}
```

とした際に、( 本家の ) `derive(Serialize)` にもそのまま `#[serde(validate = "...")]` が渡されてしまい「そんなの知らないが？」と言われてしまうので、

https://github.com/ohkami-rs/serdev/blob/1f6bdf9a005e1348d8ae7eea79310b9d1827c6d1/serdev_derive/src/internal.rs#L12-L22

のように `#[serde(validate ...)]` があったら外してから渡してあげます。


## 本題：Deserialize

長々と準備してきましたが、実はあとは [`#[serde(try_from = "FromType")]`](https://serde.rs/container-attrs.html#try_from) を使えばいけることに気づけばほぼ*やるだけ* です：

1. 対象の構造体 ( target とします ) の名前だけ変えたもの ( proxy とします ) を用意し、`#[derive(serde::Deseirlize)]` をつける

2. proxy のフィールドを target に移し替え、バリデーションを行ってから返すような `impl TryFrom<{proxy}> for {target} ...` を用意する

3. 本家の `derive(Deserialize)` に `#[serde(try_from = "{proxy}")]` をつけて target に対する `Deserialize` 実装を導出する

`#[serde(validate ...)]` を抜き取る実装、target の generics の扱い、 `TryFrom::Error` をどうするか ( 関数の path を渡されるだけではその error の型を認識できない ) など細かい課題はありますが、ここまで理解できた方なら

https://github.com/ohkami-rs/serdev

に行って眺めれば分かると思うので省略します。

１つだけ非自明かもしれない点があるので補足します。1. の proxy を作る処理

https://github.com/ohkami-rs/serdev/blob/1f6bdf9a005e1348d8ae7eea79310b9d1827c6d1/serdev_derive/src/internal/target.rs#L65-L87

で `ident` を変更したあと、なにやら `attrs` を filter しているのですが、これは proxy に `#[serde(〜)]` 以外の attribute を残しておくと、それが他の derive macro 由来のものだった場合に

```rust:serde_derive/src/internal.rs
#[derive(::serdev::__private__::serde::Deserialize)]
#[serde(crate = "::serdev::__private__::serde")]
#[allow(non_camel_case_types)]
#proxy
```

の部分で ( その derive がないので ) コンパイルエラーになるからです。例：

https://github.com/ohkami-rs/serdev/blob/1f6bdf9a005e1348d8ae7eea79310b9d1827c6d1/serdev/examples/validator.rs

```rust
// filter しない場合
#[derive(::serdev::__private__::serde::Deserialize)]
#[serde(crate = "::serdev::__private__::serde")]
#[allow(non_camel_case_types)]
struct serdev_proxy_SignupData {
    #[validate(email)]  // <-- ここでコンパイルエラー
    mail: String,
    〜
```


## おわりに

[Parse, dont' validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate) を踏まえて [Serde with Validation](https://github.com/ohkami-rs/serdev) についてどう思うか、いろんな人の意見を知りたい気持ちがあるので、何か思うことがある方はぜひこの記事のコメントか SNS にでも書いてください 👀　SerdeV を気に入った方は star をつけていただけると嬉しいです。
