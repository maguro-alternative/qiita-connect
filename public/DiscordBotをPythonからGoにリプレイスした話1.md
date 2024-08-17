---
title: DiscordBotをPythonからGoにリプレイスした話
tags:
  - Go
  - discord
private: false
updated_at: '2024-08-16T00:06:11+09:00'
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

API部分ではjsonを扱わせたいため、formをjsonに変換するためjsを使用します。

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

```shared```ではweb内でのパッケージを置きます。
contextによる値の引渡しや、cookieに保存されている情報の保存や読み取りなどを行います。

## データベース操作
データベースにはpostgresを採用しています。
当初は各ディレクトリに```internal```を設置し、そこでデータベース操作をしようとしていました。
ですが、
- botとwebで同様の操作を行う部分が多い。
- テストする際にモック化が大変。
- テーブルに変更があった場合、影響を最小限に抑えやすくなる。

という点で```repository```ディレクトリを作成しました。

```
.
├── bot           // DiscordBotのディレクトリ
├── core          // main.goのディレクトリ
├── repository    // データベース操作のディレクトリ
├── web           // web(api,view)のディレクトリ
├── go.mod
├── go.sum
└── README.md
```

## repository
### repositoryのライブラリ
データベース操作にはsqlxを採用しました。
標準ライブラリからの拡張で、入出力で構造体を丸ごと指定できる点がいいと思い採用しました。
SQLを直書き出来て、何をしているのか分かり易いのも評価点です。

https://github.com/jmoiron/sqlx

## パッケージ(pkg)
botとwebで共通して使用するものを置きます。
暗号化やデータベース、LINE、YouTubeのAPIなどを置いています。
```
.
├── bot           // DiscordBotのディレクトリ
├── core          // main.goのディレクトリ
├── pkg           // 共通のパッケージ
├── repository    // データベース操作のディレクトリ
├── web           // web(api,view)のディレクトリ
├── go.mod
├── go.sum
└── README.md
```

## 周期処理(tasks)
discord.pyおよびpycordには定期処理用の```tasks.loop()```デコレータがありました。

https://discordpy.readthedocs.io/ja/latest/ext/tasks/index.html

discordgoには存在しないので、自作します。

```
.
├── bot           // DiscordBotのディレクトリ
├── core          // main.goのディレクトリ
├── pkg           // 共通のパッケージ
├── repository    // データベース操作のディレクトリ
├── tasks         // 定期的に行うタスク(Webhookの送信など)
├── web           // web(api,view)のディレクトリ
├── go.mod
├── go.sum
└── README.md
```

## テスト
ほぼほぼE2Eで行います。
Goのテストには基本に従いますが、値の比較を行うため```testify```を採用しました。

https://github.com/stretchr/testify

また、webでは一部jsを使用するため、```jest```を使用します。

### repositoryのテスト
想定通りにinsertやselectが出来ているか確認します。
流れは
- テスト開始
- トランザクションを貼る
- テスト処理
- ロールバック
- テスト終了

とすることでデータベースへの影響を気にせずにテストを行えます。
フィクスチャはいい感じなものがなさそうなので自作します。

https://engineering.mercari.com/blog/entry/20220411-42fc0ba69c/

https://speakerdeck.com/maguroalternative/golangnodetabesutesutohuikusutiyazuo-cheng

### botのテスト
discordgo側で各機能は動作検証されているので、モックでの結合テストが主になります。
詳しくは以下の記事をご覧ください。

https://zenn.dev/maguro_alterna/articles/6749101c15046f

### webのテスト
こちらも結合テストが主になります。

### testunit
上記のフィクスチャとモックを定義します。
```
.
├── bot           // DiscordBotのディレクトリ
├── core          // main.goのディレクトリ
├── pkg           // 共通のパッケージ
├── repository    // データベース操作のディレクトリ
├── tasks         // 定期的に行うタスク(Webhookの送信など)
├── testutil      // テスト用のユーティリティ
├── web           // web(api,view)のディレクトリ
├── go.mod
├── go.sum
└── README.md
```

