---
title: "Polymarket CLIで予測市場のデータをターミナルから取得する"
emoji: "🔮"
type: "tech"
topics: ["polymarket", "cli", "blockchain", "rust", "予測市場"]
published: false
---

## はじめに

Polymarketは世界最大の分散型予測市場プラットフォームです。Polygon上に構築されていて、「BTCは今月$100Kを超えるか」「日銀は利上げするか」「次の米大統領は誰か」といったイベントの確率が、リアルタイムで取引されています。

2024年の米大統領選では累計取引量が$3.5Bを超え、従来の世論調査より正確な予測を出したことで注目を集めました。

このPolymarketのデータ、ブラウザからも見れますが、Rust製の公式CLIツールを使えばターミナルからサクッと取得できます。市場検索、オッズ確認、オーダーブック取得まで全部コマンド一発です。

今回はこのCLIのインストールからデータ取得まで触ってみたので、その記録を残します。

:::message
この記事ではデータ取得にフォーカスしています。CLIにはウォレット連携による取引機能もありますが、Polymarketでの取引は日本の法規制上グレーゾーンにあたるため、本記事では扱いません。データの閲覧・取得自体は公開APIを通じた情報収集であり、問題ありません。
:::

## Polymarket CLIとは

Polymarket CLIは、Polymarketの予測市場にターミナルからアクセスできるRust製のツールです。

https://github.com/polymarket/polymarket-cli

主な機能をカテゴリ別に整理します。

**データ取得（認証不要）**
- 市場の検索・一覧表示
- オッズ（価格）のリアルタイム取得
- オーダーブック（板情報）の確認
- 価格履歴の取得
- イベント・タグ・シリーズの取得
- コメントの閲覧
- リーダーボードの確認

**取引（ウォレット認証が必要）**
- 指値注文・成行注文
- ポジション管理
- CTF操作（split, merge, redeem）
- ブリッジ

データ取得だけならウォレット不要で、インストールしてすぐ使えます。

## インストール

### 方法1: pre-builtバイナリ（推奨環境のみ）

GitHubのReleasesページからバイナリをダウンロードできます。

```bash
# Linux x86_64の場合
curl -L https://github.com/polymarket/polymarket-cli/releases/latest/download/polymarket-linux-amd64 -o polymarket
chmod +x polymarket
sudo mv polymarket /usr/local/bin/
```

ただし、このバイナリはGLIBC 2.38以上を要求します。Ubuntu 24.04やFedora 39以降なら問題ありませんが、Debian 12（GLIBC 2.36）やそれ以前の環境では以下のエラーが出ます。

```
./polymarket: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

### 方法2: ソースからビルド（確実）

GLIBC問題を回避するには、ソースからビルドします。

```bash
# Rustのインストール（未インストールの場合）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# ソースからビルド
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

ビルドに必要なディスク容量は約1GBです。CIやDockerで使う場合はマルチステージビルドで最終イメージを小さくするとよいでしょう。

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

各カラムの意味は以下の通りです。

- **Price (Yes)**: Yes側のオッズ。これがそのまま市場が予測する確率を表します。50¢ = 50%、99¢ = 99%
- **Volume**: 累計取引量（USDC建て）。市場の注目度の指標
- **Liquidity**: 現在の流動性。高いほど大きなサイズの注文が通りやすい
- **Status**: Active（取引中）/ Closed（終了）

### JSON出力

`-o json` をつけるとJSON形式で出力されます。プログラムから扱う場合に便利です。

```bash
polymarket markets search "Bank of Japan" --limit 1 -o json
```

```json
[
  {
    "id": "1227992",
    "question": "Nothing Ever Happens: Interest Rates",
    "conditionId": "0xa1c7afb51e4a...",
    "slug": "nothing-ever-happens-interest-rates",
    "endDate": "2026-03-20T00:00:00Z",
    "liquidity": "5864.88",
    "outcomes": "[\"Yes\",\"No\"]",
    "outcomePrices": "[\"0.925\",\"0.075\"]",
    "volume": "42958.01",
    "active": true
  }
]
```

`outcomePrices` がオッズです。`[0.925, 0.075]` は Yes: 92.5%, No: 7.5% を意味します。

### 市場の一覧

検索以外に、条件を指定して市場を一覧取得することもできます。

```bash
# アクティブな市場を流動性順で取得
polymarket markets list --active --limit 10
```

## オーダーブック（板情報）を見る

Polymarketの裏側にはCLOB（Central Limit Order Book）があります。株式市場の板情報と同じように、各価格帯の注文状況を確認できます。

