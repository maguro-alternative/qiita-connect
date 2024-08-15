---
title: DiscordBotをPythonからGoにリプレイスした話1
tags:
  - Go
  - discord
private: true
updated_at: '2024-08-15T00:15:03+09:00'
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

# 負債一覧
- Pythonで導入しているパッケージが多い。
  - GitHubのスター数が少ないライブラリもあり、将来的に壊れる懸念があった。
- ```aiohttp```の使い方が正しくない。
  - リクエストするたびに```session```貼っていた(session貼っても使い回さない)ため、パフォーマンスがよろしくない。
- **データベースにのカラムに配列が存在する。**
- データベース操作もライブラリをラッパーしていて、不具合やパフォーマンスでの問題が多い。
- adminページのjsが複雑で読ませる気がない。
- APIの受け取り方法は```application/x-www-form-urlencoded```で、非常に複雑。
- **テストコードがない。**

# やろうとした経緯

- 負債一覧
- 設計
  - bot
  - dbtable
  - web
    - api
    - view
    - js
  - test

