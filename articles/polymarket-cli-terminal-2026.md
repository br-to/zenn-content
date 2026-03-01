---
title: "Polymarket CLIで予測市場のデータをターミナルから取得する"
emoji: "🔮"
type: "tech"
topics: ["polymarket", "cli", "blockchain", "rust", "予測市場"]
published: false
---

## はじめに

Polymarketは世界最大の予測市場プラットフォームです。「BTCは$100Kを超えるか」「日銀は利上げするか」といったイベントの確率がリアルタイムで取引されています。

このデータ、実はCLIからサクッと取得できます。Polymarket公式のRust製CLIツールを使えば、市場検索、オッズ確認、オーダーブック取得まで全部ターミナルで完結します。

今回は、Polymarket CLIのインストールから実際のデータ取得まで触ってみたので、その記録を残します。

## Polymarket CLIとは

Polymarket CLIは、Polymarketの予測市場データにターミナルからアクセスできるRust製のツールです。

主な機能は以下の通りです。

- 市場の検索・一覧表示
- オッズ（価格）のリアルタイム取得
- オーダーブック（板情報）の確認
- 価格履歴の取得
- ウォレット連携による取引（今回は扱いません）

GitHub: https://github.com/polymarket/polymarket-cli

## インストール

### Rustのインストール

Polymarket CLIはRust製なので、まずRustをインストールします。

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

### ソースからビルド

pre-builtバイナリも配布されていますが、環境によってはGLIBCのバージョン不整合で動かないことがあります（Debian 12ではGLIBC 2.36ですが、バイナリがGLIBC 2.38/2.39を要求します）。

ソースからビルドするのが確実です。

```bash
git clone https://github.com/polymarket/polymarket-cli.git
cd polymarket-cli
cargo build --release
```

ビルドには数分かかります。完了したらパスを通します。

```bash
sudo cp target/release/polymarket /usr/local/bin/
polymarket --version
```

```
polymarket 0.1.4
```

## 市場を検索する

### テーブル表示

`markets search` で市場を検索できます。

```bash
polymarket markets search "Bitcoin" --limit 5
```

```
╭───────────────────────────────────────────────────┬─────────────┬─────────┬───────────┬────────╮
│ Question                                          │ Price (Yes) │ Volume  │ Liquidity │ Status │
├───────────────────────────────────────────────────┼─────────────┼─────────┼───────────┼────────┤
│ Will Bitcoin reach $150,000 in February?          │ 0.05¢       │ $28.8M  │ $3.2M     │ Active │
│ Will Bitcoin reach $120,000 in February?          │ 0.05¢       │ $3.3M   │ $501.6K   │ Active │
│ Will Bitcoin reach $125,000 in February?          │ 0.05¢       │ $4.5M   │ $530.6K   │ Active │
│ Will Bitcoin reach $115,000 in February?          │ 0.05¢       │ $3.5M   │ $404.9K   │ Active │
│ Will Bitcoin dip to $55,000 in February?          │ 0.05¢       │ $7.3M   │ $285.5K   │ Active │
╰───────────────────────────────────────────────────┴─────────────┴─────────┴───────────┴────────╯
```

Price (Yes) がその市場の「Yes」の価格で、これがそのまま確率を表しています。0.05¢ = ほぼ0%、50¢ = 50%です。

### JSON出力

`-o json` をつけるとJSON形式で出力されます。スクリプトに組み込む時に便利です。

```bash
polymarket markets search "Bank of Japan" --limit 1 -o json
```

```json
[
  {
    "id": "1227992",
    "question": "Nothing Ever Happens: Interest Rates",
    "conditionId": "0xa1c7afb51...",
    "slug": "nothing-ever-happens-interest-rates",
    "endDate": "2026-03-20T00:00:00Z",
    "liquidity": "5864.88",
    "outcomePrices": "[\"0.925\",\"0.075\"]",
    "volume": "42958.01",
    "active": true
  }
]
```

## オーダーブック（板情報）を見る

CLOB（Central Limit Order Book）のデータにもアクセスできます。

```bash
polymarket clob book <TOKEN_ID>
```

```
Market: 0xbb57ccf5...
Last Trade: 0.515

Bids:
╭───────┬───────────╮
│ Price │ Size      │
├───────┼───────────┤
│ 0.001 │ 373060.37 │
│ 0.01  │ 1036542   │
│ 0.02  │ 105157    │
│ 0.1   │ 128.84    │
│ 0.15  │ 905       │
╰───────┴───────────╯
```

TOKEN_IDは市場検索のJSON出力から取得できます。`clobTokenIds`フィールドの値です。

## 価格履歴の取得

```bash
polymarket clob price-history <TOKEN_ID> --interval 1d --fidelity 7
```

`--interval` で期間（1m, 1h, 6h, 1d, 1w, max）、`--fidelity` でデータポイント数を指定できます。

## Pythonスクリプトと組み合わせる

CLIのJSON出力をPythonで処理すれば、オッズの変動監視ができます。

簡単な例として、市場のオッズを定期的に取得して変動を検知するスクリプトを書いてみました。

```python
import subprocess
import json

def search_markets(query, limit=5):
    result = subprocess.run(
        ["polymarket", "markets", "search", query,
         "--limit", str(limit), "-o", "json"],
        capture_output=True, text=True, timeout=15
    )
    return json.loads(result.stdout)

markets = search_markets("Bitcoin", limit=3)
for m in markets:
    prices = json.loads(m["outcomePrices"])
    yes_price = float(prices[0]) * 100
    print(f"{m['question']}: {yes_price:.1f}%")
```

```
Will Bitcoin reach $150,000 in February?: 0.1%
Will Bitcoin reach $120,000 in February?: 0.1%
Will Bitcoin reach $125,000 in February?: 0.1%
```

これをcronやAIエージェントのHEARTBEATに組み込めば、予測市場のオッズ変動を自動で監視できます。

## ハマりどころ

### GLIBC互換性

pre-builtバイナリはGLIBC 2.38以上を要求します。Debian 12（bookworm）のGLIBCは2.36なので、ソースビルドが必要でした。

```
./polymarket: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

この場合は前述の通り `cargo build --release` でビルドすれば解決します。

### データ取得は無料、取引にはウォレット設定が必要

市場データの取得（search, list, book等）にはウォレット不要です。Polymarket APIは公開されているので、認証なしでデータを取れます。

取引する場合はウォレットの設定とUSDC.eの準備が必要ですが、データソースとして使うだけならインストールしてすぐ使えます。

## まとめ

Polymarket CLIを使えば、予測市場のデータをターミナルからサクッと取得できます。

- 市場検索、オッズ確認、オーダーブック取得が全部コマンド一発
- JSON出力でスクリプトに組み込みやすい
- AIエージェントとの相性が良い（CLI = AIが一番扱いやすいインターフェース）
- データ取得だけなら無料、ウォレット不要

予測市場のオッズは「集合知による確率予測」です。ニュースの先行指標として使ったり、株やクリプトの投資判断の参考にしたり、賭けなくてもデータソースとしての価値があります。
