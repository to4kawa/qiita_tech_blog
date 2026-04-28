---
title: 今の仮想化：VMで止まっていた理解をモダン化する
tags: システム開発
author: toshikawa
slide: false
---
# 今の仮想化：VMで止まっていた理解をモダン化する

![今の仮想化](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130729/9b0e2a5b-6c2c-4111-b271-b628bd8a58c5.png)

## 1. はじめに

昔の自分は、仮想化といえば VM の話として理解していた。

物理サーバがあり、その上に Hypervisor があり、その上で VM が動く。  
VM の中には OS があり、その上でアプリケーションが動く。

この構成までは、だいたい説明できた。

```text
物理サーバ
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
App
```

ただ、最近の構成を見ていると、VM だけでは説明しきれない言葉が増えてくる。

Docker、containerd、Kubernetes、Service Mesh、mTLS。
それぞれの名前は聞いたことがある。Kubernetes も、コンテナを管理するものだという理解はある。

しかし、それらが全体のどの層にあり、何を担当していて、昔の VM 中心の理解とどうつながるのかは、少し曖昧だった。

この記事では、その曖昧さを整理する。

目的は、新しい概念を網羅的に説明することではない。
昔の VM 中心の仮想化理解を土台にして、今の構成ではその上に何が重なっているのかを、自分が説明できる形にすることだ。

今回扱う範囲は、アプリケーションが動く前提になる実行基盤と通信制御の層である。

ざっくり言えば、次のような範囲を扱う。

```text
物理サーバ / クラウド基盤
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Service Mesh
  ↓
App
```

ここで見たいのは、App や DB の中身ではない。

アプリケーションがどこで動いているのか。
コンテナは VM と何が違うのか。
Kubernetes はその中で何を管理しているのか。
Service Mesh は、なぜ Kubernetes の上で出てくるのか。

そのあたりを整理する。

逆に、今回は次の話には踏み込まない。

```text
- DB設計
- Webアプリケーションの設計
- Webサーバやリバースプロキシの詳細
- CI/CDの詳細
- Zero Trust全体の設計
- クラウドサービスごとの細かい実装差分
```

mTLS については触れる。
ただし、Zero Trust の全体像を説明するためではなく、Service Mesh がサービス間通信を守るときの具体例として扱う。

つまり、今回の中心はこうである。

```text
VMで止まっていた仮想化理解を、
コンテナ、Kubernetes、Service Mesh まで広げる。
```

この範囲に絞って、昔の理解から順番に積み上げていく。

---

## 2. 昔の仮想化理解：VM中心の構成

昔の仮想化理解は、かなりシンプルだった。

物理サーバの上に Hypervisor があり、その上で VM が動く。
VM は仮想的なサーバのように見え、その中に OS を入れて、アプリケーションを動かす。

図にすると、こうなる。

```text
物理サーバ
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
App
```

この構成では、VM が大きな単位になる。

1つの VM の中には、Guest OS があり、その上にアプリケーションやミドルウェアが入る。
別のアプリケーションを分けたい場合は、別の VM を用意する。

つまり、分離の単位は VM だった。

```text
物理サーバ
  ↓
Hypervisor
  ├─ VM A
  │   └─ Guest OS
  │       └─ App A
  │
  └─ VM B
      └─ Guest OS
          └─ App B
```

この理解は今でも重要だと思う。

Hypervisor は物理リソースを分け、VM はそれぞれ独立したサーバのように扱える。
そのため、OSごと分けたい場合や、強めの分離が必要な場合には、VM の考え方は今でも土台になる。

ただし、VM は OS ごと持つ。

そのため、アプリケーションごとに VM を増やすと、Guest OS もその分だけ増える。
これは分かりやすい一方で、起動や運用、リソース効率の面では重くなる。

そこで出てくるのが、コンテナである。

VM が「OSごと分ける」考え方だとすると、コンテナは「同じ OS の上で、アプリケーション実行環境を分ける」考え方に近い。

次は、VM とコンテナの違いを整理する。

---

## 3. VMとコンテナの違い

VM の次に出てくるのが、コンテナである。

VM は、物理サーバや仮想化基盤の上に、仮想的なサーバを作る。
その中に Guest OS を入れて、アプリケーションを動かす。

一方、コンテナは OS ごと仮想化するわけではない。

ホスト OS の上で、アプリケーションとその実行に必要なものをまとめて、分離された環境として動かす。

