---
title: "【GitHub認証】Next.js + Lucia Auth + Prisma + Supabase"
emoji: "💜"
type: "tech"
topics: ["nextjs", "prisma", "lucia", "auth", "supabase"]
published: true
---

# 🚀 [Next.js](https://nextjs.org/) + [Lucia Auth](https://lucia-auth.com/) + [Prisma](https://www.prisma.io/) + [Supabase](https://supabase.com/) を使用したGitHub認証

この記事では、[Next.js](https://nextjs.org/)、[Lucia Auth](https://lucia-auth.com/)、[Prisma](https://www.prisma.io/)、[Supabase](https://supabase.com/) を使用して、GitHub認証を実装する方法を紹介します。

> Luciaは、セッション処理の複雑さを抽象化する、サーバ用の認証ライブラリです。データベースと共に動作し、使いやすく、理解しやすく、拡張しやすいAPIを提供します。

https://lucia-auth.com/


👇 本記事のソースコードはこちら
https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase

:::message
ひよっこ学生エンジニアのため、誤りや不足があればご指摘いただけると幸いです。🐥
:::

## 🌟 1. [Next.js](https://nextjs.org/)のセットアップ
### 1 - 1. [Next.js](https://nextjs.org/)のプロジェクトを作成します。
```bash
pnpm dlx create-next-app@latest next-lucia-prisma
cd next-lucia-prisma
```
### 1 - 2. 必要なライブラリをインストールします。
```bash
pnpm add @lucia-auth/adapter-prisma @prisma/client lucia
pnpm add -D prisma
```

## 🗄️ 2. [Supabase](https://supabase.com/)のセットアップ
ローカル環境で簡単にデータベースを使用できるので[Supabase](https://supabase.com/)を使用していますが、他のデータベースでも構いません。

### 2 - 1. ローカル環境の[Supabase](https://supabase.com/)をセットアップします。
詳しくは[公式ドキュメント](https://supabase.com/docs/guides/cli/local-development?queryGroups=access-method&access-method=postgres)を参照してください
```bash
pnpm dlx supabase init
```
### 2 - 2. ローカル環境の[Supabase](https://supabase.com/)を起動します。
内部で[Docker](https://www.docker.com/ja-jp/)を使用しているため、事前にインストールが必要です。
```bash
pnpm dlx supabase start
```

::::details 🛑 停止する場合
ローカル環境の[Supabase](https://supabase.com/)を停止する場合は以下のコマンドを実行してください。
```bash
pnpm dlx supabase stop
```
::::

## 📊 3. Prismaのセットアップ
### 3 - 1. Prismaの初期化
実行すると、`prisma`ディレクトリと`.env`ファイルが作成されます。
```bash
pnpm prisma init
```
### 3 - 2. Prismaのスキーマを作成します。
`prisma/schema.prisma`を以下のように編集します。
```js
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id       String    @id
  sessions Session[]
  githubId Int       @unique
  username String
}

model Session {
  id        String   @id
  userId    String
  expiresAt DateTime

  user User @relation(references: [id], fields: [userId], onDelete: Cascade)
}
```
### 3 - 3. Prismaのデータベースを設定します。
`.env`ファイルを以下のように編集します。
[GitHubのOAuthアプリを作成](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)します。リダイレクトURLは`http://localhost:3000/login/github/callback`を指定してください。
```bash
DATABASE_URL="postgresql://postgres:postgres@127.0.0.1:54322/postgres"

GITHUB_CLIENT_ID="xxx"
GITHUB_CLIENT_SECRET="xxx"
```
### 3 - 4. Prismaのデータベースをマイグレーションします。
```bash
pnpm dlx prisma migrate dev --name init
```

## 🔒 4. Lucia Authのセットアップ
### 4 - 1. Lucia Authの初期化
👇 [src/libs/auth.ts](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/libs/auth.ts)の作成
```ts
import { PrismaAdapter } from "@lucia-auth/adapter-prisma";
import { PrismaClient } from "@prisma/client";
import { Lucia, Session, User } from "lucia";
import { GitHub } from "arctic";
import { cache } from "react";
import { cookies } from "next/headers";

const client = new PrismaClient();

const adapter = new PrismaAdapter(client.session, client.user);

export const lucia = new Lucia(adapter, {
  sessionCookie: {
    // this sets cookies with super long expiration
    // since Next.js doesn't allow Lucia to extend cookie expiration when rendering pages
    expires: false,
    attributes: {
      // set to `true` when using HTTPS
      secure: process.env.NODE_ENV === "production",
    },
  },
  getUserAttributes: (attributes) => {
    return {
      // attributes has the type of DatabaseUserAttributes
      githubId: attributes.githubId,
      username: attributes.username,
    };
  },
});

// IMPORTANT!
declare module "lucia" {
  interface Register {
    Lucia: typeof lucia;
    DatabaseUserAttributes: DatabaseUserAttributes;
  }
}

interface DatabaseUserAttributes {
  githubId: number;
  username: string;
}


export const github = new GitHub(process.env.GITHUB_CLIENT_ID!, process.env.GITHUB_CLIENT_SECRET!);

export const validateRequest = cache(
	async (): Promise<{ user: User; session: Session } | { user: null; session: null }> => {
		const sessionId = cookies().get(lucia.sessionCookieName)?.value ?? null;
		if (!sessionId) {
			return {
				user: null,
				session: null
			};
		}

		const result = await lucia.validateSession(sessionId);
		// next.js throws when you attempt to set cookie when rendering page
		try {
			if (result.session && result.session.fresh) {
				const sessionCookie = lucia.createSessionCookie(result.session.id);
				cookies().set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
			}
			if (!result.session) {
				const sessionCookie = lucia.createBlankSessionCookie();
				cookies().set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
			}
		} catch {}
		return result;
	}
);
```

👇 [src/libs/db.ts](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/libs/db.ts)の作成
```ts
import { PrismaClient } from '@prisma/client';

export const db = new PrismaClient();
```

## 🔑 5. GitHub認証の実装
👇 [src/app/login/github/route.ts](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/app/login/github/route.ts)の作成
```ts
import { generateState } from "arctic";
import { cookies } from "next/headers";
import { github } from "~/libs/auth";

export async function GET(): Promise<Response> {
	const state = generateState();
	const url = await github.createAuthorizationURL(state);

	cookies().set("github_oauth_state", state, {
		path: "/",
		secure: process.env.NODE_ENV === "production",
		httpOnly: true,
		maxAge: 60 * 10,
		sameSite: "lax"
	});

	return Response.redirect(url);
}
```
👇 [src/app/login/github/callback/route.ts](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/app/login/github/callback/route.ts)の作成
```ts
import { github, lucia } from "~/libs/auth";
import { cookies } from "next/headers";
import { OAuth2RequestError } from "arctic";
import { generateIdFromEntropySize } from "lucia";
import { db } from "~/libs/db";

export async function GET(request: Request): Promise<Response> {
  try {
    const url = new URL(request.url);
    const code = url.searchParams.get("code");
    const state = url.searchParams.get("state");
    const storedState = cookies().get("github_oauth_state")?.value ?? null;

    if (!code || !state || !storedState || state !== storedState) {
      console.error("Invalid OAuth state or code", {
        code,
        state,
        storedState,
      });
      return new Response(null, { status: 400 });
    }

    const tokens = await github.validateAuthorizationCode(code);

    const githubUserResponse = await fetch("https://api.github.com/user", {
      headers: {
        Authorization: `Bearer ${tokens.accessToken}`,
      },
    });

    if (!githubUserResponse.ok) {
      console.error(
        "Failed to fetch GitHub user data",
        await githubUserResponse.text()
      );
      return new Response(null, { status: 500 });
    }

    const githubUser: GitHubUser = await githubUserResponse.json();
    const existingUser = await db.user.findUnique({
      where: { githubId: githubUser.id. },
    });

    let userId: string;
    if (existingUser) {
      userId = existingUser.id;
    } else {
      userId = generateIdFromEntropySize(10);
      await db.user.create({
        data: {
          id: userId,
          githubId: githubUser.id,
          username: githubUser.login,
        },
      });    }

    const session = await lucia.createSession(userId, {});
    const sessionCookie = lucia.createSessionCookie(session.id);
    cookies().set(
      sessionCookie.name,
      sessionCookie.value,
      sessionCookie.attributes
    );

    return new Response(null, {
      status: 302,
      headers: {
        Location: "/",
      },
    });
  } catch (e) {
    console.error("Error in OAuth callback handler", e);
    if (e instanceof OAuth2RequestError) {
      return new Response(null, { status: 400 });
    }
    if (e instanceof Error) {
      console.error(e.message);
    }
    return new Response(null, { status: 500 });
  }
}

interface GitHubUser {
  id: string;
  login: string;
}
```

👇 [src/app/login/page.tsx](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/app/login/page.tsx)の作成
```tsx
export default async function Page() {
  return (
    <>
      <h1>Sign in</h1>
      <a href="/login/github">Sign in with GitHub</a>
    </>
  );
}
```
[src/app/page.tsx](https://github.com/R1013-T/Next.js-LuciaAuth-Prisma-Supabase/blob/main/src/app/page.tsx)も必要に応じて編集してください。

## 🏃‍♂️ 6. 実行
以下のコマンドを実行して、アプリケーションを起動し、`http://localhost:3000/login`にアクセスしてください。
```bash
pnpm dev
```
`Sign in with GitHub`をクリックして、GitHub認証を試してみてください。

## 🎉 7. まとめ
この記事では、[Next.js](https://nextjs.org/)、[Lucia Auth](https://lucia-auth.com/)、[Prisma](https://www.prisma.io/)、[Supabase](https://supabase.com/)を使用してGitHub認証を実装する方法を紹介しました。[Lucia Auth](https://lucia-auth.com/)の柔軟性と型安全性を活かしつつ、モダンなスタックで堅牢な認証システムを構築できることが分かりました。


::::details コラム「他の認証ライブラリとの比較」
Lucia Authと他の人気のある認証ライブラリを比較してみましょう。

- Clerk
特徴: フルマネージドの認証・ユーザー管理サービス
利点: 簡単に実装でき、UIコンポーネントも提供
違い: Luciaはより低レベルなAPIを提供し、カスタマイズ性が高いが、実装の手間は増える

- Auth.js (NextAuth.js):
特徴: Next.jsに特化、多数のプロバイダーをサポート
利点: Next.jsプロジェクトとの統合が容易
違い: Luciaはフレームワーク非依存で、より細かい制御が可能

**Clerk**: 迅速な開発と豊富な機能が必要な場合
**Auth.js**: Next.jsプロジェクトで簡単に認証を実装したい場合
**Lucia**: 細かい制御と高度なカスタマイズが必要な場合

Supabase等のBasSを使用する場合、その[認証機能](https://supabase.com/docs/guides/auth)を利用するのでも良いでしょう。
::::