---
title: "ド・モルガンの法則を Rust で証明してみた - const 編"
emoji: "🌊"
type: "tech"
topics: ["rust", "型", "math", "logic"]
published: true
---

[ド・モルガンの法則を Rust で証明してみた](https://zenn.dev/kanal/articles/de-morgan-rust) の続編というか補足です。元記事 (

https://zenn.dev/mutex_inc/articles/de-morgan-ts

) の冒頭に挙げられている、実行結果をもって証明となるコード：

```ts:TypeScript
const booleans = [true, false]

booleans.forEach(
  p =>
    booleans.forEach(
      q => {
        console.assert(!(p || q) === !p && !q)
        console.assert(!(p && q) === !p || !q)
      }
    )
)
```

について、Rust なら const 文脈での計算によってこの方針でコンパイル時に証明できることを示します。

## 準備

### static assert

とりあえず、const 文脈で値が分かる bool 値について、それが `true` であることをコンパイル時に検証する手段がいります。

外部クレートを使ってもいいですが、簡単に作れるのでここでは自作します：

```rust
macro_rules! static_assert {
    ($condition:expr) => {
        {
            const _: bool = $condition;
            const _: [(); $condition as usize] = [(); true as usize];
        }
    };
}
```

### 「あらゆる２つの bool の組み合わせ」について何かすることを静的に保証する

冒頭のコードの

```ts:TypeScript
const booleans = [true, false]
```

について、これは当然正しいのですが、bool 型の値を手動で列挙していて、それが bool 型のすべての値であることは検証されていないのが少し気になります。

Rust では `match` で網羅性を静的に検証できるので、「静的にチェック済みの bool のすべての値について何かする処理」みたいなものを作れます。

ここでは、せっかくなので直接「あらゆる２つの bool の組み合わせについて何かする処理」を作ります：

```rust
macro_rules! for_all_cominations_of_two_bools {
    (($P:ident, $Q:ident) => $proc:expr) => {
        for_all_cominations_of_two_bools! {
            @($P, $Q) in checked[
                (true,  true ),
                (true,  false),
                (false, true ),
                (false, false)
            ] {
                $proc
            }
        }
    };
    (@($P:ident, $Q:ident) in checked[$( ($p:literal, $q:literal) ),*] $proc:expr) => {
        fn __assert_exausted__(p: bool, q: bool) {
            match (p, q) {$(
                ($p, $q) => {
                    const $P: bool = $p;
                    const $Q: bool = $q;
                    $proc
                },
            )*}
        }
    };
}
```

２つ目の arm によって生成される `match (p, q) { 〜 }` の網羅性チェックのおかげで、１つ目の arm の `checked[ 〜 ]` 内ですべての組み合わせを挙げていないとコンパイルエラーになります。

## ド・モルガンの法則の証明

```rust
const fn main() {
    for_all_cominations_of_two_bools! {(P, Q) => {
        static_assert!({!(P || Q)} == {(!P) && (!Q)});
        static_assert!({!(P && Q)} == {(!P) || (!Q)});
    }}
}
```

( `!` と `==` と `||`, `&&` の優先順位を意識したくないので冗長にかっこをつけまくっています )

## まとめ

この記事のコードは

https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=fcae945eff0770ca41f5fef78f56a754

に置いてあるので、気になった方は `static_assert!` の中をいじってコンパイルエラーを吐かせるなどして遊んで、これでド・モアブルの定理がコンパイル時に証明されていることを確かめてみてください。