ざっくり図にすると、VM はこう。

```text
物理サーバ
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
App
```

コンテナはこう。

```text
物理サーバ / VM
  ↓
Host OS
  ↓
Container runtime
  ↓
Container
  ↓
App
```

ここで出てくる Container runtime が、コンテナを実際に動かす層である。
Docker や containerd がこのあたりに入る。

大事なのは、Hypervisor と Docker/containerd は同じ階層ではない、ということだ。

Hypervisor は VM を動かすための層。
Docker や containerd は、OS の上でコンテナを動かすための層。

つまり、こう見ると整理しやすい。

```text
Hypervisor
  → VM を動かす

Docker / containerd
  → コンテナを動かす
```

VM は OS ごと分ける。
コンテナは、OS は共有しつつ、アプリケーションの実行環境を分ける。

そのため、コンテナは VM より軽く扱いやすい。
アプリケーション単位で作り、配布し、起動しやすい。

ただし、コンテナは VM の完全な置き換えではない。

実際には、VM の上でコンテナを動かす構成も多い。

```text
物理サーバ
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
Container runtime
  ↓
Container
  ↓
App
```

クラウド上の Kubernetes でも、裏側では VM の上でコンテナが動いていることがある。

つまり、VM とコンテナは対立するものというより、役割が違う。

VM は、サーバ単位・OS単位で環境を分ける。
コンテナは、アプリケーション単位で実行環境を分ける。

ここまで来ると、次の疑問が出てくる。

コンテナが便利なのはわかった。
では、そのコンテナが大量に増えたとき、誰が配置し、起動し、止まったら再起動し、通信できるようにするのか。

そこで出てくるのが Kubernetes である。

---

## 4. Kubernetesは何をしているのか

コンテナは、アプリケーションを軽く分けて動かす仕組みとして便利である。

ただ、実際のシステムでは、コンテナは1個だけでは終わらない。

アプリケーション本体、API、バッチ処理、認証サービス、監視用の部品など、複数のコンテナが動く。
さらに、それらを複数台のサーバ上で動かすこともある。

そうなると、次のような問題が出てくる。

```text
- どのサーバで、どのコンテナを動かすのか
- コンテナが落ちたら、どう再起動するのか
- コンテナを何個動かすのか
- バージョンをどう更新するのか
- コンテナ同士をどう通信させるのか
- 外部からどうアクセスさせるのか
```

これを管理するのが Kubernetes である。

Kubernetes は、コンテナを直接1個ずつ手で起動するためのものではない。
複数のコンテナを、どこで、何個、どの状態で動かすかを管理するための基盤である。

ざっくり言えば、Kubernetes は **コンテナを運用するための管理レイヤー** である。

```text
物理サーバ / VM
  ↓
OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Pod / Container
  ↓
App
```

Kubernetes でよく出てくる単位に Pod がある。

Pod は、Kubernetes が扱う最小の実行単位である。
多くの場合、1つの Pod の中に1つのアプリケーションコンテナが入る。

```text
Pod
  └─ Container
      └─ App
```

場合によっては、1つの Pod の中に複数のコンテナが入ることもある。
ただし、最初の理解としては「Kubernetes はコンテナを直接ではなく、Pod という単位で扱う」と見ておくとよい。

Kubernetes には Deployment や Service という考え方もある。

Deployment は、「このアプリケーションを何個動かしたいか」「どのバージョンを動かしたいか」を管理する。
Service は、「動いている Pod に対して、安定した通信先を用意する」ための仕組みである。

ざっくり並べると、こうなる。

```text
Deployment
  → Pod の数や更新を管理する

Pod
  → コンテナを動かす単位

Service
  → Pod への通信先を安定させる
```

Kubernetes があることで、コンテナは単に「起動できるもの」から、「複数台にまたがって運用できるもの」になる。

VM の時代は、サーバ単位で環境を見ることが多かった。
Kubernetes では、アプリケーションを Pod として配置し、必要な数だけ動かし、落ちたら戻し、通信先を用意する。

つまり、Kubernetes はアプリケーションそのものではなく、アプリケーションを動かし続けるための土台である。

ただ、Kubernetes だけでサービス間通信の細かい制御がすべて終わるわけではない。

