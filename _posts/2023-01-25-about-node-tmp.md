---
layout: post
title: node-tmpについて
---

昨日の投稿に続いて[Tmp](https://github.com/raszi/node-tmp)というライブラリを使おうと思っています。

> If graceful cleanup is set, tmp will remove all controlled temporary objects on process exit, otherwise the temporary objects will remain in place, waiting to be cleaned up on system restart or otherwise scheduled temporary object removal.

こちらはREADMEからの引用なのですが、プロセスが終了するまでは`/tmp`の中身は削除されません。一回きりのスクリプトであればこの要件を満たすことができますが、Webアプリケーションだとこうはいきません。そのため一定期間で削除する必要が出てくるので、今回はこのあたりの挙動を考えてみます。

## 一時ディレクトリを作成してから実行する

```javascript
var Queue = require("bull");
var tmp = require("tmp");

var queue = new Queue();

queue.process(function (job, done) {
  tmp.dir(function (err, tmpPath) {
    if (err) {
      done(err);
    }

    console.log(tmpPath);

    done();
  });
});
```

この例では[Bull](https://github.com/OptimalBits/bull)を使っています(2023年にアロー関数や`const`を使わないのはどうなんだと思われるかもしれませんが)。コードの内容は非同期で`/tmp`ディレクトリに任意のディレクトリを作成して、そのパスを返して終了するというものです。実際に実行してみましょう。

```
$ docker exec ea6f3816455d ls -1 /tmp
tmp-64-40TJfmXd0xeD
v8-compile-cache-0
```

ここでは`tmp-64-40TJfmXd0xeD`というディレクトリが生成されました。このライブラリが説明するところではこのディレクトリの中で作業を行い、ファイルがすべて削除されていればプロセス終了後にこのディレクトリごとなくなっているはずです。

```javascript
queue.process(function (job, done) {
  tmp.dir(function (err, tmpPath) {
    if (err) {
      done(err);
    }

    console.log(tmpPath);

    if (Math.random() > 0.25) {
      throw new Error("oops.");
    }

    done(null, "success!");
  });
});
```

ではここでプログラムを書き換えてみましょう。もし処理中に何らかの理由で失敗した場合、Bullはディレクトリをそのまま残すでしょう。ディレクトリの扱いをどうするかはその時々によって変わると思うのですが、もし失敗してそのファイルそのものが不要になるのであれば`try...catch`で失敗時にファイルやディレクトリを削除してしまえばよいと思います。実際にそういう実装はそれまで行っていました。

では今回どのような方法を取りたいかというと、**失敗したファイルはそのまま残しておいて、途中で再開を試みる**ような実装をしてみたいと思います。最もわかりやすい例としてはファイルのダウンロードだと思います。ブラウザのダウンロード機能でもリトライボタンを押すとダウンロードフォルダにあるpartファイルをもとにダウンロードが再開できることがありますよね。

しかし上述のコードの場合は何が問題かというと、Bullで失敗したジョブをリトライしようとすると毎回別のディレクトリに移動してしまい、毎回新規のファイルで開始してしまうのです。

```javascript
tmp.dir(function (err, tmpPath) {
  if (err) {
    console.error(err);
    return;
  }

  queue.add({ tmpPath: tmpPath });
});
```

その解決策としてはBullを使って新しいジョブをエンキューする前に一時ディレクトリを作成しておけばよいと思いました。ただしこの方法はあくまでもホストOS側にBullが動作している前提なので必ずしも正しくありません。

```javascript
function work(job, done) {
  console.log(job.data.tmpPath);

  if (Math.random() > 0.25) {
    done(new Error("oops."));
  }

  done(null, "success!");
}

queue.process(function (job, done) {
  if (job.data.tmpPath) {
    work(job, done);
    return;
  }

  var tmpobj = tmp.dirSync();

  job
    .update(
      Object.assign(job.data, {
        tmpPath: tmpobj.name,
      })
    )
    .then(function () {
      work(job, done);
    })
    .catch(done);
});
```

**NOTE:** 調べてみると`Object.assign`はES5の時代には使えなかったようなので、ES6も許容すると`const`やアロー関数使わない縛りの意味をなしませんね。😅

もしタスクを開始あるいは再開した場合に一時ディレクトリがあればそれを使うし、なければ新しく作ってからタスクを開始するようにしてあげればよいと思います。実際処理の終了後にディレクトリを残すのか、あるいは消すのかはディスク容量とも相談なのですが、`/tmp`の処理はホストOS側に任せるとしてひとまずはこれでよいのではないでしょうか。

```
$ docker exec ea6f3816455d ls /tmp/tmp-287-KcwsoHrB0N9K
```

いざジョブを開始してみたところ運悪く(?)5回も連続でエラーが発生しましたが、一度作成したディレクトリを使いまわしてくれるようになったのでディレクトリが増え続けることがなくなりました。この処理であれば`os.tmpdir()`を使えばわざわざライブラリの力を借りる必要はなかったのかもしれませんが、`cleanupCallback`や`cleanupCallback`は後から選択すればよいでしょう。

## おまけ

この章では先ほどの例に出てきた`work`関数の中身について少し見てみたいと思います。

```javascript
var { spawn } = require("child_process");

module.exports = function work(job, done) {
  var pwd = spawn("pwd", {
    cwd: job.data.tmpPath,
  });
};
```

Node.jsの[child_process.spawn()](https://nodejs.org/api/child_process.html#child_processspawncommand-args-options)では`cwd`をオプション引数として渡すことができるので、ここに使いたいコマンドと生成した一時ディレクトリを渡せばそのディレクトリの中に`cd`で移動して実行しているようにプログラムを実行することができます。あくまでこの書き方だと`cleanupCallback`を使えないという課題は残るのですが、これで任意のプログラムを制御しやすくなりました。
