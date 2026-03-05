---
title: "x402でAIエージェントが価格比較して買い物するシステムを作った話"
emoji: "🛒"
type: "tech"
topics: ["x402", "blockchain", "ai", "agentic-commerce", "base"]
published: false
---

## はじめに

こんにちは！ブロックチェーン×AI Agentで自律経済圏を創るKomlock labでエンジニアをしている小原です。

「シャンプー買って」とAIに頼んだら、複数の店舗を横断して最安値を探して、ブロックチェーン上で決済して、勝手に注文してくれる。

そんな世界を実際に作ってみました。

使ったのはCoinbaseが開発した**x402**というプロトコル。HTTPの402ステータスコード（Payment Required）を使って、AIエージェントがWebリクエストの中で自動的に支払いを行う仕組みです。

今回はx402を使って「AIエージェントが2つのECサイトを比較して、安い方で購入する」システムを作ったので、その記録を残します。

:::message
Base Sepolia（テストネット）上で動作するPoCです。実際のお金は動いていません。
:::

## Agentic Commerceとは

最近、OpenAI × Stripeが**ACP（Agentic Commerce Protocol）**を発表し、ChatGPTの中で直接買い物ができるようになりました。

一方で、Coinbaseは**x402**というプロトコルでWeb3側からアプローチしています。

どちらも「AIエージェントが人間の代わりに買い物する」という同じゴールを目指していますが、決済の仕組みが違います。

- **ACP**: 既存のカード決済（Stripe）の上にAI層を載せる
- **x402**: ブロックチェーン上のステーブルコイン（USDC）で直接決済する

今回はx402を使います。

## アーキテクチャ

```
ユーザー: 「シャンプー買って」
        ↓
AIエージェント
  ├─ Store A に商品検索（無料）
  ├─ Store B に商品検索（無料）
  ├─ 価格を比較 → 最安の店を選択
  └─ 最安の店で購入（x402: USDC決済）
        ↓
ECサーバー
  ├─ 402 Payment Required を返す
  ├─ エージェントが署名付き支払いを送信
  ├─ Facilitatorがオンチェーンで決済を検証
  └─ 注文確定
        ↓
ユーザー: 「ボタニカルシャンプー、Store Aで購入しました（$0.001）」
          BaseScan: https://sepolia.basescan.org/tx/0x...
```

## 技術スタック

- **ECサーバー**: Express.js + @x402/express
- **エージェント**: @x402/fetch + viem
- **決済**: x402 protocol（USDC on Base Sepolia）
- **Facilitator**: https://x402.org/facilitator

## ECサーバーを作る

まずはエージェントが買い物する先のECサーバーを作ります。

### プロジェクトのセットアップ

```bash
mkdir agent-commerce && cd agent-commerce
pnpm init
pnpm add express dotenv @x402/express @x402/evm @x402/core
pnpm add -D tsx typescript @types/express
```

### 商品データ

日用品を6つ用意しました。テストネットなので価格は$0.001程度にしています。

```json:server/products.json
[
  {
    "id": "shampoo-001",
    "name": "ボタニカル シャンプー 500ml",
    "description": "オーガニック成分配合、ノンシリコン、ラベンダーの香り",
    "price": "$0.001",
    "currency": "USDC",
    "stock": 40,
    "category": "haircare"
  },
  {
    "id": "shampoo-002",
    "name": "スカルプケア シャンプー 400ml",
    "description": "メントール配合、頭皮の脂をしっかり落とす、メンズ向け",
    "price": "$0.001",
    "currency": "USDC",
    "stock": 35,
    "category": "haircare"
  }
]
```

（他にも洗剤、ティッシュ、ボディソープなど全6商品）

### ECサーバーの実装

x402のポイントは`paymentMiddleware`です。購入エンドポイントにアクセスすると、まず402（Payment Required）が返ります。エージェントが支払い署名を添えてリトライすると、注文が確定します。

