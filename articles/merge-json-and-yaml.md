---
title: "JSON や YAML ファイルのマージをする"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["json", "yaml"]
published: true
---

JSON や YAML でマージをする方法を調べたところ、 `jq` や `yq` を利用することで手軽に行なうことができます。

本記事では `jq` や `yq` の基本的に利用方法については、知っているものとして記述します。

# `jq` を利用したマージ

`jq` は入出力ともに JSON のみに対応しています。

https://stedolan.github.io/jq/

```shellsession
$ jq --version
jq-1.6
```

以下のファイルをマージしてみます。

```json:a.json
{
  "a": 1
}
```

```json:b.json
{
  "a": 2,
  "b": 2
}
```

```json:c.json
{
  "a": 3,
  "c": 3
}
```

- コマンド実行

  ```shellsession
  $ jq -s 'add' a.json b.json c.json
  ```

- 結果

  ```json
  {
    "a": 3,
    "b": 2,
    "c": 3
  }
  ```

`-s` オプションはすべての入力を配列として扱うオプションです。\
例として下記のコマンドを実行します。

`add` は配列を入力として取り、すべての要素をマージしたものを出力します。

# `yq` を利用したマージ

`yq` の入出力は JSON や YAML のほかに、XML や Properties などに対応しています。

https://github.com/mikefarah/yq/

https://mikefarah.gitbook.io/yq/

```shellsession
$ yq --version
yq (https://github.com/mikefarah/yq/) version 4.27.5
```

:::message

`yq` は記法がバージョンによって変化することがあります。

:::

以下のファイルをマージしてみます。

```json:a.json
{
  "a": 1
}
```

```yaml:b.yaml
a: 2
b: 2
```

```yaml:c.yaml
a: 3
c: 3
```

- コマンド実行

  ```shellsession
  $ yq eval-all -P '. as $item ireduce ({}; . * $item)' a.json b.yaml c.yaml
  ```

  本来ファイルフォーマットを指定する必要がありますが、yq のデフォルトフォーマットは YAML です。\
  JSON は YAML のサブセットであるため、入力に JSON と YAML を混在しても動作します。

- 結果

  ```yaml
  a: 3
  b: 2
  c: 3
  ```

`eval-all` または、 `ea` は複数のファイルを処理できるオプションです。1 つのファイルでよい場合は `eval`, `e`
または、未指定とします。

`-P` は `--prettyPrint` オプションの省略形で、綺麗に出力されます。これを利用しない場合、以下のように出力されます。

```yaml
"a": 3
b: 2
c: 3
```

`. as $item ireduce ({}; . * $item)` は単純なマージの利用時には変更する必要はありませんが、以下に説明します。

- `. as $item` の `.` は現在のファイルトップレベルを表しています。 `$item` に設定します。`$v` 等でも問題ありません。
- `ireduce` は、プログラム言語での `reduce` や `fold` 関数に近いものです。
- `{}` は `ireduce` の第一引数で、初期値を指定します。 ここでは、 JSON 用語だと空オブジェクト、\
  YAML 用語だと空の Map オブジェクトを表しています。
- `{}` の後の `;` は区切り文字です。
- `. * $item` は `ireduce` の第二引数で、 上記の関数での `.` は `acc` 引数、`$item` は `x`
  引数に近いものとなります。 `*` 演算子はオブジェクトのマージができます。

`yq` のマージの詳細については下記に記載されています。

https://mikefarah.gitbook.io/yq/operators/multiply-merge

また、今回は JSON と YAML の混在したマージを行っていますが、 これは JSON が YAML のサブセットであるためできます。\
例えば XML 等を混在する場合は、プロセス置換などを利用して、 XML を YAML または、 JSON に変換してから引数に与える必要があります。\
例として下記のようなコマンドを実行する必要があります。

```shellsession
$ yq eval-all -P '. as $item ireduce ({}; . * $item)' a.json <(yq -p xml b.xml) c.yaml
```

入力フォーマットの指定は `-p` または `--input-format` オプションで指定できます。\
同様に出力フォーマットは、 `-o` または `--output-format` オプションで指定できます。\
どちらのオプションもデフォルトは YAML となっています。
