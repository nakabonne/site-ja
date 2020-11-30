---
title: "Goプロセスのランタイム統計を可視化する"
description: ""
date: 2020-11-01
tags: ["Go"]
draft: false
---

本記事は [Go 2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/go2) の1日目の記事です。

Goにはpprofをはじめとしたプロファイリングエコシステムが揃っています。しかしリソース使用量のトレンドをリアルタイムで監視したいだけのケースで手軽に扱えるツールが見つからなかったので、[gosivy](https://github.com/nakabonne/gosivy)というシンプルなTUIツールを作りました。

{{< figure src="/img/gosivy-demo.gif" width="100%" height="auto">}}

見つからなかったと書きましたが、正確には[statsviz](https://github.com/arl/statsviz)というWebベースの可視化ツールが[arl](https://github.com/arl)氏により先月公開されています。
非常に優れたツールで、対象アプリケーションに数行のコードを追加するだけでチャートを描画することができます。
仕組みがシンプルでUIも見やすくてとても良ツールなのですが、何点か気になる点がありました：

- チャートを閲覧してるかどうかに関わらず、対象プロセスで毎秒Stop the Worldを発生させてランタイム統計を計算している
- ランタイム統計の計算だけでなく可視化も対象のプロセス内で行っている
- 表示する情報が固定の割に情報量が多すぎる
- WebベースのUIなのでブラウザが必須

総じて、メトリクスを取りたい対象プロセスの負担が少し大きすぎるなと感じていました。

{{< tweet 1322084832831184896 >}}

そこでstatsvizの手軽さを残しつつ、監視対象プロセスの負担をなるべく減らしたツールを作ることにしました。

## 使い方
基本的な使い方をご紹介します。

まずはREADMEに従って、お好みの方法で `gosivy` バイナリをインストールします。

[![nakabonne/gosivy - GitHub](https://gh-card.dev/repos/nakabonne/gosivy.svg?fullname=)](https://github.com/nakabonne/gosivy)


次にメトリクスを可視化したいアプリケーションに `github.com/nakabonne/gosivy/agent` をimportし、以下のようにエージェントを起動するための処理を追加します。保存したらそのアプリケーションを実行してください。

```go
package main

import "github.com/nakabonne/gosivy/agent"

func main() {
	if err := agent.Listen(agent.Options{}); err != nil {
		panic(err)
	}
	defer agent.Close()
}
```

最後にgosivyプロセスを起動すれば、対象アプリケーションのメトリクスがリアルタイムで描画されます。gosivyプロセスは監視対象プロセスと同じホスト・ユーザーで実行する必要があります。

```
$ gosivy
```

しれっと引数無しでの実行を紹介しましたが、gosivyはエージェントが実行されているプロセスを自動で検出し、最初に見つけたプロセスを引数として実行します。そのため、エージェントが複数実行されている場合はまず `-l` オプションでPIDを確認します。

```console
$ gosivy -l
PID   Exec Path
15788 foo  /path/to/foo
16841 bar  /path/to/bar
```

そしてそれを引数として与える必要があります。
```
$ gosivy 15788
```

#### リモートモード
対象プロセスのアドレスさえ分かれば、簡単に別ホストから監視することができます。

エージェントがリッスンするアドレスを指定します。

```go
package main

import "github.com/nakabonne/gosivy/agent"

func main() {
	err := agent.Listen(agent.Options{
		Addr: ":9090",
	})
	if err != nil {
		panic(err)
	}
	defer agent.Close()
}
```

gosivyプロセスからアクセス可能なアドレスを引数として与えると、ローカルモードと同じように描画が始まるはずです。

```
$ gosivy host.xz:9090
```

## 可視化までの流れ

全体的なアーキテクチャは[gops](https://github.com/google/gops)を参考にしています。

{{< figure src="/img/gosivy-arch.jpg" width="100%" height="auto">}}

gosivyプロセスがエージェントに対してスクレイピングする形でメトリクスを収集しているため、gosivyプロセスを立ち上げない限り対象プロセスは何もしません。
つまりエージェントは、チャートを閲覧している時のみランタイム統計を計算するようになっています。

また、プロセス間はTCPソケットを介して通信しているため、ローカルホストもリモートホストも同じインターフェイスで通信することが可能です。


## 今後の展望
もともとこのツールは[ali](https://github.com/nakabonne/ali)というツールのデバッグが捗るようにと作った背景があり、正直必要な機能は揃っています。
しかし以前[Gophers Slack](https://gophers.slack.com/)でstatsvizの作者と少し話したところ、可視化とメトリクス収集を分けていることに将来性を感じてくれているようでした。
最近は[go-echarts](https://github.com/go-echarts/go-echarts)などのパワフルな可視化ライブラリが続々と登場していて、せっかく可視化処理を分離しているので様々な場所にプロット出来るようにする予定です。
