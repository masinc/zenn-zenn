---
title: "Rustのdbg!マクロについて"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

# `dbg!` マクロとは

Rust1.32.0 より stable に追加されたマクロです。

https://doc.rust-lang.org/std/macro.dbg.html

その名の通り、デバッグに便利なマクロです。
式と結果が標準エラーに出力されます。
`println!`などのマクロと違ってそのまま式が出力されます。出力後に値を返してくれるのが大きな違いです。

# 利用方法

`dbg!`マクロにて出力する値は`Debug`トレイトを実装している必要があります。

@[gist](https://gist.github.com/rust-play/4aa01bcb1cd8490d6b52d91402a67e64)

```
[src/main.rs:5] "rust" = "rust"
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4aa01bcb1cd8490d6b52d91402a67e64)

式の結果が`Debug`トレイトを実装していればよく式をそのまま渡すこともできます。

@[gist](https://gist.github.com/rust-play/9f6d3b5f0d93ffa8895a462a5f13ff4b)

```
[src/main.rs:2] 1 + 2 + 3 + 4 = 10
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=9f6d3b5f0d93ffa8895a462a5f13ff4b)

式の結果が`Copy`トレイトを実装していない場合、値渡しをした場合、元の所有権は失われます。
`Copy`トレイトを実装してなくても参照渡しをすれば所有権は保持されます。

@[gist](https://gist.github.com/rust-play/7d103eb4ac9123bbb76ee4ffc3a5b63c)

```
[src/main.rs:7] S(1 + 2) = S(
    3,
)
[src/main.rs:8] a.0 + 1 = 4
[src/main.rs:10] &a.0 + 1 = 4
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7d103eb4ac9123bbb76ee4ffc3a5b63c)

`Copy`トレイトを実装していれば、元の所有権は残ったままとなります。

@[gist](https://gist.github.com/rust-play/5572a97f17d0d4d8bd732e47e316b727)

```
[src/main.rs:7] S(1 + 2) = S(
    3,
)
[src/main.rs:8] a.0 + 1 = 4
[src/main.rs:9] a = S(
    3,
)
[src/main.rs:10] &a.0 + 1 = 4
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=5572a97f17d0d4d8bd732e47e316b727)

実用的な例としては、イテレータの途中経過をデバッグするとき等に利用できます。