たとえば、サービス同士の通信を暗号化したい。
どのサービスがどのサービスへ通信してよいかを制御したい。
通信のメトリクスを細かく取りたい。
リトライやタイムアウトをアプリごとではなく、共通の仕組みとして扱いたい。

そこで出てくるのが Service Mesh である。

---

## 5. Service Meshは何をしているのか

Kubernetes を使うと、複数のコンテナを Pod として配置し、Service を通じて通信できるようになる。

ただ、システムが大きくなると、単に「通信できる」だけでは足りなくなる。

たとえば、次のようなことを考える必要が出てくる。

```text
- サービス間通信を暗号化したい
- どのサービスがどのサービスへ通信してよいか制御したい
- 通信が失敗したときにリトライしたい
- 応答が遅いときにタイムアウトさせたい
- 通信量やエラー率を観測したい
- 一部の通信だけ新しいバージョンへ流したい
```

これらをすべてアプリケーション側に実装すると、かなり大変になる。

アプリケーションごとに、認証、認可、リトライ、タイムアウト、ログ、メトリクスの実装が必要になる。
さらに、サービスが増えるほど、それぞれの実装差分も増えていく。

Service Mesh は、このサービス間通信の制御を、アプリケーションの外側で扱うための仕組みである。

典型的には、Pod の中にアプリケーションコンテナとは別に proxy を置く。

```text
Pod A
  ├─ App A
  └─ Proxy A

Pod B
  ├─ App B
  └─ Proxy B
```

この構成では、App A が App B に直接通信するのではなく、proxy を経由する。

```text
App A
  ↓
Proxy A
  ↓
Proxy B
  ↓
App B
```

この proxy が、サービス間通信の制御を担当する。

たとえば、通信の暗号化、認可、リトライ、タイムアウト、メトリクス収集などを、アプリケーションコードとは別の層で扱う。

ここでのポイントは、Service Mesh がアプリケーションそのものを動かす仕組みではない、ということだ。

Kubernetes は、Pod を配置し、動かし続ける。
Service Mesh は、その Pod やサービス同士の通信を制御・観測・保護する。

ざっくり分けると、こうなる。

```text
Kubernetes
  → コンテナを配置し、動かし続ける

Service Mesh
  → サービス間通信を制御・観測・保護する
```

Service Mesh の代表例には、Istio、Linkerd、Consul Connect などがある。
Kubernetes 環境では、Istio や Linkerd のように、サービス間通信を proxy 経由で扱う構成がよく出てくる。

この構成にすると、アプリケーションは本来の処理に集中しやすくなる。

通信の暗号化や認可、リトライ、タイムアウト、メトリクスといった横断的な処理を、Service Mesh 側に寄せられるからである。

ただし、Service Mesh を入れれば何でも簡単になるわけではない。

proxy が増えるため、構成は複雑になる。
通信経路も増える。
証明書やポリシーの管理も必要になる。

そのため、Service Mesh は小さな構成では過剰になることもある。

それでも、大きな Kubernetes 環境でサービス間通信を統一的に扱いたい場合には、Service Mesh が有効になる。

ここまで来ると、次に気になるのは「通信を守る」とは具体的に何をするのか、という点である。

その具体例として出てくるのが mTLS である。

---

## 6. Service Meshで通信を守るとはどういうことか

Service Mesh は、サービス間通信を制御・観測・保護する層である。

その中でも、セキュリティ寄りの話として重要なのが、「通信相手を確認する」という考え方である。

Kubernetes 上では、複数の Pod や Service が動く。
Service A が Service B に通信しているように見えても、セキュリティの観点では次のようなことを考える必要がある。

```text
- 通信先は本当に想定した Service B なのか
- 通信元は本当に許可された Service A なのか
- 通信内容は途中で読まれないか
- 通信内容は途中で改ざんされないか
- 許可されていないサービスが通信してこないか
```

単にネットワーク的につながるだけでは、十分ではない。

特に、サービスが増えてくると、「どのサービスが、どのサービスと通信してよいのか」を明確にする必要が出てくる。

たとえば、次のような構成があったとする。

```text
Frontend Service
  ↓
API Service
  ↓
Internal Service
```

このとき、Frontend Service から API Service への通信は許可したい。
API Service から Internal Service への通信も許可したい。

しかし、Frontend Service から Internal Service へ直接通信できる必要はないかもしれない。

```text
Frontend Service
  ↓ 許可
API Service
  ↓ 許可
Internal Service

Frontend Service
  ↓ 直接通信は不要
Internal Service
```

