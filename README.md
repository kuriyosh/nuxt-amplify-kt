# Amplify KT
## Amplify とは
フロントエンド (Web Frontend, モバイルアプリ) とバックエンドの連携をお手伝いしてくれるツール

- Amplify Library: Web Frontend やモバイルアプリから AWS を利用したサーバーレスバックエンドに接続する機能の提供
  - 前のプロジェクトでフロントとバックエンドの連携を担っていた部分
- Amplify CLI: サーバーレスバックエンドを構築する機能
  - 前のプロジェクトでは AWS SAM, (CloudFormation CFn) がやってくれた部分
- Amplify Admin UI: Amplify CLI でやってくれるようなバックエンドの構築部分や、CMS チックな機能の提供

今回のプロジェクトは CLI も使ってより、バックエンドをマネージドにしていきたい  
Admin UI も場合によっては採用しても良いかも？

## ゆるく使い方説明
### プロジェクトの初期設定
基本的にチュートリアルの内容を話す

Amplify CLI のインストールどの AWS アカウント、どのリージョンにリソースをデプロイするか等、Amplify 全体の設定を予めやっておく必要がある  
今回は説明割愛
https://docs.amplify.aws/start/getting-started/installation/q/integration/vue

`amplify init` を実行するとプロジェクトに　Amplify が導入される

以下を聞かれるのでそれぞれ答えて設定する
```
? Enter a name for the project → nuxtAmplifyKt
? Enter a name for the environment → dev
? Choose your default editor: → Visual Studio Code
? Choose the type of app that you're building → javascript
Please tell us about your project
? What javascript framework are you using → vue
? Source Directory Path: → src
? Distribution Directory Path: → dist
? Build Command: → yarn build
? Start Command: → yarn start
Using default provider  awscloudformation
? Select the authentication method you want to use: → AWS profile
```

この時点で AWS Management Console 側に Amplify のプロジェクトが作成されている

### 作成物
ちょっとしたブログアプリを作成していく

### GraphQL の API を作成する
`amplify add api` を実行  
interactive form が出てくるので、入力していく  
今回は要件としてポストされた記事は誰でも見れる、記事の編集は owner しかできないという機能を実現したい  
そのため、API の認証に API key と Cognito の両方を指定している

```
? Please select from one of the below mentioned services: GraphQL
? Provide API name: nuxtamplifykt
? Choose the default authorization type for the API API key
? Enter a description for the API key:
? After how many days from now the API key should expire (1-365): 7
? Do you want to configure advanced settings for the GraphQL API Yes, I want to make some additional changes.
? Configure additional auth types? Yes
? Choose the additional authorization types you want to configure for the API Amazon Cognito User Pool
Cognito UserPool configuration
Using service: Cognito, provided by: awscloudformation

 The current configured provider is Amazon Cognito.

 Do you want to use the default authentication and security configuration? Default configuration
 Warning: you will not be able to edit these selections.
 How do you want users to be able to sign in? Username
 Do you want to configure advanced settings? No, I am done.
Successfully added auth resource nuxtamplifykt71bebe76 locally

Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud

? Enable conflict detection? No
? Do you have an annotated GraphQL schema? No
? Choose a schema template: Single object with fields (e.g., “Todo” with ID, name, description)
```

ブログ記事 Post のスキーマを定義する

```graphql
type Post 
@model 
@auth(rules: [{ allow: owner }, { allow: public, operations: [read] }])
{
  id: ID!
  title: String!
  content: String!
}
```

`@auth` directive には、owner にはすべての許可、public には read のみと設定されている
(ややこしいけど、API へのアクセスがそもそも API key がないといけないので、public と書く)

この状態だとローカルにリソースの設計図があるだけなので、実際に AWS 上にリソースを作成する必要がある

`amplify push` の実行
codegen 的な形で型定義も作成してくれる、クエリ系は一般的な CRUDL は勝手に作ってくれる

```
? Do you want to generate code for your newly created GraphQL API → Yes
? Choose the code generation language target → typescript
? Enter the file name pattern of graphql queries, mutations and subscriptions → src/graphql/**/*.ts
? Do you want to generate/update all possible GraphQL operations - queries, mutations and subscriptions → Yes
? Enter maximum statement depth [increase from default if your schema is deeply nested] → 2
? Enter the file name for the generated code → src/API.ts
```

**作成されるファイルと使われ方**
- GraphQL 系
  - src/graphql/mutations.ts: mutation 系の定義
  - src/graphql/queries.ts: query 系の定義
  - src/API.ts: GraphQL 型定義全般
- Amplify 設定系
  - aws-export.js: 実際のリソース ID の定義

実際に Management Console を見に行くとリソースが作成されている

### フロント側との連携
認証画面と、ブログ投稿・一覧画面くらいを作る

Amplify と連携するために、Amplify Library と、今回は hands-on したいだけなので、ログイン周りの UI set をインストールする

plugin 化しとく

Amplify Library と UI set を import
```js
import Vue from 'vue'
import Amplify from 'aws-amplify'
import '@aws-amplify/ui-vue'
import awsExports from '~/aws-exports.js'

Amplify.configure(awsExports)
Vue.use(Amplify)
```

`nuxt.config.js` にも plugin を import
```
  plugins: [{ src: '~/plugins/amplify.js', mode: 'client' }],
```

**すまん、本当は純粋に SSR したかったけど間に合わんかった、できんのんかもしれん**

#### ログイン画面
`<amplify-authenticator>` で囲っておけば、ログイン画面だしてくれるし、囲ってる部分は認証されないと見えないようになる

```vue
<template>
  <amplify-authenticator>
    <section class="section">
      ...
      <amplify-sign-out></amplify-sign-out>
    </section>
  </amplify-authenticator>
</template>
```

サインアップしてみる、うまくいっている

#### 一覧、投稿画面
適当に画面を設計 (`src/pages/index.vue` を参照)

GraphQL API を利用したいので、aws-amplify から API を import

```js
import { API } from 'aws-amplify'
```

また、利用する operation を import

```js
import { createPost, deletePost } from '~/graphql/mutations'
import { listPosts } from '~/graphql/queries'
```

基本的な使い方は以下

```js
const res = await API.graphql({
          authMode: GRAPHQL_AUTH_MODE.AMAZON_COGNITO_USER_POOLS, // 今回、デフォルトは API キー認証なので、Cognito で認証したいときは指定
          query: deletePost, // import した opeartion を記載する
          variables: { // query にパラメータが指定可能な場合はここで object 型で指定
            input: {
              id: postId,
            },
          },
        })
```

createPost すると、新しく Post が作成されることがわかる  
実際にバックエンドを見ると作成されている  
owner オブジェクトが自動生成されている

スキーマの `@auth` directive で read 以外は public ではない  
なので、deletePost が自分が owner の Post でないと削除できないことがわかる