---
title: "Ohkami×Yew TODO - Rust で SPA を作る on Cloudflare Workers"
emoji: "🗒️"
type: "tech"
topics: ["ohkami", "yew", "rust", "cloudflare", "cloudflareworkers"]
published: true
---

https://zenn.dev/kanal/articles/ohkami-worker-urlshortener

に続いて、自作 web framework [Ohkami](https://github.com/kana-rus/ohkami) の記事です。
前回は template engine で UI を作っていたのに対して、今回は [Yew](https://yew.rs/ja/) を使い、Rust で SPA を作りました。

作ったもの：

https://ohkami-yew-todo.kanarus.workers.dev/

https://github.com/kana-rus/ohkami-yew-todo

## 注目ポイント

### Rust だけで SPA を作るメリット

僕のような Rustacean にとっては Rust だけ書けばいいという体験それ自体が大きなメリットなのですが、一般的にはやはり何も考えなくても API まわりの型 ( や util 類 ) を server・client 間で共有できることが挙げられると思います。

Ohkami×Yew TODO Demo では

```
.
├── api/
├── models/
├── ui/
│
├── front.rs
└── server.rs
```

というディレクトリ構成をとり、API の request / response body が入った `models` を全体で共有しています。

### Cloudflare Workers で Yew component を配信する構成

#### CSR vs SSR

現在の Yew には CSR と [SSR ( experimental )](https://yew.rs/ja/docs/advanced-topics/server-side-rendering) の２つの選択肢があります。後者の実用例としては

https://masahiro.me/posts/attempted-to-ssr-a-rust-based-yew-blog

が参考になりましたが、今回のアプリはページの概念がないものなのと、SSR するとしても開発時は CSR しないと hot reload できないのでデプロイ時だけ SSR に切り替えることになり、だいぶ面倒そうなので、無難に CSR による SPA を選びました。

:::details ビルドの遅さについて

上記記事で Rust ビルドの遅い問題が取り上げられていますが、僕の開発機ではそこまで気になっていなくて、アプリの内容の違い・PC の性能の違い・気になり方の個人差などいろんな要因がある話ですが、解決策の１つとして `sccache` と `mold` の導入を挙げておきます。参考記事：

https://qiita.com/neruneruna7/items/1569350742c686ceefbe#sccache-moldのインストール

:::

#### --assets と CORS

Cloudflare Workers で Ohkami API を立てつつ SPA ( index.html, wasm, js, css ) をどう配信するかですが、Ohkami の static directory serving はサーバーが動いている場所に serve したいディレクトリがある前提なので使えません。最悪全部 `include_bytes!` するかーと思いつつ `wrangler deploy --help` してみると

```
wrangler deploy [script]

🆙 Deploy your Worker to Cloudflare.

Positionals:
  script  The path to an entry point for your worker  [string]

Flags:
  -j, --experimental-json-config  Experimental: Support wrangler.json  [boolean]
  -c, --config                    Path to .toml configuration file  [string]
  -e, --env                       Environment to use for operations and .env files  [string]
  -h, --help                      Show help  [boolean]
  -v, --version                   Show version number  [boolean]

Options:
      --name                                       Name of the worker  [string]
      --no-bundle                                  Skip internal build steps and directly deploy Worker  [boolean] [default: false]
      --outdir                                     Output directory for the bundled worker  [string]
      --compatibility-date                         Date to use for compatibility checks  [string]
      --compatibility-flags, --compatibility-flag  Flags to use for compatibility checks  [array]
      --latest                                     Use the latest version of the worker runtime  [boolean] [default: false]
      --assets                                     Static assets to be served  [string]
      --site                                       Root folder of static assets for Workers Sites  [string]
      --site-include                               Array of .gitignore-style patterns that match file or directory names from the sites directory. Only matched items will be uploaded.  [array]
      --site-exclude                               Array of .gitignore-style patterns that match file or directory names from the sites directory. Matched items will not be uploaded.  [array]
      --var                                        A key-value pair to be injected into the script as a variable  [array]
      --define                                     A key-value pair to be substituted in the script  [array]
      --triggers, --schedule, --schedules          cron schedules to attach  [array]
      --routes, --route                            Routes to upload  [array]
      --jsx-factory                                The function that is called for each JSX element  [string]
      --jsx-fragment                               The function that is called for each JSX fragment  [string]
      --tsconfig                                   Path to a custom tsconfig.json file  [string]
      --minify                                     Minify the Worker  [boolean]
      --node-compat                                Enable Node.js compatibility  [boolean]
      --dry-run                                    Don't actually deploy  [boolean]
      --keep-vars                                  Stop wrangler from deleting vars that are not present in the wrangler.toml
                                                   By default Wrangler will remove all vars and replace them with those found in the wrangler.toml configuration.
                                                   If your development approach is to modify vars after deployment via the dashboard you may wish to set this flag.  [boolean] [default: false]
      --logpush                                    Send Trace Events from this worker to Workers Logpush.
                                                   This will not configure a corresponding Logpush job automatically.  [boolean]
      --upload-source-maps                         Include source maps when uploading this worker.  [boolean]
      --old-asset-ttl                              Expire old assets in given seconds rather than immediate deletion.  [number]
      --dispatch-namespace                         Name of a dispatch namespace to deploy the Worker to (Workers for Platforms)  [string]
```

となって `--assets` `Static assets to be served  [string]` なるオプションを見つけ、試してみると確かに `--assets ＜front のビルドディレクトリ＞` で ( まだ experimental option のようですが ) うまくいきました。

`wrangler dev` のほうにも同じオプションがあり、これでいいといえばいいのですが、これだと開発時に hot reload が効かないので、できれば開発時は `trunk serve` したいところです。そこで

```json:package.json
{
    ...

    "scripts": {
        "deploy": "trunk build --release && wrangler deploy --assets dist",
        "dev": "wrangler dev --env dev"
    },

    ...
}
```

```toml:wrangler.toml
build = { command = "cargo install -q worker-build && worker-build --release" }

...

[env.dev]
build = { command = "cargo install -q worker-build && worker-build --dev" }
# Then, run `trunk serve --watch src/ui --open` in another terminal window
```

```rust:src/server.rs
...

#[ohkami::worker]
async fn my_worker() -> Ohkami {
    ...

    let fangs = {
        #[cfg(debug_assertions)]
        ohkami::builtin::fang::CORS::new("http://127.0.0.1:8080")
    };

    Ohkami::with(fangs, (
        /* in production, `./dist` is served by `--assets dist` of `deploy` script in package.json */

        "/signup"
            .POST(signup),

        "/api".By(Ohkami::with(jwt::fang(), (
            "/cards"
                .GET(list_cards)
                .POST(create_card),
            "/cards/:id"
                .PUT(update_card)
                .DELETE(delete_card),
        ))),
    ))
}
```

とすることで、開発時には miniflare と別で `trunk serve` して hot reload を効かせ、デプロイ時は `--assets dist` で static directory serving するという形を作りました。

### Tailwind CSS

今回スタイリングに Tailwind CSS を使ったのですが、地味にちょっと詰まったので trunk での導入手順をまとめておきます。

まず、[README](https://github.com/kana-rus/ohkami-yew-todo/blob/main/README.md) の Prerequisites で

> - `tailwindcss` CLI ( see https://tailwindcss.com/blog/standalone-cli )

としているように `tailwindcss` という standalone CLI を用意します。

次に

```js:＜好きな名前＞.js
module.exports = {
    content: [
        "./index.html",
        "./src/**/*.{rs,html,css}",
    ],
    theme: {},
    variants: {},
    plugins: [],
};
```

```css:＜好きな名前＞.css
@config "＜上の js ファイルのパス＞"
@tailwind base;
@tailwind components;
@tailwind utilities;
```

を作ります ( 内容は自由に編集してください ) 。
( Ohkami×Yew TODO Demo ではプロジェクトルートに `tailwind.config.js` と `tailwind.css` を作りました )

最後に index.html の `<head></head>` 内に

```html
    <link data-trunk rel="tailwind-css" href="＜後者の css のパス＞"/>
```

というタグを追加すると、trunk がビルド時に tailwind CLI を使って `class="..."` をいい感じに処理してくれるようになります。

### 自動 Bindings

Ohkami が前回記事から大きく進化した点のひとつに `#[bindings]` という attribute があり、

```rust:src/server.rs
#[ohkami::bindings]
struct Bindings;
```

とするだけで自動的にプロジェクト内の `wrangler.toml` の Binding 類を読んで struct のフィールドに追加し、`FromRequest` の実装を導出して handler の引数に使えるようにしてくれます。
`vars` についてはフィールドだけでなく同名の static method も生やしてくれるので `Bindings::VAR_NAME()` のようにも値を参照できます。
また、`#[bindings(env_name)]` で参照する env を切り替えられます ( デフォルトで default ( top-level ) env を参照します ) 。

( Hono の

```ts
type Bindings = {
  MY_BUCKET: R2Bucket
  USERNAME: string
  PASSWORD: string
}

const app = new Hono<{ Bindings: Bindings }>()
```

みたいなコードを自動的に生成してくれるイメージです。何でもできる Rust の proc macro ならではのアプローチで気に入っています )

使用例：

```rust:src/api/jwt.rs
use super::Bindings;
use ohkami::serde::{Serialize, Deserialize};
use ohkami::builtin::{fang::JWT, item::JWTToken};
use ohkami::utils::unix_timestamp;


#[derive(Serialize, Deserialize)]
pub struct JWTPayload {
    pub user_id: String,
    iat: u64,
}

pub fn fang() -> JWT<JWTPayload> {
    JWT::default(Bindings::JWT_SECRET_KEY())
}

pub fn new_token_for(user_id: String) -> JWTToken {
    self::fang().issue(JWTPayload { user_id, iat: unix_timestamp() })
}
```

```rust:src/api/mod.rs
...

#[worker::send]
pub async fn signup(
    b: Bindings,
) -> Result<SignupResponse, ServerError> {
    let user_id = WorkerGlobalScope::unchecked_from_js(js_sys::global().into())
        .crypto().unwrap()
        .random_uuid();

    b.DB.prepare("INSERT INTO users (id) VALUES (?)")
        .bind(&[(&user_id).into()])?
        .run().await?;

    Ok(SignupResponse {
        token: jwt::new_token_for(user_id)
    })
}

...
```

<br>

ちなみに `trunk serve` は `--watch src/ui` のように watch するディレクトリを指定できるのですが、`worker-build` にはそういうオプションがなく `src/` 以下全ファイルの変更に反応してしまうので、`ui` を変更したときもサーバーが再起動してしまいます。
これくらいの規模ならそれでもまあいいかと思ってそのままにしているのですが、気になる場合は workspace にして `ui` と `models` と `server` みたいな member に分けるといいと思います。

:::details なぜそれをここで言うのだ

実は現状、cargo workspace 内で呼ばれた proc macro がファイルシステムにアクセスするとき、相対パスは **workspace root からのパス** になるため、例えば

```
root
│
├── member_server/
│   ├── src/
│   ├── Cargo.toml
│   └── wrangler.toml
│
├── member_ui/
│   └── src/
│   └── Cargo.toml
│
└── Cargo.roml
```

みたいな workspace の `member_server` 内で `#[bindings]` が呼ばれたとしても、`#[bindings]` 自身は自分がそこで呼ばれたことを知らないので、まず root からそれを探しに行く必要があります。

そこで、`#[bindings]` はプロジェクトルートに `wrangler.toml` がない場合、プロジェクトルートの `Cargo.toml` の `workspace.members` を読み、それらを全探索して `wrangler.toml` を持っている member を探し出してそこで呼ばれたと認識し、その `wrangler.toml` を使うという挙動をします。

( `wrangler.toml` を持っている member が複数ある場合はエラーにするようにしてあるので、該当する member はあれば一意に定まります )

何が言いたいかというと、`#[bindings]` はちゃんと workspace に対応してるので、workspace にしたい場合も安心して使ってくださいということです。。。

:::

## まとめ

お読みいただきありがとうございます。

このデモを作ったあと、使い回せる要素をまとめてテンプレート化したものを https://github.com/kana-rus/ohkami-templates/tree/main/worker_yew_spa に置いておいたので、Rust だけ書いて SPA を作りたい方はぜひ使ってみてください！
