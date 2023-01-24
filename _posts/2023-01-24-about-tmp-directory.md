---
layout: post
title: /tmpディレクトリについて
---

Alpine Linuxを使っていて、ふと`/tmp`ディレクトリがどう扱われているのか気になりました。というのも`/tmp`ディレクトリにとあるファイル`X`を作成したとき、このファイルはOSを再起動するまでは消えません。しかしコンテナというものはそもそもOSに対して`reboot`コマンドを実行する機会はありません。CI/CDで管理しているのですが、もし何か変更があった場合そのコンテナは再起動もしくは再作成するタイミングがあるのかもしれませんが、基本的には起動したままのはず。そしてコンテナは基本的に1プロセスでしか動かしていないので`cron`コマンドが実行されないということは`X`というファイルは`/tmp`ディレクトリという場所に配置されているだけで永久に消えない可能性があるでしょう。そうするとディスクの容量は枯渇してしまうかもしれない、そんなわけでいくつか実験してみることにしました。

## ホストOSの再起動

```
# touch /tmp/secret
# reboot
```
まずは**Ubuntu 22.04**の仮想マシンを作成して、`secret`というファイルを作成してから再起動してみました。
```
# cat /tmp/secret
cat: /tmp/secret: No such file or directory
```
ファイルが存在しないので、これは意図した挙動であるといえるでしょう。

## コンテナの再起動

続いて仮想マシンにDockerをインストールし、**node:18.13.0-alpine**のイメージを使用したコンテナを作成してみました。

```dockerfile
# Dockerfile
FROM node:18.13.0-alpine
RUN npm install --global serve
CMD serve
```
```yaml
# docker-compose.yml
version: "3"
services:
  node:
    build: .
```

このファイルを使ってまずはコンテナを起動します。

```
# docker exec 9d841b27e10c touch /tmp/secret
```

続いてコンテナに先ほどと同じように`secret`というファイルを作成します。

```
# docker compose restart
# docker exec 9d841b27e10c cat /tmp/secret
```

コンテナを再起動してみたところファイルはまだ残っていました。つまり単純にコンテナを再起動しただけではファイルは残っているということでしょうね。

## コンテナの再作成

```
# docker compose down
# docker compose up -d
# docker exec 474461d20542 cat /tmp/secret
cat: can't open '/tmp/secret': No such file or directory
```

コンテナを再作成してみると当然といえば当然なのでしょうけれどもファイルは残りませんでした。こちらは考えてみると当然の挙動と言えるでしょう。

## ホストOSの`/tmp`ディレクトリをマウントする

```yaml
version: '3'
services:
  node:
    build: .
    restart: always
    volumes:
      - /tmp:/tmp
```

続いてホストOSの`/tmp`ディレクトリをマウントしてみました。この方法はすべてのユースケースに必ずしも当てはまらないと思いますが、仮想マシンを作ってその上にDockerコンテナを作成する方法であればコンテナ側で作成したファイルはOS側のタスクとして処理されるはずです。

```
# docker exec feec05b45e84 touch /tmp/secret
# cat /tmp/secret
```

案外すんなり`/tmp`ディレクトリをマウントできるのかとも思いましたが、コンテナ側でファイルを作成してホストOSで同じファイルにアクセスすればそのファイルに問題なくアクセスすることが可能です。つまりこの状況でホストOSを再起動すればこのファイルは消えているはずです。

```
# cat /tmp/secret
cat: /tmp/secret: No such file or directory
# docker exec feec05b45e84 cat /tmp/secret
cat: can't open '/tmp/secret': No such file or directory
```

興味深いことにホストOSの再起動をしてもコンテナIDは変更されませんでした。それでいてファイルは消えていたのでこれで実質コンテナ上に`/tmp`ディレクトリができたと言っても差し支えないでしょう。

## `systemd-tmpfiles`でファイルを消す

ここからが実は今回の肝だったりするのですが、OSを再起動すると`/tmp`ディレクトリにあるファイルが消えているというタスクが実は`cron`や`systemd`などに定義されているようです。`grep`コマンドなどで探してみたところ、`systemd-tmpfiles`というコマンドが見つかりました。

```
# cat /usr/lib/systemd/system/systemd-tmpfiles-clean.service
#  SPDX-License-Identifier: LGPL-2.1-or-later
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)
DefaultDependencies=no
Conflicts=shutdown.target
After=local-fs.target time-set.target
Before=shutdown.target

[Service]
Type=oneshot
ExecStart=systemd-tmpfiles --clean
SuccessExitStatus=DATAERR
IOSchedulingClass=idle
```

`systemd-tmpfiles-clean.service`というファイルを見ると、シャットダウン時に`systemd-tmpfiles --clean`というコマンドを実行するように見えます。実際に`--clean`という引数ではすぐにファイルは消えなかったのですが、`--remove`というコマンドにすれば即座に`/tmp`ディレクトリの中が空になりました。このコマンドの役割としては単純な`rm`コマンドにアクセスタイムなどの状態をみてから消すか残すかを決めているのだと思います。

再起動時だけでなく、1日おきにも実行しているようなファイルも見受けられたのでおそらくは`/tmp`ディレクトリになにかファイルを残しておいても自動的に消えているのだろうと思います。

### `tmpreaper`を使う

今回は検証していませんが、`tmpreaper`というコマンドも存在しているようです。このコマンドと`cron`を組み合わせてもよいでしょう。

### ホストOSにマウントする理由

Alpine Linuxはコンテナであることから`systemd-tmpfiles`というコマンド以前に`systemd`がインストールされていません。`cron`も動かないことを考えると定期的に実行するにはコンテナ側でファイルの削除を定期実行するか、ホストOS側に`/tmp`ディレクトリの管理を任せるのがよいと思います。今回私のユースケースでは直接マウントが最も合致しそうですが、一般的にコンテナより上のOS領域はアプリケーションに直接関係ない箇所なのでこれが毎回最適な手段とは思いません。

もしホストOSの`/tmp`ディレクトリをマウントする方法が取れない場合のもう一つの対策としては`cron`を実行するためのコンテナを用意してそのコンテナでファイルを消してあげるなどでしょうか。原則1プロセスにつき1コンテナのルールに従いたいところですが、個人的にはそれをするくらいであるならばホストOSの`crontab`を書き換えて`docker exec`を遠隔で実行したりするほうがマシと思いました。

今回は検証のみで実際のアプリケーションにはまだ組み込んでいないのですが、実際に運用してみて問題なければ改めてこの投稿を更新したいと思っています。