こうした通信関係を整理すると、サービス間通信は単なる接続ではなく、認証と認可の問題になる。

通信元が誰なのか。
通信先が誰なのか。
その通信は許可されているのか。

Service Mesh は、このサービス間通信の認証・認可・暗号化を、アプリケーションの外側で扱えるようにする。

その代表的な仕組みの一つが mTLS である。

mTLS を使うと、サービス同士が証明書を使ってお互いを確認できる。
つまり、通信する相手が本当に想定したサービスなのかを、通信の段階で確認する。

ここで重要なのは、mTLS が単なる暗号化だけではないことだ。

通常の TLS は、主にクライアントがサーバを確認する。
mTLS では、サーバもクライアントを確認する。

つまり、サービス間で「お互いに本人確認する」仕組みになる。

次は、この mTLS をもう少し具体的に見る。

---

## 7. mTLSはその具体例

mTLS は、mutual TLS の略である。

日本語では、相互 TLS や相互認証 TLS と呼ばれることがある。

通常の TLS では、主にクライアントがサーバを確認する。

たとえば、ブラウザで HTTPS のサイトにアクセスするとき、ブラウザはサーバ証明書を確認する。
これによって、「接続先のサーバが本物か」「通信が暗号化されているか」を確認する。

ざっくり書くと、こうなる。

```text
Client
  ↓ 接続
Server
  ↓ サーバ証明書を提示
Client が Server を確認する
```

一方、mTLS では、サーバだけでなくクライアント側も証明書を提示する。

```text
Client
  ↓ 接続
Server
  ↓ サーバ証明書を提示
Client が Server を確認する

Client
  ↓ クライアント証明書を提示
Server が Client を確認する
```

つまり、mTLS では、通信する両者がお互いを確認する。

```text
通常の TLS
  → Client が Server を確認する

mTLS
  → Client が Server を確認する
  → Server が Client を確認する
```

Service Mesh の文脈では、この仕組みをサービス間通信に使う。

たとえば、Service A が Service B に通信するとき、Service B は「本当に Service A から来た通信なのか」を確認できる。
Service A も、「本当に Service B に接続しているのか」を確認できる。

```text
Service A
  ↓ mTLS
Service B
```

Service Mesh では、この mTLS をアプリケーションコードではなく、proxy 側で扱える。

```text
App A
  ↓
Proxy A
  ↓ mTLS
Proxy B
  ↓
App B
```

App A と App B は、通常のアプリケーション処理に集中する。
その外側で、Proxy A と Proxy B が証明書を使って相互認証し、通信を保護する。

これにより、サービス間通信について次のようなことができる。

```text
- 通信元サービスを確認する
- 通信先サービスを確認する
- 通信を暗号化する
- 許可されたサービス間通信だけを通す
```

これは、APIキーやID/パスワードとは少し性質が違う。

APIキーは、アプリケーション層で扱う秘密情報である。
一方、mTLS は通信の段階で証明書を使って相手を確認する。

もちろん、mTLS も万能ではない。

証明書や秘密鍵をどう発行するか。
どう配布するか。
期限が切れたらどう更新するか。
漏えいした場合にどう失効するか。

こうした運用が必要になる。

そのため、mTLS は「入れれば終わり」の仕組みではない。
証明書管理やポリシー管理とセットで考える必要がある。

ただ、Service Mesh と組み合わせると、サービスごとに個別実装しなくても、サービス間通信の相互認証を共通の仕組みとして扱いやすくなる。

ここまでをまとめると、mTLS は Service Mesh が通信を守るときの代表的な具体例である。

Kubernetes はコンテナを動かし続ける。
Service Mesh はサービス間通信を制御・観測・保護する。
mTLS は、その中でサービス同士がお互いを証明書で確認するための仕組みである。

次は、ここまで出てきた層をまとめて、今の構成として図にしてみる。

---

## 8. 今の構成を図にする

ここまでで、VM、コンテナ、Kubernetes、Service Mesh、mTLS を順番に見てきた。

昔の理解では、仮想化は VM 中心だった。

```text
物理サーバ
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
App
```

今の構成では、この VM の上、または物理サーバ上に、さらにコンテナ実行基盤や Kubernetes、Service Mesh が重なる。

VM の上で Kubernetes を動かす場合は、ざっくりこう見える。

