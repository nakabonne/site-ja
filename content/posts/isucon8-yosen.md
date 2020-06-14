---
title: "ISUCON8予選は学生枠6位で本戦出場が決まりました"
date: 2018-09-17
tags: ["参加記"]
draft: false
---

{{< hatenaembed src="https://hatenablog-parts.com/embed?url=http://isucon.net/archives/52459414.html" >}}


ISUCON8の予選に、メルカリで一緒だった[@zaq1tomo](https://twitter.com/zaq1tomo)と[@inatonix](https://twitter.com/inatonix)と`zin-gonic`というチーム名で出場しました。  
結果は学生枠6位だったので、やったこととかまとめます。  


# 事前準備

- 二度のisucon合宿を行った（どちらも環境構築で頓挫）
- 何回か集まってisucon7の過去問練習
- ishoconという[株式会社scouty](https://scouty.co.jp/)さん主催のコンテストで練習した
- hidakkathonという[株式会社会社サイバーエージェント](https://www.cyberagent.co.jp/)さん主催のコンテストで練習した
- 前日に当日と同じ要領でhidakkathonの問題をみんなでやった

とまあ初出場ということもあり結構ちゃんと準備して、何も言い訳できない状況だったのでとりあえず通過できてよかったです

# やったこと

チューニング対象は、イベントの座席予約アプリでした。  
言語は、3人の技術スタック的にGo一択でした。  

[![nakabonne/isucon8-qualify - GitHub](https://gh-card.dev/repos/nakabonne/isucon8-qualify.svg?fullname=)](https://github.com/nakabonne/isucon8-qualify)


@nakabonne: インフラ担当  
@zaq1tomo: アプリ担当  
@inatonix: アプリ担当  

### 序盤

##### @nakabonne

- ssh公開鍵登録
- [dotfile](https://github.com/nakabonne/dotfiles-ubuntu)をclone
- ベンチ走らせて[htop](https://github.com/hishamhm/htop)でリソース確認
- [リスタートスクリプト](https://github.com/nakabonne/isucon8-qualify/blob/master/scripts/restart.sh)の準備
- [スロークエリログ](https://github.com/nakabonne/isucon8-qualify/commit/916a01883ebe72977cd8adb9cd66de98aa0ed38a)を仕込む
- 再起動に備え、自動起動設定をする
- プロファイルングツール[alp](https://github.com/tkuchiki/alp)を仕込む
- [ログローテートスクリプト](https://github.com/nakabonne/isucon8-qualify/blob/master/scripts/rotate_alplog.sh)を仕込む
- alpに渡すためにh2oのアクセスログを[LTSV形式にする](https://github.com/nakabonne/isucon8-qualify/commit/1b30bd1b9d4b80158882ccceffa898cbefd2bd5e)

  
リバースプロキシは完全にnginxのつもりでいたので、h2oアクセスログをalpがパースできる形式で出力するのに結構時間かかってしまって、大迷惑かけてしまいました。  
本戦では爆速でやりたいです。  
ちなみに今回はnginxに差し替えてる方が結構いたので、インフラ担当としてその辺のトレードオフも見積もれるようになりたいです。

##### @inatonix, @zaq1tomo

- マニュアル読む
- ひたすらコード読む
- [pprof](https://golang.org/pkg/net/http/pprof/)を仕込んでボトルネック探す
- スロークエリログ見ながらボトルネック探す

#### 方針決め

##### リソース

{{< figure src="/img/isucon8-yosen-htop.png" width="100%" height="auto">}}

弱者の戦い方として、無理に複数台構成はしないと最初から決めていたため、ゴリゴリインメモリキャッシュを使おうと話していました。  
実行時にCPU使用率が100%に張り付くので、なるべくredis等の別プロセスは立ち上げたくないと感じたので、キャッシュするならGoプロセス管轄内のメモリに乗せようとなりました。  
ただ、mariadbが大分CPU食っているのでdbだけ別インスタンスに移したい気持ちはありました。

#### スロークエリ

{{< figure src="/img/isucon8-yosen-slow-query.png" width="100%" height="auto">}}

reservationsテーブルから全座席ごとにselectしてるクエリが37万回呼ばれていて、これを解消しない限り次に進まないことがすぐに分かりました。  
今回はボトルネックが明白だったため、学生でも迷わずに進める良問でした。

#### アクセスログ

{{< figure src="/img/isucon8-yosen-access-log.png" width="100%" height="auto">}}

一番上の `/api/users/:id` で、先程の重いクエリが流れていたため、平均9秒かかっていました。その他レスポンスタイム上位のエンドポイントはほとんど同じクエリがボトルになっていました。


### 中盤

##### @nakabonne

- 二人の修正が競合しないようにひたすらデプロイ
- `reservation`テーブルの`user_id`とかにindex張ってみる → 意味なし
- h2oでロードバランシングしてみる → 出来なかった

`reservations`テーブルには`event_id`と`canceled_at IS NULL`にマルチカラムインデックス張るのが正解だったっぽいです。  
単体テーブルへのクエリならまだいいのですが、内部結合となると実行計画が全く理解できず死亡しました。  

##### @inatonix, @zaq1tomo

- それぞれ別のアプローチでgetEventsクエリのN+1解消に取り組む
  - @inatonix: Goプロセス内インメモリキャッシュ
  - @zaq1tomo: クエリ変えてクエリ数減らす

僕のアクセスログ解析が遅れたせいもあり、全くスコアが上がらず完全にお通夜モードになっていた15時頃（だったかな）に、@zaq1timoの[N+1修正](https://github.com/nakabonne/isucon8-qualify/pull/4)が火を吹き1000点→10000点と10倍になりました。  
`/api/users/:id`の平均レスポンスタイムも9秒→1秒まで解消していました。

### 終盤

@zaq1tomoが合計料金をSUMするクエリを[チューニング](https://github.com/nakabonne/isucon8-qualify/pull/5)してくれてちょっとスコアが上がり、諸々のログを吐かないようにしたら13000点を超えるようになりました。  
ただ、ランダムにfailしたりしてハラハラしていました。  
競技30分前に再起動し最終的に14000点を超えたところでコードフリーズし、お片付けを始めました。

# 感想

1392名参加している大会で、障害等を何も起こさなかった運営の方々がかっこよすぎました。  
今回は各チームごとにベンチマークを用意してくれた（構成的にするしかなかった）Conohaさんにも感謝の念が溢れます。  
本戦勝つぞー
