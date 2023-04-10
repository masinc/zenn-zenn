---
title: "sudo -i と sudo su - の違い"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "sudo", "su"]
published: true
---

どちらも root ユーザーに昇格できるがこれらの違いが気になったので調査結果をメモ。

まず、これらのコマンドがどのような意味であるかを調べました。

- `sudo`: **S**uper **U**ser **DO**
- `su`: **S**ubstitute **U**ser

`sudo` `-i`オプションの意味は `simulate initial login` であるようだ。

# 違い 1: 実行されるプロセス数が違う

当然だが、`sudo su -` は`sudo`と`su`コマンドを実行する。root に昇格している間はこれらのプロセスが立ち上がったままとなります。
対して`sudo -i`コマンドは`sudo`コマンドのみを実行する。root に昇格している間は`sudo`プロセスのみとなります。

どちらも利用したことがあればわかるが、体感的な速度差はないです。

リソースは以下のような結果となりました。

## `sudo su -`昇格時

```bash
~# pidstat -r -C su
Linux 5.4.0-62-generic (ubuntu2004)     2021年01月25日  _x86_64_        (1 CPU)

20時50分59秒   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
20時50分59秒     0    233553      0.00      0.00   10368    4752   0.51  sudo
20時50分59秒     0    233554      0.00      0.00    9428    4208   0.45  su
```

```bash
$ time sudo su - -c :

real    0m0.013s
user    0m0.008s
sys     0m0.003s
```

## `sudo -i`昇格時

```bash
~# pidstat -r -C su
Linux 5.4.0-62-generic (ubuntu2004)     2021年01月25日  _x86_64_        (1 CPU)

20時51分47秒   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
20時51分47秒     0    233567      0.00      0.00   10368    4584   0.49  sudo
```

```bash
$ time sudo -i :

real    0m0.009s
user    0m0.006s
sys     0m0.003s
```

リソース面だけでみれば、`sudo`コマンドの結果はほぼ変わらず、`su`コマンドが加わった分、`sudo -i` が有利なようです。

# 違い 2: 環境変数

## `sudo -i`の場合以下の環境変数がセット

- SUDO_COMMAND: 実行コマンド
- SUDO_UID: 昇格前の UID
- SUDO_GID: 昇格前の GID
- SUDO_USER: 昇格前の USER

## CentOS の `sudo -i`だと/usr/local/bin が PATH 環境変数がセットされていない。

CentOS8 の動作(CentOS7 でも同様)

```bash
$ sudo -i echo '$PATH'
/usr/local/sbin:/sbin:/bin:/usr/sbin:/usr/bin:/root/.dotnet/tools:/root/bin

$ sudo su - -c 'echo $PATH'
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/.dotnet/tools:/root/bin
```

Ubuntu20.04 では PATH 環境変数に /usr/local/bin がセットされていることを確認。

```bash
$ sudo -i echo '$PATH'
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/root/.dotnet/tools

$ sudo su - -c 'echo $PATH'
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/root/.dotnet/tools
```

これは `/etc/sudoers` のディストリビューションごとのデフォルト設定による違いとなる模様です。

CentOS8

```bash
$ sudo grep secure_path /etc/sudoers
Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
```

Ubuntu20.04

```bash
$ sudo grep secure_path /etc/sudoers
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

`su`コマンドで PATH 環境変数は `su`コマンドがデフォルトでセットしているようです。

```
       ENV_SUPATH (string)
           Defines the PATH environment variable for root.  The default value is /usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin.
```

## 環境変数の引継ぎ

`sudo -i`では環境変数が引き継がれます。

```bash
$ sudo -i date
2021年  1月 25日 月曜日 20:36:09 JST

$ LANG=C sudo -i date
Mon Jan 25 20:36:25 JST 2021
```

`sudo su -`では CentOS8, Ubuntu2004 いずれでもデフォルトでは環境変数が引き継がれませんでした。

```bash
$ sudo su - -c date
2021年  1月 25日 月曜日 20:34:49 JST

$ LANG=C sudo su - -c date
2021年  1月 25日 月曜日 20:35:44 JST
```

これは一長一短であり、個人的には引き継ない`sudo su -`の動作が好きです。

# まとめ

| 違い               |                                        sudo -i                                        | sudo su -                                                    |
| :----------------- | :-----------------------------------------------------------------------------------: | ------------------------------------------------------------ |
| プロセス           |                                     sudo のみ実行                                     | sudo, su を実行                                              |
| デフォルト環境変数 | /etc/sudoers の secure_path <br> (ディストリビューションによってデフォルト値が異なる) | /usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin |
| 環境変数の引継ぎ   |                                       引き継ぐ                                        | デフォルトでは引き継がない                                   |

各種リソース面では `sudo -i`が有利、環境変数関連は `sudo su -`のほうが好みでした。(ディストリビューションの違いを意識しなくてよい)

用途によって使い分けるとよさそう。

# 参考情報

https://www.maketecheasier.com/differences-between-su-sudo-su-sudo-s-sudo-i/

https://serverfault.com/questions/359856/what-is-the-difference-between-sudo-i-and-sudo-su

https://askubuntu.com/questions/376199/sudo-su-vs-sudo-i-vs-sudo-bin-bash-when-does-it-matter-which-is-used

http://min117.hatenablog.com/entry/2017/05/21/081703

https://qiita.com/aosho235/items/05d4a4f549016e41cde7