```text
物理サーバ / クラウド基盤
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Service Mesh
  ↓
App
```

全体図にすると、次のようなイメージになる。

![今の仮想化](./virtualization_technology_stack_comparison_diagram.png)

ここで、それぞれの役割を一言で置くと、こうなる。

```text
Hypervisor
  → VMを動かす

VM
  → OSごと分離された仮想サーバ

Guest OS
  → VMの中で動くOS

Container runtime
  → コンテナを動かす

Kubernetes
  → コンテナをPodとして配置し、動かし続ける

Service Mesh
  → サービス間通信を制御・観測・保護する

App
  → 実際のアプリケーション
```

つまり、昔の VM 理解に対して、今は次の層が追加されていると見るとわかりやすい。

```text
昔の理解:
  物理サーバ
    ↓
  Hypervisor
    ↓
  VM
    ↓
  Guest OS
    ↓
  App

今の理解:
  物理サーバ / クラウド基盤
    ↓
  Hypervisor
    ↓
  VM
    ↓
  Guest OS
    ↓
  Container runtime
    ↓
  Kubernetes
    ↓
  Service Mesh
    ↓
  App
```

ただし、必ず VM があるとは限らない。

ベアメタル Kubernetes のように、物理サーバ上の OS で直接 Kubernetes を動かす構成もある。

その場合は、こうなる。

```text
物理サーバ
  ↓
OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Service Mesh
  ↓
App
```

つまり、Hypervisor と VM は、構成によって入る場合と入らない場合がある。

一方で、コンテナを動かすなら Container runtime が必要になる。
Kubernetes は、そのコンテナを大量に扱うための管理レイヤーになる。
Service Mesh は、その上でサービス間通信を細かく制御したい場合に出てくる。

このように見ると、今の仮想化は単に「VMを作る話」ではない。

OSをどう分けるか。
アプリケーション実行環境をどう分けるか。
コンテナをどう配置するか。
サービス間通信をどう制御するか。

そうした複数の層が重なったものとして見たほうが理解しやすい。

次は、この構成をセキュリティ視点で見直してみる。

---

## 9. セキュリティ視点で見る

ここまでの構成を、セキュリティ視点で見ると少し整理しやすくなる。

それぞれの層は、単に「アプリケーションを動かすための部品」ではなく、分離や制御の単位にもなっている。

まず、Hypervisor は VM を分離する。

```text
Hypervisor
  → VMごとに環境を分ける
```

VM は、それぞれ独立したサーバのように扱える。
OS ごと分けられるので、アプリケーションやミドルウェアだけでなく、OS レベルでも環境を分けられる。

ただし、その分だけ重い。

次に、Container runtime はコンテナを分離する。

```text
Container runtime
  → コンテナごとにアプリケーション実行環境を分ける
```

コンテナは VM より軽く、アプリケーション単位で扱いやすい。
ただし、OS をまるごと分ける VM とは違い、ホスト OS の機能を使って分離している。

そのため、VM とコンテナは同じ種類の分離ではない。

VM は OS ごと分ける。
コンテナは OS 上でアプリケーション実行環境を分ける。

Kubernetes は、そのコンテナを運用するための制御面になる。

```text
Kubernetes
  → Podの配置、再起動、スケール、Service経由の通信を管理する
```

Kubernetes では、どの Pod をどこで動かすか、いくつ動かすか、どう更新するかを管理する。
また、ServiceAccount、Role、RoleBinding などを使って、Kubernetes 内の権限も管理する。

つまり Kubernetes は、単にコンテナを動かすだけではなく、「誰が何を操作できるか」「どのリソースをどう管理するか」という制御にも関わる。

Service Mesh は、サービス間通信の制御面になる。

```text
Service Mesh
  → サービス間通信を制御・観測・保護する
```

Kubernetes だけでも Service を使って通信はできる。
しかし、通信元と通信先の確認、通信の暗号化、サービス間の認可、リトライ、タイムアウト、メトリクス収集などを統一的に扱いたい場合、Service Mesh が出てくる。

mTLS は、その中でもサービス間通信を守る具体例である。

```text
mTLS
  → サービス同士が証明書で相互確認する
```

通常の TLS では、主にクライアントがサーバを確認する。
mTLS では、サーバもクライアントを確認する。

Service Mesh と組み合わせると、App A と App B が直接証明書処理を実装しなくても、Proxy A と Proxy B の間で相互認証を扱える。

