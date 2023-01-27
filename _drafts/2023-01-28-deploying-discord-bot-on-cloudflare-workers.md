---
layout: post
title: Cloudflare WorkersでDiscord Botを使う
---

私のLAN内にはDockerコンテナで作ったサービスが稼働していて、現状はそこに`curl`を飛ばすことでしかジョブの開始をすることしかできていません。最近実行結果はDiscordのWebhookを利用しているのですが、せっかくDiscordを使っていることだしコマンドの開始も同様にDiscord上から行えるようにしてみようというのが今回の試みです。

以前まではHerokuというサービスが無料枠で実行環境を提供してくれていたのですが、今は使うことができません。2023年になった今は同様のサービスがいくつかありますが、私はCloudflare Workersをデプロイ先に選んでみることにしました。というのも[Hosting a Reddit API Discord app on Cloudflare Workers](https://discord.com/developers/docs/tutorials/hosting-on-cloudflare-workers#hosting-a-reddit-api-discord-app-on-cloudflare-workers)という記事がDiscord上で公開されているのでこれを参考にしようと思った次第です。しかしこうして今回ブログの投稿にしているのはここで紹介されている方法が古いのか、CloudflareのAPIがちょうど変わった時期のようでいくつか注意点があったため参考に残しておこうと思ったためです。

## Cloudflare側の設定

### `wrangler`のインストール

まずは[Cloudflare](https://workers.cloudflare.com/)のトップページを見てそのまま以下のコマンドを実行してみます:

```sh
$ npx wrangler init azusabot
```

特にオプションを指定しなければ対話型のコマンドとしてGitは使うかとか、TypeScriptは使うかなど聞かれるのでその内容にそって入力していきます。私はすべて**y**にしました。

```typescript
export interface Env {
}

export default {
	async fetch(
		request: Request,
		env: Env,
		ctx: ExecutionContext
	): Promise<Response> {
		return new Response("Hello World!");
	},
};
```

コマンド生成後は上のファイルが生成されました(コメントは削除しています)。それぞれ`request`はDiscordなどからくるリクエスト、`env`は`wrangler`で設定した内容が使えます。ここで注意したいのが、**`wrangler`内では`process.env`が使えないということ。**[Cloudflare WorkersのランタイムはNode.jsとは別物](https://github.com/remix-run/remix/issues/1285#issuecomment-1014978033)のようです。つまりローカルのみで開発がしたいと思ってもCloudflare内で*Total Requests*が増えてしまうのが少々残念です。

### Cloudflareでログインする

**TODO:** 次回また新しいWorkerを作ったときなどに追記予定です。

## Discord側の設定

次にDiscordにBotとして認識させるために[Applications](https://discord.com/developers/applications)から新しいBotを作成します。今回は以前に作ったまま放置されていたBotをそのまま流用したのですが、必要な環境変数さえあればよいので作成に関しては残しません。

### 必要なトークンの取得

**TODO:** TOKENの取得の方法を追記する予定です。再取得は*Settings*の*Bot*で*Reset Token*からできた気がします。

*Settings*の*General Information*からそれぞれ**APPLICATION ID**と**PUBLIC KEY**をコピーしておきます。

続いて取得したトークンを`wrangler`に追加します:

```sh
$ wrangler secret put DISCORD_TOKEN
$ wrangler secret put DISCORD_PUBLIC_KEY
$ wrangler secret put DISCORD_APPLICATION_ID
```

### Botをテスト用のチャンネルに招待する

続いて作成したBotを招待します。*Settings*の*OAuth2 URL Generator*で**bot**と**applications.commands**を選択します。

*Bot Permissions*ではサンプルなので**Send Messages**と**Use Slash Commands**にチェックを入れました。招待後からでもサーバー管理者であればあとから権限を変更できるのでここでは気にしません。

あとは生成されたURLを実際にブラウザでアクセスしてBotを自分のサーバーに招待できればOKです。

必要に応じて`wrangler`に登録したサーバー(Discord上では*Guild*)を登録してもよいでしょう。自分の*Guild ID*の取得は*サーバー設定*の*ウィジェット*内に**サーバーID**があるのでそこから取得できます。

```sh
$ wrangler secret put DISCORD_TEST_GUILD_ID
```

## コードの追加

### コマンドの登録

まずDiscord上に使う予定のコマンドを登録してあげないといけません。

これは本来ならば実際のアプリケーション内に組み込みたいところですが、あくまでCloudflare Workersは関数として実行される都合上起動時に実行するなどの操作はできません。なので単体のスクリプトとして用意しておいて、ローカルのNode.jsから実行するのが一番手っ取り早いと思います。

```javascript
const dotenv = require('dotenv')

dotenv.config()

const Axios = require('axios')

/**
 * This file is meant to be run from the command line, and is not used by the
 * application server.  It's allowed to use node.js primitives, and only needs
 * to be run once.
 */

const token = process.env.DISCORD_TOKEN;
const applicationId = process.env.DISCORD_APPLICATION_ID;

if (!token) {
  throw new Error('The DISCORD_TOKEN environment variable is required.');
}
if (!applicationId) {
  throw new Error(
    'The DISCORD_APPLICATION_ID environment variable is required.'
  );
}

/**
 * Register all commands globally.  This can take o(minutes), so wait until
 * you're sure these are the commands you want.
 */
async function registerGlobalCommands() {
  const url = `https://discord.com/api/v10/applications/${applicationId}/commands`;
  await registerCommands(url);
}

async function registerCommands(url) {
  const response = await Axios({
    method: 'PUT',
    url: url,
    data: [
      {
        name: 'awwww',
        description: 'Drop some cuteness on this channel.',
      },
      {
        name: 'invite',
        description: 'Get an invite link to add the bot to your server',
      }
    ],
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bot ${token}`,
    },
  });

  if (response.status === 200) {
    console.log('Registered all commands');
  } else {
    console.error('Error registering commands');
    const text = response.data;
    console.error(text);
  }
  return response;
}

;(async function() {
  await registerGlobalCommands()
})()
```

先ほどセットしたトークンを`.env`ファイルに保存しておき、このスクリプトを実行すると`/awwww`と`/invite`が実際にDiscord上で使えるようになります。

### Interactions Endpoint URLに登録する

続いて`index.ts`を次のように修正します:

```typescript
import { InteractionResponseType, InteractionType, verifyKey } from 'discord-interactions';

export interface Env {
  DISCORD_PUBLIC_KEY: string;
}

export default {
  async fetch(
    request: Request,
    env: Env,
    ctx: ExecutionContext
  ): Promise<Response> {
    const rawBody = await request.text();
    const signature = request.headers.get('X-Signature-Ed25519');
    const timestamp = request.headers.get('X-Signature-Timestamp');
    const pubKey = env.DISCORD_PUBLIC_KEY;

    if (signature == null || timestamp == null) {
      return new Response('Sorry, you have supplied an invalid key.', {
        status: 403,
      });
    }

    const isValidRequest = verifyKey(rawBody, signature, timestamp, pubKey);

    if (!isValidRequest) {
      return new Response('Bad request signature', {
        status: 401,
      });
    }

    return new Response(JSON.stringify({
      type: InteractionResponseType.PONG,
    }), {
      headers: {
        'content-type': 'application/json;charset=UTF-8',
      },
    });
  },
};
```

これは常にDiscordのInteractionsを通過させるためのレスポンスを返すだけですが、この状態でないとDiscordからリクエストを通す画面に登録できません。まずはこのコードで登録できる状態にしておいてから徐々にBotの開発を進めていくようにしました。