```typescript:server/index.ts
import { config } from "dotenv";
import express from "express";
import { paymentMiddleware, x402ResourceServer } from "@x402/express";
import { ExactEvmScheme } from "@x402/evm/exact/server";
import { HTTPFacilitatorClient } from "@x402/core/server";
import { readFileSync } from "fs";
import { fileURLToPath } from "url";
import { dirname, join } from "path";

config();

const __dirname = dirname(fileURLToPath(import.meta.url));
const products = JSON.parse(
  readFileSync(join(__dirname, "products.json"), "utf-8")
);

const payToAddress = process.env.PAY_TO_ADDRESS as `0x${string}`;
const facilitatorUrl =
  process.env.FACILITATOR_URL || "https://facilitator.x402.org";

const facilitatorClient = new HTTPFacilitatorClient({ url: facilitatorUrl });
const resourceServer = new x402ResourceServer(facilitatorClient).register(
  "eip155:84532",
  new ExactEvmScheme()
);

const app = express();
app.use(express.json());

// 商品一覧（無料）
app.get("/api/products", (_req, res) => {
  res.json({ products });
});

// 商品検索（無料）
app.get("/api/products/search", (req, res) => {
  const query = (req.query.q as string)?.toLowerCase() || "";
  const results = products.filter(
    (p: any) =>
      p.name.toLowerCase().includes(query) ||
      p.description.toLowerCase().includes(query)
  );
  res.json({ results });
});

// 購入エンドポイント（x402で有料）
const purchaseRoutes: Record<string, any> = {};
for (const product of products) {
  purchaseRoutes[`GET /api/purchase/${product.id}`] = {
    accepts: [
      {
        scheme: "exact" as const,
        price: product.price,
        network: "eip155:84532" as const,
        payTo: payToAddress,
      },
    ],
    description: `Purchase: ${product.name}`,
    mimeType: "application/json",
  };
}

app.use(paymentMiddleware(purchaseRoutes, resourceServer));

app.get("/api/purchase/:productId", (req, res) => {
  const product = products.find((p: any) => p.id === req.params.productId);
  if (!product) {
    return res.status(404).json({ error: "Product not found" });
  }

  const orderId = `ORD-${Date.now()}`;
  res.json({
    order: {
      orderId,
      product: product.name,
      price: product.price,
      status: "confirmed",
    },
  });
});

app.listen(4021, () => {
  console.log("Agent Commerce Server running at http://localhost:4021");
});
```

検索エンドポイントは無料、購入エンドポイントだけx402で課金される構成です。

## ショッピングエージェントを作る

次にエージェント側です。`@x402/fetch`を使うと、402レスポンスを受け取った時に自動的に署名付き支払いを行ってリトライしてくれます。

```bash
pnpm add @x402/fetch viem
```

```typescript:agent/index.ts
import { config } from "dotenv";
import { x402Client, wrapFetchWithPayment, x402HTTPClient } from "@x402/fetch";
import { ExactEvmScheme } from "@x402/evm/exact/client";
import { privateKeyToAccount } from "viem/accounts";

config();

const EC_SERVER_URL = process.env.EC_SERVER_URL || "http://localhost:4021";
const BASESCAN_URL = "https://sepolia.basescan.org/tx";

const agentAccount = privateKeyToAccount(
  process.env.AGENT_PRIVATE_KEY as `0x${string}`
);

const client = new x402Client();
client.register("eip155:*", new ExactEvmScheme(agentAccount));
const httpClient = new x402HTTPClient(client);
const fetchWithPayment = wrapFetchWithPayment(fetch, client);

async function searchProducts(query: string) {
  const res = await fetch(
    `${EC_SERVER_URL}/api/products/search?q=${encodeURIComponent(query)}`
  );
  const data = await res.json();
  return data.results;
}

async function purchaseProduct(productId: string) {
  const url = `${EC_SERVER_URL}/api/purchase/${productId}`;
  const res = await fetchWithPayment(url, { method: "GET" });

  if (!res.ok) throw new Error(`Purchase failed (${res.status})`);

  // オンチェーンの決済情報を取得
  let payment = null;
  try {
    payment = httpClient.getPaymentSettleResponse((name: string) =>
      res.headers.get(name)
    );
  } catch {}

  const body = await res.json();
  return { order: body.order, payment };
}
```

`wrapFetchWithPayment`が全部やってくれるのがx402の強みです。通常の`fetch`と同じように使えて、402が返ってきたら自動で支払い処理が走ります。

## 動かしてみる

```bash
# ターミナル1: ECサーバー起動
pnpm run server

# ターミナル2: エージェント実行
pnpm run agent -- シャンプー
```

