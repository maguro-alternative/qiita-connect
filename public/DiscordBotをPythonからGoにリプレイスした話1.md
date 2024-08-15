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
- ステージングが欲しかった。
  - 本番での不具合が多く、ステージングがないため確認のしようがなかった。
  - 用意しようにも1ヶ月3.5ドルの運用費がかかり、デプロイ先の無料枠5ドルを超えてしまう。
- デプロイ先での運用ルール変更。
  - 外部DBの使用が禁止される。
  - 新たにマイグレートするより設計し直したほうがよくね？となる。

上記のような負債を考慮し、リファクタリングよりもリプレイスの方が工数が削減できると考え、リプレイスを決断しました。

# 設計
## botとwebの分離
まず真っ先に考えたのは、Botとadminページ(以降Web)の分離です。
リプレイス前も分離はしていたのですが、型定義やデータベース操作などの細かい部分まで区分していなかったため、曖昧な構成になっていました。

以下のように大雑把にディレクトリ分けをしました。

```
.
├── bot     // DiscordBotのディレクトリ
├── core    // main.goのディレクトリ
├── web     // web(api,view)のディレクトリ
├── go.mod
├── go.sum
└── README.md
```

## bot
### botのライブラリ
discordのライブラリとしてdiscordgoを採用しました。
採用理由はGoであればなんでも良かったので特にないです。

https://github.com/bwmarrin/discordgo

音声を扱うためdgvoiceも入れてます。

https://github.com/bwmarrin/dgvoice

### ディレクトリ構造(bot)
イベントとスラッシュコマンドの分離をしているだけです。
```
.
├── bot                         // DiscordBotを動かすためのディレクトリ
│   ├── cogs                    // DiscordBotのコグ
|   ├── commands                // スラッシュコマンド
│   ├── config                  // 環境変数設定ファイル
│   ├── ffmpeg                  // 動画、音声の変換
│   └── main.go
```

ファイルは以下のように配置します。
一機能につき一ファイルといった感じです。

```
├── bot
│   ├── cogs
│   │   ├── internal
│   │   │   └── entity.go
│   │   ├── cog_handler.go
│   │   ├── on_message_create.go
│   │   ├── on_message_create_test.go
│   │   ├── on_voice_state_update.go
│   │   └── on_voice_state_update_test.go
|   ├── commands
|   |   ├── command_handler.go
|   |   ├── command_handler_test.go
|   |   ├── ping_test.go
|   |   ├── ping.go
|   |   ├── voicevox_test.go
|   |   └── voicevox.go
│   ├── config
│   │   ├── internal
│   │   │   └── env.go
│   │   └── config.go
│   ├── ffmpeg
│   │   ├── ffmpeg_test.go
│   │   └── ffmpeg.go
│   └── main.go
```

## web
### webのライブラリ
フレームワークには頼らず、標準ライブラリの```net/http```と```html/template```で実装します。
ですがadminの権限があるか確認するために、OAuth2での認証が必要です。
認証情報の保存がうまくできるか不安だったので```sessions```を採用しました。

https://github.com/gorilla/sessions

API側のバリデーションチェックには```govalidator```を採用しています。

https://github.com/asaskevich/govalidator

### ディレクトリ構造(web)
```
├── web
│   ├── components              // Webサーバーのviewコンポーネント
│   ├── config                  // 環境変数設定ファイル
│   ├── handler                 // Webサーバーのハンドラ
│   ├── middleware              // Webサーバーのミドルウェア
│   ├── service                 // Webサーバーのサービス
│   ├── shared                  // Webサーバー内での共通のパッケージ
│   └── templates               // WebサーバーのHTMLテンプレート
```

```components```はviewページのコンポーネントを置いています。
以下のような認証情報の表示に使っています。
|before|after|
|---|---|
|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2853914/5e022626-e9ee-9b75-5b49-20c656cbf85c.png)|![image-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2853914/928843dc-d3cb-be32-2277-2aa27ceea136.png)|

```handler```はそれぞれのhttpパスの処理を書きます。
```middleware```は認証情報の確認やログをとるミドルウェアを置いています。
```service```はhttpパスで使用する構造体フィールドの宣言を置いてます。

  - bot
  - dbtable
  - web
    - api
    - view
    - js
  - test

