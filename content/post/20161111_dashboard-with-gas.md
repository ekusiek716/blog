---
date: 2016-11-11T04:42:33+09:00
title: "サーバレス・KPI分析ダッシュボードをGAS + Slackで"
draft: false
categories: ["Product"]
description: "集計・分析・チャート作成・Slack共有までをノンストップで定期処理する仕組みをGSheet + Google Apps Scriptで構築した。もちろんサーバレス・無料である。"
eyecatch: "/images/gas_ogp.png"
tags: ["Product"]
author: "Yamotty"
---

データはコミュニケーションツールである。<!--more-->

webサービスではGoogle Analyticsや、Big Query、その他数多くのデータ集計ツール上に集積しているログを都度取り出し、分析を行うことは一般的だ。特にプロダクトマネージャーはこれらのデータを加工し、コミュニケーション・ツールとして『良い塩梅に』使用することが求められる。しかしながらデータをチームに共有したり、毎日目に触れるようにするカロリーはいまだに高い。

Google AnalyticsやBig Queryからログを収集し・加工し、chartを作成、その上でchartをチームのコミュニケーションの場である「Slack」へポストする、ということを日常的に行っているわけだが、

- 毎日定点観測したい
- 都度SQLクエリを叩くのが手間
- データを取得しに行く場所が複数
などの要因から非常にカロリーが高い。

そこで、集計・分析・チャート作成・Slack共有までをノンストップで定期処理する仕組みをGSheet + Google Apps Scriptで構築した。もちろんサーバレス・無料である。

![完成形のイメージです。自動で最新のKPIチャートをSlackへ定期ポストし、Slackをダッシュボード化します](/images/gas_001.png)

## 事前準備
- Google AnalyticsやBig Queryへのアクセス権を管理者からもらっておく
- ここから投稿したいSlack TeamのSlack tokenを調べておく。xoxo-....という文字列のはず。
- （できれば）javascriptについての基礎的な文法を学ぶ

## 処理の流れ
以下のような手順で処理を実施する。

- Big Query（orGoogle Analytics） -> GSheetへ数字を更新する*3 ※
- GSheet上で任意の示唆を取り出すために、chartを作成する
- chartをSlackの特定のチャネルへポストする ※
- 1~3を束ねてトリガーを設定する ※

※Google Apps Scriptを用いて1, 3, 4の工程を自動化する。

## 工程1 / Big Query（or Google Analytics） -> GSheetへ数字を更新する
例えば、

- DAU
- RR
- FQ
など、毎日基礎的なデータを定点観測にBig Query上で毎日同じクエリを叩いている場合は以下のGoogle Apps Scriptを利用して自動化してしまうと良い（トレタさんありがとうございます🙏）。

[BigQueryを簡単にグラフにするGoogle Apps Script : TORETA（トレタ） ブログ](http://toreta.blog.jp/archives/20649904.html)

詳細は記事に預けるが、たったの2ステップでBig QueryからGSheetへのデータ更新を自動化できる。

記事を参考にGSheetを用意。Queriesというシートを作成し、シート名, クエリ, タイトルを記入する。

![記入例](/images/gas_002.png)

Google Apps Scriptを作成する。
- [Google SpreadsheetからBigQueryを呼び出すスクリプト · GitHub](https://gist.github.com/masuidrive/d8fddb0d78b259c1a440)
  - Big QueryのprojectIdを調べておく
  - GASおよびGCPのBigquery APIをEnableにしておく。

## 工程2 / GSheet上で任意の示唆を取り出すために、chartを作成する

GASのrunAllQueriesを実行すると、シートに記載されているSQLクエリが全て実行され、結果が各シートへ更新される。このデータを使ってchartを一度創っておくと、あとはデータが更新される度、chartの内容も最新状態に更新される。chart内のマージンや、色、データラベルの有無なども反映されるため、見やすいように始めに作り込んでおくと良い。

![チャート例](/images/gas_003.png)
ちなみに個人的には1枚のchartで多くの情報が得られるものが好き。サンプルのような多軸グラフや、バブルチャート、モンドリアンチャートがいい味を出してくれる。

## 工程3 / chartをSlackの特定のチャネルへポストする
[GASで作ったグラフをSlackに投稿する - Qiita](https://qiita.com/fumisoro/items/95114e1c5bed0cf4e4f7)


この記事を参考に作成した。この工程は「chartを画像化する」「Slackへ画像をポストする」という2つに分けられる。

実は「chartを画像化する」という機能を結構長い期間諦めていたのだが、Google Apps ScriptのSheet Classがデフォルトで持つgetChartsを使用することでいとも簡単にクリアできた。2つのchartを取得し、Slackへ送るケースでのサンプルコードを掲載しておく。

```
var slack = {
  postUrl : 'https://slack.com/api/files.upload',
  token : "xoxp-hogehoge",// Slackのtoken
  channelId : "blog" //投稿したいSlackのチャネル名
}
var uploadFile = function(data){
  UrlFetchApp.fetch(slack["postUrl"], {
    "method" : "post",
    "payload" : {
      token: slack["token"],
      file: data,
      channels: slack["channelId"]
    }
  });
}
/* 1. UU&PVを投稿 */
function postSlack_PVperUU() {
  var sheet = SpreadsheetApp.openById("hoge").getSheetByName("test");
  var chart = sheet.getCharts()[0];
  uploadFile(chart.getAs("image/png").setName("PV&UU.png"));
}
/* 2. Channel別UUを投稿 */
function postSlack_Channel() {
  var sheet = SpreadsheetApp.openById("hoge").getSheetByName("Channel");
  var chart = sheet.getCharts()[0];
  uploadFile(chart.getAs("image/png").setName("Channel.png"));
}

```

## 補足

- `openById`は開きたいGSheetのIDを引数に使用する。調べ方は簡単で、GSheetのURLの以下の囲い部分。
- `getSheetByName`にはその名の通りシート名を引数に入れてあげよう。
- `getAs`で画像のフォーマットを指定、setNameで画像名を指定する。

1~3を束ねてトリガーを設定するクエリの実行からchartをSlackへポストするところまでを一気通貫で実行できるよう、各functionを束ねよう。以下ではdoAllというfunctionで全て実行できるようにしてある。

```
/* GSheetのクエリを全て実行し、全chartをSlackへ投稿 */
function doAll(){
  runAllQueries(),
  postSlack_PVperUU(),
  postSlack_Channel()
}
```

あとはGoogle Apps Scriptのリソース > 現状のプロジェクトのトリガー よりdoAllを毎日定時に実行するよう設定することで、自動化完了。


出社の前後あたりに設定するとスマホで確認が完結し、大幅に生産性をUPできたりする。

## さいごに
この一連のすごいところは、『GSheetを介すことができればあらゆるチャートをSlack上で可視化できる』という点にある。

僕は妻とのメッセージも全てSlackに集約しているが、昨今流行するIoTなどのサービスからGSheetへ連携することができれば、以下のような体験も全て自動化することができるはずだ。

- 毎日の睡眠時間をSlackでチェックして、不足時には「寝ろ」アラートを上げる
- 歩数をSlackでチェックして、不足時には「運動しろ」アラートを上げる
- 家計の予実推移をSlackでチェックして、節約したり奮発したりできる

…あまりいい例が思い浮かばなかったが、もう少しいい感じにHackできるよ、というアドバイスが有ればぜひお待ちしております。