```
[Agent] Wallet: 0x589A4572e1c2d90af913ec498774b4525D6B7d56

[User] "シャンプーを買って"
==================================================

[Agent] Step 1: Searching for "シャンプー"...
[Agent] Found 2 product(s):
  - ボタニカル シャンプー 500ml ($0.001) [shampoo-001]
  - スカルプケア シャンプー 400ml ($0.001) [shampoo-002]

[Agent] Step 2: Selected "ボタニカル シャンプー 500ml" ($0.001)

[Agent] Step 3: Purchasing via x402 payment...

[Agent] Purchase complete!
  Order ID: ORD-1772641982289
  Product:  ボタニカル シャンプー 500ml
  Price:    $0.001
  Status:   confirmed

[Payment] On-chain settlement:
  Tx Hash:  0x6a6a7a2c5d02c4639f28520ea17c04a7e5303957810f744eabad580733e5ecb0
  Network:  eip155:84532
  Payer:    0x589A4572e1c2d90af913ec498774b4525D6B7d56
  BaseScan: https://sepolia.basescan.org/tx/0x6a6a7a2c5d...
```

エージェントが「シャンプー」で検索し、商品を選んで、x402で決済し、注文が確定しました。BaseScanで実際にオンチェーンのトランザクションを確認できます。

## 複数店舗の価格比較

1つの店で買えるだけだと面白くないので、2つのECサーバー（Store A / Store B）に異なる価格を設定して、エージェントが最安値を比較して購入するようにしました。

```
「シャンプー」を2店舗で検索しました。最安値を比較します。

  ボタニカル シャンプー 500ml
    Store A: $0.001 [最安]  |  Store B: $0.0012
    → $0.0002 お得

  スカルプケア シャンプー 400ml
    Store A: $0.001  |  Store B: $0.0008 [最安]
    → $0.0002 お得
```

ボタニカルはStore A、スカルプケアはStore Bが安い。エージェントは自動的に安い方の店で購入します。

これがAmazon定期便との決定的な違いです。定期便は固定の店から固定の間隔で届くだけ。エージェントは必要な時に、最安の店から買ってくれます。

## x402の仕組み

x402の決済フローを整理するとこうなります。

```
1. Agent → EC Server: GET /api/purchase/shampoo-001
2. EC Server → Agent: 402 Payment Required（支払い条件を返す）
3. Agent → Facilitator: 署名付き支払いを送信
4. Facilitator → Base Sepolia: オンチェーンでUSDC送金を実行
5. Agent → EC Server: 支払い証明付きでリトライ
6. EC Server → Facilitator: 支払いを検証
7. EC Server → Agent: 200 OK（注文確定）
```

ポイントは、エージェントもECサーバーもFacilitator（x402.org）を信頼する構成になっていることです。Facilitatorがオンチェーンの決済を仲介し、検証します。

## ACP vs x402

Agentic Commerceのプロトコルは今、2つのアプローチが出てきています。

**ACP（Agentic Commerce Protocol）**はOpenAIとStripeが共同開発した仕様で、既存のカード決済インフラの上でエージェントが買い物する仕組みです。ChatGPTのInstant Checkout機能として実装されています。ただし現時点では米国限定です。

**x402**はCoinbaseが開発したプロトコルで、ブロックチェーン上のUSDCで直接決済します。HTTPの標準ステータスコード（402）を使っているのでWeb標準に沿っており、グローバルに使えます。

どちらが主流になるかはまだ分かりませんが、「規制型（ACP）」と「クリプトネイティブ（x402）」の2つの路線が並走している状況です。

## 作ってみて思ったこと

技術的にはちゃんと動きます。検索、比較、決済、オンチェーン証明まで全部繋がる。

ただ、実際にこれが普及するには**店舗側の対応**が必要です。今回は自分でECサーバーを作りましたが、現実のAmazonや楽天がエージェント向けのAPIを公開して、x402やACPに対応しないと、この仕組みは使えません。

また、日用品の自動補充をやろうとすると「いつ必要になったか」の検知が難しいという問題もあります。定期便のように固定間隔で買うならAmazonで十分。「本当に必要な時に自動で買う」を実現するには、検知の仕組みも含めてまだ課題が残っています。

とはいえ、インフラは揃いつつある。x402もACPもオープンソースで公開されていて、今すぐ試せる状態です。先にエージェント対応した店舗が、次の時代のSEOで勝つ可能性は十分あると思います。

## リポジトリ

https://github.com/br-to/agent-commerce

## まとめ

- x402を使ってAIエージェントが自律的に買い物するPoCを作った
- 複数店舗の価格比較 → 最安店で自動購入 → オンチェーン決済証明まで動く
- 現時点での課題は店舗側のAPI対応と、需要検知の仕組み
- ACP（Stripe + OpenAI）とx402（Coinbase）の2つのアプローチが並走中
- インフラは揃ってきている。あとは店舗が追いつくかどうか
