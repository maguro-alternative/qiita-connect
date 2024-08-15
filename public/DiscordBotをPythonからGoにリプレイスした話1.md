---
title: DiscordBotをPythonからGoにリプレイスした話1
tags:
  - Go
  - discord
private: true
updated_at: '2024-08-15T15:13:26+09:00'
id: 080deb431adeefc61080
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
こんにちは。社会人1年目のサーバーサイドエンジニアのマグロです。
コロナの熱にうなされながら書いてます。

今回はふんわりLTで発表した「DiscordBotをPythonからGoにリプレイスした話」をしていこうかと思います。

# bot概要
以下はリプレイス前のBotのコードです。

https://github.com/maguro-alternative/discordfast

機能は
- LINEとのメッセージ連携
- ボイスチャンネルの入退室通知
- Web版VOICEVOXによる読み上げ機能
- niconico,YouTubeのWebhook通知
- 上記を管理するadminページ

になります。

botは```pycord```、webは```FastAPI```&```jinja2```&```bootstrap```を採用しています。

# 負債一覧
- Pythonで導入しているパッケージが多い。
  - GitHubのスター数が少ないライブラリもあり、将来的に壊れる懸念があった。
- ```aiohttp```の使い方が正しくない。
  - リクエストするたびに```session```貼っていた(session貼っても使い回さない)ため、パフォーマンスがよろしくない。
- **データベースにのカラムに配列が存在する。**
- データベース操作もライブラリをラッパーしていて、不具合やパフォーマンスでの問題が多い。
- adminページのjsが複雑で読ませる気がない。
- APIの受け取り方法は```application/x-www-form-urlencoded```で、非常に複雑。
- webhookを取得するリクエストを1分に1回飛ばしており、ネットワークの負荷を上げていた。
  - レートリミットに引っかかることもあり、不具合や稼働率低下の一因にもなっていた。
- **テストコードがない。**

# やろうとした経緯
- 内定者インターンでGoを触る機会があり、Goで何かを作ってみたかった。
  - Goでのテストの書き方も教わった。
- 本番での不具合が多く、ステージングがないため確認のしようがなかった。
  - 用意しようにも1ヶ月3.5ドルの運用費がかかり、デプロイ先の無料枠5ドルを超えてしまう。
- デプロイ先での運用ルール変更。
  - 外部DBの使用が禁止される。
  - 新たにマイグレートするより設計し直したほうがよくね？となる。

- 負債一覧
- 設計
  - bot
  - dbtable
  - web
    - api
    - view
    - js
  - test

