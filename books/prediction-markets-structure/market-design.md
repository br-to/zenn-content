---
title: "第5章 市場設計"
---

# 予測市場の本質は設計にある

予測市場を理解するとき、つい「どんなテーマを扱っているか」に目が行きがちです。でも本質はそれ以上に、どう設計されているかにあるんですよね。

同じイベントでも、市場の作り方次第で価格の質は大きく変わります。

# CLOB

CLOB は Central Limit Order Book の略で、買い注文と売り注文を板に並べて、価格と数量に応じてマッチングさせる方式です。

一般的な金融市場に近いこの方式の利点は、価格形成が比較的明示的であることです。どの価格にどれだけの注文があるかが見えやすくて、流動性の厚みやスプレッドも把握しやすい特徴があります。

# AMM

AMM は Automated Market Maker です。注文板の代わりに、あらかじめ決められた数式に従って価格を更新する仕組みです。

参加者が少ないときに初期流動性を確保しやすいというメリットがあります。DeFiでよく見るUniswapみたいな仕組みですね。

# Market Scoring Rule

予測市場の市場設計を語るうえで、market scoring rule、特に LMSR は重要です。

板を常時維持するのが難しい場面でも市場を成立させやすくて、学術的には非常に大きな役割を果たしました。Robin Hanson が提唱したこの仕組みは、予測市場の設計思想に大きな影響を与えています。

# 薄い市場で何が起こるか

薄い市場では、スプレッドが広がって、小さな取引で価格が飛んで、操作が容易になります。

「価格が出ている」ことと「その価格に意味がある」ことは別なんですよね。設計がどれだけ優れていても、流動性が足りなければ価格は信頼できるシグナルにならない。

## 参考URL

- Polymarket Prices & Orderbook  
  https://docs.polymarket.com/concepts/prices-orderbook
- Robin Hanson, Logarithmic Market Scoring Rules for Modular Combinatorial Information Aggregation  
  https://mason.gmu.edu/~rhanson/mktscore.pdf