@[gist](https://gist.github.com/rust-play/ddfbb447cf37b504f423c69c5dc5884c)

```
[src/main.rs:5] x + 9 = 9
[src/main.rs:6] x * 7 = 63
[src/main.rs:7] acc + x = 63
[src/main.rs:5] x + 9 = 10
[src/main.rs:6] x * 7 = 70
[src/main.rs:7] acc + x = 133
[src/main.rs:5] x + 9 = 11
[src/main.rs:6] x * 7 = 77
[src/main.rs:7] acc + x = 210
[src/main.rs:5] x + 9 = 12
[src/main.rs:6] x * 7 = 84
[src/main.rs:7] acc + x = 294
[src/main.rs:5] x + 9 = 13
[src/main.rs:6] x * 7 = 91
[src/main.rs:7] acc + x = 385
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ddfbb447cf37b504f423c69c5dc5884c)

引数に何もわたさないと`dbg!`マクロ記述のファイル名と行が出力されます。

@[gist](https://gist.github.com/rust-play/ebf356dddc8763b0eb794146735417aa)

```rust
[src/main.rs:2]
```

[Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ebf356dddc8763b0eb794146735417aa)

複数の式を渡すこともできます。戻り値はタプルで帰ってきます。

@[gist](https://gist.github.com/rust-play/f82734a862ef663dd5e64f0166345a95)

```rust
[src/main.rs:3] 1 + 2 = 3
[src/main.rs:3] 3 + 4 = 7
[src/main.rs:3] 5 + 6 = 11
[src/main.rs:3] 7 + 8 = 15
[src/main.rs:4] std::any::type_name_of_val(&a) = "(i32, i32, i32, i32)"
```

[Rust Playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=f82734a862ef663dd5e64f0166345a95)

# `dbg!`マクロ利用で警告/エラーを表示 - Clippy との連携

`dbg!`マクロは release ビルド時でも出力されます。
また、バージョン管理システムでの `dbg!`マクロ利用をエラーにしたい等があります。

これらは Rust の lint ツールである Clippy を利用することで対策できます。

https://github.com/rust-lang/rust-clippy

clippy をインストール後、`cargo check`コマンドのように
`cargo clippy`コマンドを利用できます。
`dbg!`マクロで警告/エラーを表示する場合、
`clippy::dbg_macro`のレベルを変更することで表示できます。

https://rust-lang.github.io/rust-clippy/master/index.html#dbg_macro

設定方法には大きく分けて 2 通りあり、Rust の属性で設定する方法と、cargo clippy のコマンドラインオプションで指定する方法があります。

1. 属性での指定

   `warn`属性で警告が発生、`deny`属性ではエラーが発生します。

   @[gist](https://gist.github.com/rust-play/940e7dfd05a9f4d90dacfedc0a882cf7)

   `cargo clippy`の結果(一部抜粋)

   - 警告時

     ```
     warning: `dbg!` macro is intended as a debugging tool
     --> src/main.rs:4:5
     |
     4 |     dbg!();
     |     ^^^^^^
     |
     ```

   - エラー時

     ```
     error: `dbg!` macro is intended as a debugging tool
     --> src/main.rs:4:5
     |
     4 |     dbg!();
     |     ^^^^^^
     |
     ```

   [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=940e7dfd05a9f4d90dacfedc0a882cf7)

   :::message
   こちらは属性を追加するだけで楽ですがファイルごとのチェックになるため、
   全ソースファイルに属性付与する必要があります。
   規模が大きい場合はコマンドラインオプション指定が楽です。
   :::

2. コマンドラインオプションでの指定

   Git の pre-commit や CI 等を使いチェックをするならコマンドラインで指定するのが便利です。

   - 警告表示

     ```bash
     cargo clippy -- -W clippy::dbg_macro
     ```

   - エラー表示

     ```bash
     cargo clippy -- -D clippy::dbg_macro
     ```

   :::message
   注意点としてローカル等で以前に`cargo clippy`を実行している場合、
   コマンドラインオプションが反映されないため事前に`cargo clean`を実行する必要があります。
   :::

   Git の pre-commit の一例としては以下のような内容になります。

   ```bash
   #!/bin/bash -eu

   cargo clean && cargo clippy -- -D clippy::dbg_macro
   ```

# rust-analyzer との連携

`dbg!`マクロは LSP である rust-analyzer を利用することで便利な機能があります。
rust-analyzer は様々なエディタで利用できますが、本記事では VSCode を利用します。

## `dbg!` マクロの挿入

Magic Completions と呼ばれる機能を使うことでお手軽に`dbg!`マクロを挿入できます。
https://rust-analyzer.github.io/manual.html#magic-completions

1. expr.dbg

   ```rust
   let x = 1;
   x.dbg💡
   ```

   💡 位置でカーソルがある状態で`Tab`キーを押すと以下に変換してくれます。

   ```rust
   let x = 1;
   dbg!(x)
   ```

2. expr.dbgr

   ```rust
   let x = 1;
   x.dbgr💡
   ```

   💡 位置でカーソルがある状態で`Tab`キーを押すと以下に変換してくれます。

   ```rust
   let x = 1;
   dbg!(&x)
   ```

## `dbg!` マクロの削除

`dbg!` マクロにカーソルがある状態で`Ctrl + .`またはエディタの左側に表示される 💡 アイコンをクリックすると、
`Remove dbg!()`メニューが表示され、それをクリックすると`dbg!`マクロを削除します。

```rust
    💡dbg!(1)
```

`Remove dbg!()`を実行すると`dbg!`マクロを削除してくれます。

```rust
    1
```

## clippy の利用

rust-analyzer はデフォルトでは保存時に`cargo check`を実行をしますが、設定を変更することにより、`cargo clippy`を実行できます。

VSCode の settings.json の設定例。

```json5
{
  // 保存時のコマンドをcargo checkから cargo clippyに変更します。
  "rust-analyzer.checkOnSave.command": "clippy",
  // 保存時のコマンドの引数を指定。ここではdbg!マクロ利用時に警告が出るようにします。
  "rust-analyzer.checkOnSave.extraArgs": ["--", "-W", "clippy::dbg-macro"],
}
```