## テーブル設計
以下のような配列のあるテーブルは徹底的に排除します。
```
CREATE TABLE IF NOT EXISTS guild_set_permissions (
    guild_id NUMERIC PRIMARY KEY,
    line_permission NUMERIC NOT NULL DEFAULT 8,
    line_user_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    line_role_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    line_bot_permission NUMERIC NOT NULL DEFAULT 8,
    line_bot_user_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    line_bot_role_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    vc_permission NUMERIC NOT NULL DEFAULT 8,
    vc_user_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    vc_role_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    webhook_permission NUMERIC NOT NULL DEFAULT 8,
    webhook_user_id_permission NUMERIC[] NOT NULL DEFAULT '{}',
    webhook_role_id_permission NUMERIC[] NOT NULL DEFAULT '{}'
);
```
上記のテーブルは以下のように3つに分割しました。
```
CREATE TABLE IF NOT EXISTS permissions_code (
    guild_id TEXT NOT NULL,
    type TEXT NOT NULL,
    code BIGINT NOT NULL,
    PRIMARY KEY(guild_id, type)
);
CREATE TABLE IF NOT EXISTS permissions_user_id (
    guild_id TEXT NOT NULL,
    type TEXT NOT NULL,
    user_id TEXT NOT NULL,
    permission TEXT NOT NULL,
    PRIMARY KEY(guild_id, type, user_id)
);
CREATE TABLE IF NOT EXISTS permissions_role_id (
    guild_id TEXT NOT NULL,
    type TEXT NOT NULL,
    role_id TEXT NOT NULL,
    permission TEXT NOT NULL,
    PRIMARY KEY(guild_id, type, role_id)
);
```

これでbotの全体の設計の説明は終了です。

# リプレイスによるメリット
## テストコードのおかげで手動テストがスムーズに
自動テストが通っても念の為手動テストも行なっていましたが、ほとんど想定通りの動作をしました。
リプレイス前は手動テストで手間をかける部分が多かったため、改善した点と言えます。

## テストコードのおかげでbotを起動させる手間も省けた
Botの挙動も自動テストすることで、手動テスト時の手間であるBot起動を省けました。
上記の通り、手動テストでも確認していましたが、自動テストと同じ結果が返ってきたためBotのテストも意味を成したと言えるでしょう。

|テストコード(PASS)|実際の出力|
|---|---|
|![スクリーンショット 2024-08-16 15.20.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2853914/125fbfb6-d2a8-08b7-3cc1-0d819bbb4549.png)|![スクリーンショット 2024-08-16 15.19.27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2853914/0fa271fd-f04f-ce2d-179a-d9d54d262134.png)|

## ランニングコストが1/3に
ランニングコストの大半はメモリが占めていました。
リプレイス前はPythonだったため、常に300MBほどのメモリを消費していたのですが、Goにリプレイスした結果、100MB以内に抑えることに成功しました。

![スクリーンショット 2024-08-16 15.24.30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2853914/d36bcbe7-688b-22f2-9677-13487abef367.png)

これにより、無料枠に収める場合難しかった、ステージングの用意も可能になりました！

## webページの動作速度も早くなった
リプレイス前はクエリやビューの最適化が成されておらず、ページの表示まで2~5秒ほどかかることがありました。
改善の余地はあるものの、リプレイス後は1~2秒ほどで表示されるようになりました。

# 今後の展開
## フロント部分の改善
流石にformをjsonに変換するのはキツイのでどうにかしたいです、、、

## repository
前述の通り当初はinternalに格納していました。
共通操作が多いためrepositoryに集約させましたが、逆に言うとそれ以外メリットがないので戻すのもありかなーと考えています。

## 終わりに
かなり根気のいるリプレイスでしたが、開発自体は楽しめました。
botやwebといった領域を意識することで、迷走することなく実装を進められたのが良かったと思っています。
書ききれていない部分もあり、もう少し続きを書いてもいいかなと考えてます。
