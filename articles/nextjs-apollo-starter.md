---
title: "Next.js + Apollo でシュッとGraphQLを始める"
emoji: "🏃"
type: "tech"
topics: ["Next.js", "Apollo", "GraphQL", "React"]
published: true
---

Next.js + Apollo な構成で、簡単に GraphQL を始めるためのテンプレートを作成しました。

https://github.com/yuta4j1/nextjs-apollo-starter

シンプルかつ良い感じのGraphQLサンドボックスとして使うつもりなので、以下の構成を取り入れてます。

- モノレポ構成 ... Next.js の API Routes 上に GraphQL サーバーを構築
- スキーマ駆動 ... GraphQL Code Generator にて client/server の型定義生成


## Next.js プロジェクトの作成

まずは `create-next-app` で Next.js プロジェクトを作成します。

```sh
npx create-next-app <project-name> --ts
```

## Apollo 関連の依存関係インストール

Apollo Client , Apollo Server の構成に必要な依存関係をインストールします。

```sh
npm install graphql @apollo/client @apollo/server @as-integrations/next
```

※ ```@as-integrations/next``` は、Next.js の API Routes と Apollo Serverの統合に使用します（[Apollo Server Integrations](https://www.apollographql.com/docs/apollo-server/integrations/integration-index/)）

## GraphQL Code Generator の依存関係インストール

GraphQL Code Generator の構成に必要な依存関係をインストールします。

```sh
npm install -D @graphql-codegen/cli @graphql-codegen/client-preset @graphql-codegen/typescript @graphql-codegen/typescript-resolvers
```

| 用途                     | 依存関係                                                               |
| ------------------------ | ---------------------------------------------------------------------- |
| CLI                      | `@graphql-codegen/cli`                                                 |
| client 型定義            | `@graphql-codegen/client-preset`                                       |
| server（リゾルバ）型定義 | `@graphql-codegen/typescript`, `@graphql-codegen/typescript-resolvers` |

## GraphQL のドキュメント定義

client/server 間で共通となるスキーマを作成します。

```graphql:apollo/documents/schema.gql
schema {
  query: Query
}

type Query {
  users: [User!]!
}

type User {
  name: String
}

query ALL_USERS {
  users {
    name
  }
}
```

## `codegen.ts` の作成

プロジェクトルートに、GraphQL Code Generator の構成ファイルを作成します。
※ 構成についての詳細は [`codegen.ts` file](https://the-guild.dev/graphql/codegen/docs/config-reference/codegen-config) を参照

```ts:codegen.ts
import { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "apollo/documents/**/*.gql",
  documents: ["apollo/documents/**/*.gql"],
  generates: {
    "./apollo/__generated__/client/": {
      preset: "client",
      plugins: [],
      presetConfig: {
        gqlTagName: "gql",
      },
    },
    "./apollo/__generated__/server/resolvers-types.ts": {
      plugins: ["typescript", "typescript-resolvers"],
    },
  },
  ignoreNoDocuments: true,
};

export default config;
```

## 型定義の自動生成

npm scripts に以下のコマンドを追加します。

```diff json:package.json
  "scripts": {
+   "compile": "graphql-codegen",
+   "watch": "graphql-codegen -w",
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
```

試しにスキーマから型を自動生成するコマンドを実行してみます。

```sh
npm run compile
> nextjs-apollo-starter@0.1.0 compile
> graphql-codegen

✔ Parse Configuration
✔ Generate outputs
```

`apollo/___generated___` フォルダ配下に、自動生成された型定義ファイルが確認できれば OK です ✅

## GraphQL サーバーの作成

Next.js の [API Routes](https://nextjs.org/docs/api-routes/introduction) を使って、Next.js サーバー上に GraphQL サーバーを作成します。

:::message
API Routes は現在（2023/3/14）時点で ```app``` directory に対応していないため、app directory の構成で Next.js プロジェクトを構成している場合は、新たに `pages/api` ディレクトリを作成する必要があります。
:::

```ts:pages/api/graphql.ts
import { readFileSync } from "fs";
import { join } from "path";
import { startServerAndCreateNextHandler } from "@as-integrations/next";
import { ApolloServer } from "@apollo/server";
import { Resolvers } from "../../apollo/__generated__/server/resolvers-types";

const typeDefs = readFileSync(
  join(process.cwd(), "apollo/documents/schema.gql"),
  "utf-8"
);

const resolvers: Resolvers = {
  Query: {
    users() {
      return [{ name: "Nextjs" }, { name: "Nuxtjs" }, { name: "Sveltekit" }];
    },
  },
};

const apolloServer = new ApolloServer<Resolvers>({ typeDefs, resolvers });

export default startServerAndCreateNextHandler(apolloServer);
```

ここで一度 Next.js サーバーを起動し、`api/graphql` にアクセスしてみます。  
以下のように、Apollo Sandbox が起動していることが確認できれば GraphQL サーバーの作成は完了です ✅

![Apollo Sandbox 画面](/images/nextjs-apollo-starter/apollo-sandbox.png)

さて、サーバーの準備ができたので、最後にクライアント側からサーバー側への GraphQL リクエストを投げられるようにしていきます。

## apollo client の作成

まず、apollo client を作成します。

```ts:apollo/client.ts
import { ApolloClient, InMemoryCache } from "@apollo/client";

export const client = new ApolloClient({
  uri: "/api/graphql",
  cache: new InMemoryCache(),
});
```

## コンポーネントの作成

Apollo Provider を定義するためのコンポーネントを作成します。

```tsx:app/components/WithApollo.tsx
"use client";

import { ApolloProvider } from "@apollo/client";
import { client } from "../../apollo/client";

const WithApollo = ({ children }: React.PropsWithChildren) => {
  return <ApolloProvider client={client}>{children}</ApolloProvider>;
};

export default WithApollo;
```

クエリを実行するコンポーネントを以下のように定義します。

```tsx:app/components/User.tsx
"use client";

import { useQuery } from "@apollo/client";
import { gql } from "../../apollo/__generated__/client";

const ALL_USERS = gql(`query ALL_USERS {
  users {
    name
  }
}`);

const User = () => {
  const { data, loading, error } = useQuery(ALL_USERS);
  if (loading) {
    return <div>読み込み中</div>;
  }
  return (
    <div>
      {error && <div>{error.message}</div>}
      <ul>
        {data && data.users.map((v, i) => <li key={String(i)}>{v.name}</li>)}
      </ul>
    </div>
  );
};

export default User;
```

ルートページの定義は以下の形。

```tsx:app/page.tsx
import styles from "./page.module.css";
import WithApollo from "./components/WithApollo";
import User from "./components/User";

export default function Home() {
  return (
    <main className={styles.main}>
      <WithApollo>
        <User />
      </WithApollo>
    </main>
  );
}
```

Next.js サーバーを起動して確認してみると、以下のようにサーバーからリストデータが取得できていることを確認できます。

![完成画面](/images/nextjs-apollo-starter/complete-screen.png)


完成！🎉

