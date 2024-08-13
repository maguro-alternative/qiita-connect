---
title: DiscordBotをPythonからGoにリプレイスした話1
tags:
  - Go
  - discord
private: true
updated_at: '2024-08-12T12:11:40+09:00'
id: 080deb431adeefc61080
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
こんにちは。社会人1年目のサーバーサイドエンジニアのマグロです。
コロナの熱にうなされながら書いてます。

今回はふんわりLTで発表した「DiscordBotをPythonからGoにリプレイスした話」をしていこうかと思います。
- bot概要
- やろうとした経緯
- 負債一覧
- 設計
  - bot
  - dbtable
  - web
    - api
    - view
    - js
  - test

