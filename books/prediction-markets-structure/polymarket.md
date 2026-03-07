---
title: "第6章 Polymarketの技術構造"
---

# Polymarketは何が新しかったのか

Polymarket の新しさは、予測市場を単なるWebアプリではなく、ブロックチェーンインフラとして再構成したことにあります。

ユーザーは将来事象に関する市場でポジションを取り、そのポジションは条件付きトークンとして扱われます。注文は署名ベースで出され、マッチングと決済は分離されます。この構造によって、予測市場は「ブラウザの中の賭け」ではなく、「プログラム可能な資産市場」に近づきました。

# ハイブリッド構造

Polymarket は、完全オンチェーン order book ではなく、オフチェーン注文とオンチェーン決済を組み合わせたハイブリッド構造で理解するとわかりやすいです。

# EIP-712署名

Polymarket を技術的に理解するうえでは、EIP-712 署名ベースの注文という視点が重要です。

# 条件付きトークン

Polymarket の核にあるのが、結果をトークン化するという発想です。YES / NO の結果ポジションを条件付きトークンとして扱うことで、予測市場のポジションを明確なデジタル資産として表現できます。

# CTF

Conditional Token Framework は、条件に応じて資産を分割・統合するための枠組みです。予測市場では結果ごとのポジションを作る土台になります。

# オラクルと解決

どれだけ美しく売買できても、最終結果をどう判定するかが曖昧なら市場は成立しません。取引の分散性と、解決の信頼性は別問題です。

# APIとデータ市場

Polymarket が開発者にとって面白いのは、予測市場がデータソースにもなっていることです。価格、出来高、約定、オッズ変化、マーケット一覧など、予測市場の情報はそのままアプリケーションの入力になります。

## 参考URL

- Polymarket Documentation
  https://docs.polymarket.com/
- Polymarket Trading Overview
  https://docs.polymarket.com/trading/overview
- Conditional Token Framework overview
  https://docs.polymarket.com/trading/ctf/overview
- Prices & Orderbook
  https://docs.polymarket.com/concepts/prices-orderbook
- Contract addresses
  https://docs.polymarket.com/developers/CLOB/introduction
