---
title: "ド・モルガンの法則を Rust で証明してみた"
emoji: "🦀"
type: "tech"
topics: ["rust", "型", "math", "logic"]
published: true
---

## はじめに

本記事は、[mutex Official Tech Blog](https://zenn.dev/p/mutex_inc) さんの

https://zenn.dev/mutex_inc/articles/de-morgan-ts

の ~~パクリ~~ Rust バージョンです ( [湯婆婆](https://qiita.com/tags/%e6%b9%af%e5%a9%86%e5%a9%86) みたいにいろんな言語でやってくれる人たちが出てきたら面白いなーという期待も込めて ) 。

コードは https://github.com/kanarus/rust_de_morgan にあるので必要に応じて参照してください。

以下、元記事の流れに沿って進めますが、nightly 前提なのと、筆者の好みにより元記事の略語命名は略される前の単語に変更してます。


## 型システムと命題論理の対応づけ

都合により

```rust
pub trait Variable: Copy + 'static {}
impl<T: Copy + 'static> Variable for T {}
```

という trait を用意しておきます。

### 含意

元記事では

```ts:TypeScript
type Imp<P, Q> = (_: P) => Q
```

となっていますが、Rust 的には、この後定義するものとの兼ね合いも考えつつきれいに見せようとすると地味に大変です。とりあえずの結論として以下のようにしました：

```rust:Rust
/*
    都合により Copy であってほしい
        type Imply<...> = Box<dyn FnOnce(P) -> Q + Copy>
    みたいなのは例によって + Copy できないので無理 ( あと rustc・rust-analyzer
    が認識する型は結局 `Box<dyn ...>` になってしまい見づらい )
*/
#[derive(Clone, Copy)]
pub struct Imply<P: Variable, Q: Variable>(
    /* Rc などでは Copy にできないのでなまぽ */
    *const dyn Fn(P) -> Q
);

/* そして #![feature(unboxed_closures, fn_traits)] */
impl<P: Variable, Q: Variable> FnOnce<(P,)> for Imply<P, Q> {
    type Output = Q;
    extern "rust-call" fn call_once(self, (p,): (P,)) -> Self::Output {
        (unsafe {self.0.as_ref()}.unwrap())(p)
    }
}

/*
    associated method で値を作らせるようにするとどうしても
    `move` を書いてもらわないといけない場合があり美しくないので
    同名の macro で TS 風に
*/
#[doc(hidden)]
impl<P: Variable, Q: Variable> Imply<P, Q> {
    pub fn that(predicate: impl FnOnce(P) -> Q + Copy + 'static) -> Self {
        Self(&move |p: P| predicate(p))
    }
}
#[macro_export]
/// ```Imply! {(p: P) => p}```
macro_rules! Imply {
    ( ($p:ident: $P:ty) => $q:expr ) => {
        Imply::that(move |$p: $P| $q)
    };
}
```

```rust:使用例
/// (P → (P → Q)) → (P → Q)
fn example_imply<P: Variable, Q: Variable>() ->
    Imply<Imply<P, Imply<P, Q>>, Imply<P, Q>>
{
    Imply! {(impPimpPQ: Imply<P, Imply<P, Q>>) =>
        Imply! {(p: P) =>
            impPimpPQ(p)(p)
        }
    }
}
```

どうでしょう...割と見やすいのでは (？)

### 連言

```rust:Rust
#[derive(Clone, Copy)]
pub struct And<P: Variable, Q: Variable> {
    left:  P,
    right: Q
}

impl<P: Variable, Q: Variable> And<P, Q> {
    /* Imply! との一貫性のために And! の方がいいかも (？) */
    pub fn introduce(left: P, right: Q) -> Self {
        Self { left, right }
    }

    pub fn eliminate_left(&self) -> P {
        self.left
    }

    pub fn eliminate_right(&self) -> Q {
        self.right
    }
}
```

```rust:使用例
/// P ∧ Q → Q ∧ P
fn example_and<P: Variable, Q: Variable>() -> Imply<And<P, Q>, And<Q, P>> {
    Imply! {(andPQ: And<P, Q>) =>
        And::introduce(
            andPQ.eliminate_right(),
            andPQ.eliminate_left()
        )
    }
}
```

### 選言

元記事では

```ts:TypeScript
class Or<P, Q> {
  private body: {tag: "left", payload: P} | {tag: "right", payload: Q}

  〜
}
```

とタグ付き union を使って実装してますが、Rust では enum がまさにタグ付き union そのものなので、こうなります：

```rust:Rust
#[derive(Clone, Copy)]
pub enum Or<P: Variable, Q: Variable> {
    Left(P),
    Right(Q)
}

impl<P: Variable, Q: Variable> Or<P, Q> {
    pub fn eliminate<R: Variable>(self, left: Imply<P, R>, right: Imply<Q, R>) -> R {
        match self {
            Self::Left(p)  => left(p),
            Self::Right(q) => right(q)
        }
    }
}
```

```rust:使用例
/// P ∨ Q → Q ∨ P
fn example_or<P: Variable, Q: Variable>() -> Imply<Or<P, Q>, Or<Q, P>> {
    Imply! {(orPQ: Or<P, Q>) =>
        orPQ.eliminate(
            Imply! {(p: P) => Or::Right(p)},
            Imply! {(q: Q) => Or::Left(q)}
        )
    }
}
```

### 矛盾

元記事では

```ts:TypeScript
type Bot = never
```

となっています。Rust では never 型を ( 型として ) 直接扱うことは今のところ unsatable なので、`#![feature(never_type)]` とした上で

```rust:Rust
pub type Bottom = !;
```

とします。

```rust:使用例
/// P ∨ ⊥ → P
fn example_bottom<P: Variable>() -> Imply<Or<P, Bottom>, P> {
    Imply! {(orPBottom: Or<P, Bottom>) =>
        orPBottom.eliminate(
            Imply! {(p: P) => p},
            Imply! {(b: Bottom) => b}
        )
    }
}
```

### 否定

```rust:Rust
pub type Not<P: Variable> = Imply<P, Bottom>;
```

ここで、`type` 宣言の type parameter の trait bound を機能させるために `lazy_type_alias` という incomplete feature がいります。

### 同値

```rust:Rust
pub type Equivalent<P: Variable, Q: Variable> = And<Imply<P, Q>, Imply<Q, P>>;
```


## ド・モルガンの法則の証明

### $\neg (P \lor Q) \leftrightarrow \neg P \land \neg Q$ の証明

```rust
/// ¬(P ∨ Q) ↔ ¬P ∧ ¬Q
fn __de_morgan_s_law_1__<P: Variable, Q: Variable>() ->
    Equivalent<Not<Or<P, Q>>, And<Not<P>, Not<Q>>>
{
    /// ¬(P ∨ Q) → ¬P ∧ ¬Q
    fn left<P: Variable, Q: Variable>() ->
        Imply<Not<Or<P, Q>>, And<Not<P>, Not<Q>>>
    {
        Imply! {(notOrPQ: Not<Or<P, Q>>) =>
            And::introduce(
                Imply! {(p: P) => notOrPQ(Or::Left(p))},
                Imply! {(q: Q) => notOrPQ(Or::Right(q))}
            )
        }
    }

    /// ¬(P ∨ Q) ← ¬P ∧ ¬Q
    fn right<P: Variable, Q: Variable>() ->
        Imply<And<Not<P>, Not<Q>>, Not<Or<P, Q>>>
    {
        Imply! {(andNotPNotQ: And<Not<P>, Not<Q>>) =>
            Imply! {(orPR: Or<P, Q>) =>
                orPR.eliminate(
                    Imply! {(p: P) => andNotPNotQ.eliminate_left()(p)},
                    Imply! {(q: Q) => andNotPNotQ.eliminate_right()(q)}
                )
            }
        }
    }

    And::introduce(left(), right())
}
```

### $\neg (P \land Q) \leftrightarrow \neg P \lor \neg Q$ の証明

```rust
/// ¬(P ∧ Q) ↔ ¬P ∨ ¬Q
fn __de_morgan_s_law_2__<P: Variable, Q: Variable>() ->
    Equivalent<Not<And<P, Q>>, Or<Not<P>, Not<Q>>>
{
    /// ¬(P ∧ Q) ← ¬P ∨ ¬Q
    fn right<P: Variable, Q: Variable>() ->
        Imply<Or<Not<P>, Not<Q>>, Not<And<P, Q>>>
    {
        Imply! {(orNotPNotQ: Or<Not<P>, Not<Q>>) =>
            orNotPNotQ.eliminate(
                Imply! {(notP: Not<P>) =>
                    Imply! {(andPQ: And<P, Q>) =>
                        notP(andPQ.eliminate_left())
                    }
                },
                Imply! {(notQ: Not<Q>) =>
                    Imply! {(andPQ: And<P, Q>) =>
                        notQ(andPQ.eliminate_right())
                    }
                }
            )
        }
    }

    /// ¬(P ∧ Q) → ¬P ∨ ¬Q
    fn left<P: Variable, Q: Variable>() ->
        Imply<Not<And<P, Q>>, Or<Not<P>, Not<Q>>>
    {
        /// P ∨ ¬P
        fn excluded_middle<P: Variable>() -> Or<P, Not<P>> {
            unreachable!("axios")
        }

        Imply! {(notAndPQ: Not<And<P, Q>>) =>
            excluded_middle::<P>().eliminate(
                Imply! {(p: P) =>
                    Or::Right(
                        Imply! {(q: Q) =>
                            notAndPQ(And::introduce(p, q))
                        }
                    )
                },
                Imply! {(notP: Not<P>) =>
                    Or::Left(notP)
                }
            )
        }
    }

    And::introduce(left(), right())
}
```

排中律は、元記事で

```ts:TypeScript
const excludedMiddle = <P>(): Or<P, Not<P>> =>
  undefined as any
```

となっているところ

```rust:Rust
/// P ∨ ¬P
fn excluded_middle<P: Variable>() -> Or<P, Not<P>> {
    unreachable!("axios")
}
```

としています ( 型システム的には、`unreachable!()` が never 型を返していて、never 型は任意の型にキャストされるのでコンパイルが通ります ) 。


### まとめ

```rust
fn main() {
    println!("De Morgan's laws are proven.");
}
```