```text
App A
  ↓
Proxy A
  ↓ mTLS
Proxy B
  ↓
App B
```

このように見ると、今の構成では、セキュリティ上の役割も層ごとに分かれている。

```text
Hypervisor
  → VMの分離

Container runtime
  → コンテナの分離

Kubernetes
  → コンテナ配置・運用・権限制御

Service Mesh
  → サービス間通信の制御・観測・保護

mTLS
  → サービス間の相互認証と暗号化
```

もちろん、これだけでセキュリティが完成するわけではない。

アプリケーション自体の認証認可、DB の権限管理、ネットワーク設計、CI/CD、脆弱性管理、ログ監視など、他にも考えることは多い。

ただし、今回の記事ではそこまでは扱わない。

ここで見たいのは、昔の VM 中心の理解に対して、今はどの層で何を分離し、どの層で何を制御しているのか、ということだった。

その意味では、今の仮想化は「VMを作る話」だけではなくなっている。

VM で OS を分ける。
コンテナでアプリケーション実行環境を分ける。
Kubernetes でコンテナを運用する。
Service Mesh でサービス間通信を制御する。
mTLS でサービス同士を相互確認する。

この重なりとして見ると、今の構成が少し見えやすくなる。

---

## 10. まとめ

昔の自分にとって、仮想化といえば VM だった。

物理サーバがあり、その上に Hypervisor があり、その上で VM が動く。
VM の中には Guest OS があり、その上でアプリケーションが動く。

この理解は、今でも間違っていない。

ただし、現在の構成を見ると、VM だけでは説明しきれない層が増えている。

コンテナは、OS 上でアプリケーション実行環境を分ける。
Kubernetes は、そのコンテナを Pod として配置し、動かし続ける。
Service Mesh は、サービス間通信を制御・観測・保護する。
mTLS は、その中でサービス同士を証明書で相互確認する。

ざっくり重ねると、こうなる。

```text
物理サーバ / クラウド基盤
  ↓
Hypervisor
  ↓
VM
  ↓
Guest OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Service Mesh
  ↓
App
```

もちろん、すべての環境がこの形になるわけではない。
ベアメタル Kubernetes のように、Hypervisor や VM を挟まない構成もある。

```text
物理サーバ
  ↓
OS
  ↓
Container runtime
  ↓
Kubernetes
  ↓
Service Mesh
  ↓
App
```

大事なのは、VM、コンテナ、Kubernetes、Service Mesh を同じものとして見ないことだと思う。

それぞれ役割が違う。

```text
VM
  → OSごと分ける

Container
  → アプリケーション実行環境を分ける

Kubernetes
  → コンテナを配置し、運用する

Service Mesh
  → サービス間通信を制御・観測・保護する

mTLS
  → サービス同士を証明書で相互確認する
```

つまり、今の仮想化は、単に「VMを作る話」ではない。

どの層で環境を分けるのか。
どの層でアプリケーションを動かし続けるのか。
どの層でサービス間通信を守るのか。

そうした複数の層を重ねて見る必要がある。

今回、自分の中では、昔の VM 中心の理解を土台にして、コンテナ、Kubernetes、Service Mesh までを同じ図の中に置けるようになった。

これで、「今の仮想化」が少し説明しやすくなった。

---

## 参考資料

この記事を書くにあたり、以下の公式ドキュメントを参考にした。

* [Microsoft Learn: Hyper-V virtualization in Windows Server and Windows](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/overview)
  Hyper-V と VM / Hypervisor の関係を確認するために参照。

* [Docker Docs: What is Docker?](https://docs.docker.com/engine/docker-overview/)
  Docker とコンテナの基本的な考え方を確認するために参照。

* [Microsoft Learn: Istio-based service mesh add-on for Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/istio-about)
  Service Mesh の役割、observability、traffic management、security の説明を確認するために参照。

* [Red Hat Documentation: About OpenShift Service Mesh](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.3/html-single/about/index)
  OpenShift Service Mesh における service-to-service authentication、metrics、monitoring などを確認するために参照。

* [Broadcom Knowledge Base: Virtual machine hardware versions](https://knowledge.broadcom.com/external/article/315655/virtual-machine-hardware-versions.html)
  VMware vSphere ESXi における VM と仮想ハードウェアの扱いを確認するために参照。
