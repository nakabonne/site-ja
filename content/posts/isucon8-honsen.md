---
title: "isucon8本戦で惨敗したからせめて良いブログを書く"
description: "ISUCON8の本戦に、メルカリで一緒だった"
date: 2018-10-21
tags: ["参加記"]
draft: false
---

ISUCON8の本戦に、メルカリで一緒だった[@zaq1tomo](https://twitter.com/zaq1tomo)と[@inatonix](https://twitter.com/inatonix)と`zin-gonic`というチーム名で出場しました。  
結果は学生9位、全体で19位だったので、やったこととかまとめます。  
予選エントリはこちらです↓


{{< hatenaembed src="https://hatenablog-parts.com/embed?url=https://ja.nakabonne.dev/isucon8-yosen" >}}

# やったこと

チューニング対象は、isucoinという取引所アプリでした。  
言語は予選と同じGoです。  

[![nakabonne/isucon8-final - GitHub](https://gh-card.dev/repos/nakabonne/isucon8-final.svg?fullname=)](https://github.com/nakabonne/isucon8-final)


担当も予選と同じです。  

@nakabonne: インフラ担当  
@zaq1tomo: アプリ担当  
@inatonix: アプリ担当  

### 序盤

##### @nakabonne

序盤は主にプロファイリングの準備です。  

- ssh公開鍵登録
- [dotfile](https://github.com/nakabonne/dotfiles-ubuntu)をclone
- ベンチ走らせて[htop](https://github.com/hishamhm/htop)でリソース確認
- [デプロイスクリプト](https://github.com/nakabonne/isucon8-final/blob/master/scripts/restart.sh)の準備
- スロークエリログを仕込む
- アクセスログのプロファイルングツール[alp](https://github.com/tkuchiki/alp)を仕込む
- [ログローテートスクリプト](https://github.com/nakabonne/isucon8-final/blob/master/scripts/rotate_alplog.sh)を仕込む
- nginxのアクセスログを[LTSV形式](https://github.com/nakabonne/isucon8-final/commit/8c65354ed3f7bdd5bf309117ab469202dce83012)に

今回はdockerでデーモン化されていたため、パフォーマンスが少し気になる上にプロファイリングが少し面倒でした。  
当初はdockerを剥がしてsystemdに乗せ換えようとしましたが、工数に対するインパクトが見込めず一旦後回しにしました。  
また、nginxの[公式イメージ](https://github.com/nginxinc/docker-nginx/blob/master/stable/alpine/Dockerfile#L135)を見ると標準出力をdocker logsに流しており、アクセスログがうまく拾えず若干詰まりました。



##### @inatonix, @zaq1tomo

- マニュアルとコード読んで仕様把握
- pprofでプロファイリング

#### 方針決め

各プロファイリング結果毎に見ていきます。

##### リソース

{{< figure src="/img/isucon8-htop.png" width="100%" height="auto">}}

実行時にCPU使用率は100%に張り付き、半分以上はmysqlが食っていました。  
メモリもギリギリで、こちらも同じく大半はmysqlのキャッシュによるものでした。  
この時点で早いうちにDBを外に切り出すことは決めました。

##### スロークエリ

{{< figure src="/img/slow-query.png" width="100%" height="auto">}}

全tradeをソートしてから取得するクエリがまず第一関門っぽかったので、初動は決まりました。  
二つ目もクエリ数減らせないか調査の必要がありそうだと話した気がします。


##### アクセスログ

{{< figure src="/img/access-log.png" width="100%" height="auto">}}

`POST /orders` `GET /info` から見ていく必要があるのは一目瞭然で、エンドポイントが少ないことから一筋縄ではいかない匂いがプンプンでした。


### 中盤

##### @nakabonne

- mysqlを別サーバーに移行
- nginx基本チューニング
- このクエリやばそうとかつぶやく

{{< figure src="/img/isucon8-htop2.png" width="100%" height="auto">}}

mysqlを切り出し、サーバー1をapp+nginxのみの構成にしたためCPUに半分くらい空きが出来ました。  
CPUがボトルになるまではしばらくappは一台で動かそうとなりました。

##### @inatonix, @zaq1tomo

- latest tradeから取得するフィールドをidのみにする[修正](https://github.com/nakabonne/isucon8-final/pull/2)

この@zaq1tomoの修正で***500点→3200点***に伸びました。

- trade取得時にLIMITかける[修正](https://github.com/nakabonne/isucon8-final/pull/3)

###### つかの間の1位

この修正で***5136点***まで伸びました。そしてこの瞬間、<b><span style="color: #1464b3">我々zin-gonicは1位</span></b>に躍り出ました。  
ただ、この時を最後に我々が笑顔になることはありませんでした。。

{{< figure src="/img/isucon8-ranking.png" width="100%" height="auto">}}


### 終盤

##### @nakabonne

- shareボタンを有効に、、

CPUに余裕が出来たところで、リソースを使い切りたいと思っていたところ、@inatonixが怪しいものを見つけました。  

```go
// TODO: trueにするとシェアボタンが有効になるが、アクセスが増えてヤバイので一旦falseにしておく
res["enable_share"] = false
```

どうやらシェアボタンを有効にすると取引成立時にシェアされてユーザーが流れ込み、一気に負荷があがるそうで、これだ！と有効にしました。  
が、`GET /info` でタイムアウトが頻発。  
DBサーバーを見てみるとCPUが使い切られていたので、ロックされてるのかな？と思いつつとりあえずこれはまだ早いと判断して、そっとブランチを切り替えました。  
講評を聞く限りここで負荷を上げても耐えられるようにすることが鍵だったらしいです。

- 悪あがき


{{< figure src="/img/isucon8-waruagaki.png" width="100%" height="auto">}}

このブランチ量を見れば悪あがきをしていたことが分かります。  
もちろん何も意味をなしていません。


##### @inatonix, @zaq1tomo

- `POST /orders` で呼ばれる `AddOrders()` というメソッドをなんとかしようと奮闘
- shareボタンを有効にしたときの`GET /Info`を直そうと奮闘

oh...

### 感想

問題作成していただいたDeNAさん、カヤックさん、主催のLINEさん、サーバー提供していただいたConoHaさん、本当にありがとうございました。  
今回の優勝者は学生であり、それを讃えている皆さんを見て、改めて最高の業界に来てしまったと感じました。  
もしisucon9があるならば、来年も学生枠使えるので（もはやそんな枠があるかは不明）出場したいと思います。

{{< figure src="/img/isucon8-cards.jpg" width="100%" height="auto">}}