```bash
polymarket clob book <TOKEN_ID>
```

TOKEN_IDは市場検索のJSON出力から取得できます。各市場にはYes/Noそれぞれのトークンがあり、`clobTokenIds` フィールドに格納されています。

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

Sizeの単位はシェア数です。Bids（買い注文）とAsks（売り注文）がそれぞれ表示されます。

板が厚い（Sizeが大きい）価格帯は、多くのトレーダーがその価格が妥当だと考えていることを示します。逆に板が薄い価格帯をまたぐような動きがあれば、市場の見方が大きく変わったサインです。

## その他のデータ取得コマンド

### 価格関連

```bash
# 現在の価格（ミッドポイント）
polymarket clob midpoint <TOKEN_ID>

# Bid-Askスプレッド
polymarket clob spread <TOKEN_ID>

# 直近の取引価格
polymarket clob last-trade <TOKEN_ID>

# 価格履歴（期間: 1m, 1h, 6h, 1d, 1w, max）
polymarket clob price-history <TOKEN_ID> --interval 1d --fidelity 7
```

### イベント・タグ

```bash
# イベント単位で取得（1つのイベントに複数市場が紐づく）
polymarket events list --limit 5

# タグ一覧
polymarket tags list
```

### リーダーボード

```bash
# トップトレーダーのランキング
polymarket data leaderboard
```

## ユースケース

CLIでデータを取得できると、様々な活用ができます。

### 1. オッズ変動の監視

定期的にオッズを取得して、前回との差分が閾値を超えたらアラートを出す仕組みが作れます。予測市場のオッズが急変動する時は、何か大きなニュースが出たサインです。ニュースサイトより先にシグナルを掴める可能性があります。

### 2. 投資判断の補助データ

「トランプ関税発動確率が急上昇」→ 関連銘柄のポジション調整、「Fed利下げ確率が上昇」→ テック株への影響を検討、といった使い方ができます。予測市場のオッズは多くのトレーダーの合意価格なので、単一のアナリストの予測より信頼性が高いケースもあります。

### 3. AIエージェントとの連携

CLIはAIエージェントと非常に相性が良いです。ブラウザ操作やAPI認証の設定が不要で、コマンド1つでデータが取れるため、エージェントのツールとして簡単に組み込めます。

例えば「日銀の利上げ確率が5%以上変動したら通知」「BTCの予測市場と実際の価格の乖離を検知」といった自律的な監視をエージェントに任せることができます。

### 4. データ分析・可視化

JSON出力をPythonやRのスクリプトに渡して、オッズの推移グラフを作ったり、複数市場の相関分析を行ったりできます。学術研究やレポート作成にも使えます。

## ハマりどころ

### GLIBC互換性（再掲）

前述の通り、pre-builtバイナリはGLIBC 2.38以上が必要です。VPSやCI環境ではOSのバージョンに注意してください。ソースビルドで回避できます。

### 市場のslugとconditionIdとtokenId

Polymarketには市場を識別するIDが複数あります。

- **slug**: URLに使われる人間が読める識別子（例: `nothing-ever-happens-interest-rates`）
- **conditionId**: 市場のコントラクト上の識別子（0xから始まるハッシュ）
- **tokenId**: Yes/Noそれぞれのトークンの識別子（長い数値文字列）

CLIのコマンドによって必要なIDが異なります。`markets` 系コマンドはslugやconditionIdを使い、`clob` 系コマンドはtokenIdを使います。JSON出力で各IDを確認できるので、最初にJSON形式で市場情報を取得しておくとスムーズです。

### レート制限

Polymarket APIには明示的なレート制限は公開されていませんが、短時間に大量のリクエストを送ると429エラーが返ることがあります。スクリプトで定期実行する場合は適度な間隔（数秒〜数分）を空けるようにしましょう。

## まとめ

Polymarket CLIを使えば、予測市場のデータをターミナルからサクッと取得できます。

- **インストール**: Rustソースビルドが確実。GLIBC問題に注意
- **データ取得**: 市場検索、オッズ、オーダーブック、価格履歴がコマンド一発
- **出力形式**: テーブルとJSONを切り替えられ、スクリプトとの連携が容易
- **ウォレット不要**: データ取得だけなら認証なしで使える

予測市場のオッズは「世界中のトレーダーによるリアルタイムの確率予測」です。賭けなくても、データソースとして十分な価値があります。

次回はこのCLIとPythonを組み合わせて、オッズ変動の自動監視スクリプトを作ってみたいと思います。
