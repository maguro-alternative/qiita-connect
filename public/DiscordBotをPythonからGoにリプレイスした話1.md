---
title: DiscordBotをPythonからGoにリプレイスした話1
tags:
  - Go
  - discord
private: true
updated_at: '2024-08-14T09:41:43+09:00'
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
- **データベースにのカラムに配列は存在する。**
- adminページのjsが複雑で読ませる気がない。

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

