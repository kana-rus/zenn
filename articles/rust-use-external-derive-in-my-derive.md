---
title: "外部クレートの derive macro を自分の derive macro 内で使う"
emoji: "🤖"
type: "tech"
topics: ["Rust", "derive", "macro"]
published: true
---

## やりたいこと
外部クレートの `#[derive(Sample)]` を自分の `#[derive(MyDerive)]` の実装内で使いたい。


## 問題
単に

```rust:lib.rs
  mod internals;
  use proc_macro::TokenStream;
  
  #[proc_macro_derive(MyDerive)]
  pub fn derive_myderive(stream: TokenStream) -> TokenStream {
      internals::derive_myderive(stream.into())
          .unwrap_or_else(|err| err.into_compile_error())
          .into()
  }
```
```rust:internals.rs
  use proc_macro2::TokenStream;
  use quote::quote;
  use syn::{Error, ItemStruct};
  
  pub(super) fn derive_myderive(input: TokenStream) -> Result<TokenStream, Error> {
      let ItemStruct { ident .. }
          = syn::parse2(input.clone())?;
  
      Ok(quote!{
          impl my_crate::MyDerive for #ident {}
  
          #[derive(external_crate::Sample)]
          #input
      })
  }
```

みたいにすると、

```rust:user.rs
  #[derive(MyDerive)]
  struct User {
      id:   u64,
      name: String,
  }
```

の展開結果が

```rust:expanded.rs
  struct User {
      id:   u64,
      name: String,
  }
  
  impl my_crate::MyDerive for User {}
  
  struct User {
      id:   u64,
      name: String,
  }
  
  impl external_crate::Sample  for User {
      // ...
  }
```

となりアウト


## 準備
今から自分は行儀が悪いコードを書こうとしているのだということを心に留めつつ、**attribute 系のマクロ ( derive, attribute ) は上から ( 外側から ) 展開される** という仕様を突いて展開後の struct を１つ消す。

具体的には、まず以下のような attribute macro を用意する：

```rust:lib.rs
  #[proc_macro_attribute]
  pub fn consume_struct(_: TokenStream, derived_struct: TokenStream) -> TokenStream {
      internals::consume_struct(derived_struct.into())
          .unwrap_or_else(|err| err.into_compile_error())
          .into()
  }
```

```rust:internals.rs
  pub(super) fn consume_struct(serde_derived_struct: TokenStream) -> Result<TokenStream, Error> {
      let _: ItemStruct = syn::parse2(serde_derived_struct)?;
      Ok(TokenStream::new())
  }
```

これは struct を１つ受け取って空の `TokenStream` を返すので、例えば

```rust:user.rs
  #[consume_struct]
  struct User {
      id:   u64,
      name: String,
  }
```
は

```rust:expanded.rs
```
となり、`#[consume_struct]` された struct が消去されている。

次がポイントで、attribute 系のマクロは**上から**順に処理、展開される。例えば

```rust:Taro.rs
  #[consume_struct]
  #[derive(MyDerive)]
  struct Taro;
```
```rust:Jiro.rs
  #[derive(MyDerive)]
  #[consume_struct]
  struct Jiro;
```
というコードを考えると、前者はまず `#[consume_struct]` が処理され、このときに `#[derive(MyDerive)] struct Taro;` の部分が消費されて空の `TokenStream` が返されるので

```rust:Taro.expanded.rs
```
となる。一方、後者はまず `#[derive(MyDerive)]` が処理されて

```rust:Jiro.inter_expanding.rs
  #[consume_struct]
  struct Jiro;

  impl my_crate::MyDerive for Jiro {}
```
となって、次に `#[consume_struct]` が処理されるので、最終的に

```rust:Jiro.expanded.rs
  impl my_crate::MyDerive for Jiro {}
```
となる ( よってコンパイルエラー ) 。


## 解決

```diff rust:internals.rs
  pub(super) fn derive_myderive(input: TokenStream) -> Result<TokenStream, Error> {
      let ItemStruct { ident .. }
          = syn::parse2(input.clone())?;
  
      Ok(quote!{
          impl my_crate::MyDerive for #ident {}
  
          #[derive(external_crate::Sample)]
+         #[consume_struct]
          #input
      })
  }
```
とすればよい。

これによって

```rust:user.rs
  #[derive(MyDerive)]
  struct User {
      id:   u64,
      name: String,
  }
```
の展開結果は

```diff rust:expanded.rs
  struct User {
      id:   u64,
      name: String,
  }
  
  impl my_crate::MyDerive for User {}

- struct User {
-     id:   u64,
-     name: String,
- }

  impl external_crate::Sample  for User {
      // ...
  }
```
となって、ちょうどやりたかったことが実現された。


## 補足
この展開は `#[derive(MyDerive)]` だけで行われるので、他の derive があったりしても影響はない。例えば

```rust:user.rs
  #[derive(A, MyDerive, B)]
  struct User {
      id:   u64,
      name: String,
  }
```
は、まず `A` で

```rust:user.inter_expanding_a.rs
  #[derive(MyDerive, B)]
  struct User {
      id:   u64,
      name: String,
  }

  impl A for User {}
```
となり、次に `MyDerive` で

```rust:user.inter_expanding_myderive.rs
  #[derive(B)]
  struct User {
      id:   u64,
      name: String,
  }

  impl my_crate::MyDerive for User {}

  impl external_crate::Sample  for User {
      // ...
  }

  impl A for User {}
```
となり、最後に `B` で

```rust:user.expanded.rs
  struct User {
      id:   u64,
      name: String,
  }

  impl B for User {}

  impl my_crate::MyDerive for User {}

  impl external_crate::Sample  for User {
      // ...
  }

  impl A for User {}
```
となる。