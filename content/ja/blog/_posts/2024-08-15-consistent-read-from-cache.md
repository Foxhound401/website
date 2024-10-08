---
layout: blog
title: 'Kubernetes v1.31: キャッシュからの整合性のある読み込みによるクラスターパフォーマンスの向上'
date: 2024-08-15
slug: consistent-read-from-cache-beta
author: >
  Marek Siarkowicz (Google)
translator: >
  [ntkm61027](https://github.com/ntkm61027) ([3-shake](https://3-shake.com/))
---

Kubernetesはコンテナ化されたアプリケーションの堅牢なオーケストレーションで知られていますが、クラスターの規模が拡大するにつれて、コントロールプレーンへの負荷がボトルネックとなる可能性があります。
特に大きな課題となっていたのは、etcdデータストアからのデータ読み込みの厳密な整合性を保証することです。
これを実現するには、リソースを大量に消費するクォーラム読み込みが必要でした。

本日、Kubernetesコミュニティは、大きな改善を発表できることを嬉しく思います。
Kubernetes v1.31において、「キャッシュからの整合性のある読み込み」がベータ版に移行しました。

### なぜ整合性のある読み込みが重要なのか

Kubernetes コンポーネントがクラスターの最新状態を正確に把握するためには、整合性のある読み込みが不可欠です。
整合性のある読み込みを保証することで、Kubernetesの操作の正確性と信頼性が維持され、各コンポーネントは最新の情報に基づいて適切な判断を下すことができます。
しかし、大規模なクラスターでは、そのためのデータの取得と処理がパフォーマンスのボトルネックとなるおそれがあります。
特に、結果のフィルタリングを伴うリクエストでこの問題が顕著になります。
Kubernetesはetcd内で名前空間ごとにデータを直接フィルタリングできますが、ラベルやフィールドセレクタによるその他のフィルタリングでは、データセット全体をetcdから取得し、Kubernetes APIサーバーがメモリ上でフィルタリングを行う必要があります。
この問題は、特にkubeletなどのコンポーネントに大きな影響を与えます。
kubeletは自身のノードにスケジュールされたPodのみをリストするだけで足りるところを、これまでの仕組みでは、APIサーバーとetcdがクラスター内のすべてのPodを処理する必要がありました。

### ブレイクスルー: 信頼性の高いキャッシング

Kubernetesは、読み込み操作を最適化するために、以前からWatchキャッシュを使用してきました。
Watchキャッシュはクラスターの状態のスナップショットを保存し、etcdのWatchを通じて更新情報を受け取ります。
しかし、これまではキャッシュが完全に最新の状態であることを保証できなかったため、整合性のある読み込みを直接提供することができませんでした。

「キャッシュからの整合性のある読み込み」機能は、etcdの[進捗通知](https://etcd.io/docs/v3.5/dev-guide/interacting_v3/#watch-progress)のメカニズムを活用してこの問題に対処します。 
この通知により、Watchキャッシュは自身とetcdを比較し、データが最新かどうかを把握できます。
整合性のある読み込みが要求されると、システムはまずWatchキャッシュの内容が最新かどうかを確認します。
キャッシュが最新でない場合、システムはキャッシュの内容が完全に更新されたと確認できるまで、etcdに進捗通知を問い合わせ続けます。
そして準備が整うと、要求されたデータはキャッシュから直接読み取られ効率的に提供されます。
このため、特にetcdから大量のデータを取得する必要があるような場面で、パフォーマンスを大幅に向上させることができます。
以上のようにして、データをフィルタリングするリクエストをキャッシュから処理できるようになり、etcdから読み取る必要のあるメタデータは最小限に抑えられます。


**重要な注意点:** この機能を利用するには、Kubernetesクラスタでetcdバージョン3.4.31以降または3.5.13以降を実行している必要があります。
古いバージョンのetcdを使用している場合、etcdから直接整合性のある読み込みを行う方式に自動で切り替わります。

### 体感できるパフォーマンスの向上

この一見単純な変更は、Kubernetesのパフォーマンスとスケーラビリティに大きな影響を与えます。

* **etcdの負荷軽減:** Kubernetes v1.31では、etcdの作業負荷を軽減し、他の重要な操作のためにリソースを解放できます。
* **レイテンシの短縮:** キャッシュからの読み込みは、etcdからデータを取得して処理するよりもはるかに高速です。
  これはコンポーネントへの応答が迅速になり、クラスター全体の応答性が向上することを意味します。
* **スケーラビリティの向上:** etcdの負荷軽減により、コントロールプレーンはパフォーマンスを犠牲にすることなくより多くのリクエストを処理できるようになるため、 数千ものノードとPodを持つような大規模なクラスターでは、最も大きなメリットが得られます。


**5,000ノードのスケーラビリティテスト結果:** 5,000ノードのクラスタで行われた最近のスケーラビリティテストでは、キャッシュからの整合性のある読み込みを有効にすることで、以下のような目覚ましい改善が見られました。

* kube-apiserverのCPU使用率が **30%削減**
* etcdのCPU使用率が **25%削減**
* PodのLISTリクエストの99パーセンタイルレイテンシが最大 **3分の1に短縮** (5秒から1.5秒)

### 今後の予定

ベータ版への移行により、キャッシュからの整合性のある読み込みはデフォルトで有効になり、サポートされているetcdバージョンを実行しているすべてのKubernetesユーザーにシームレスなパフォーマンス向上を提供します。

私たちの旅はこれで終わりではありません。
Kubernetesコミュニティは、将来的にさらにパフォーマンスを最適化するために、Watchキャッシュでのページネーションのサポートを積極的に検討しています。

### はじめ方
Kubernetes v1.31にアップグレードし、etcdバージョン3.4.31以降または3.5.13以降を使用していることを確認するのが、キャッシュからの整合性のある読み込みのメリットを体験する最も簡単な方法です。
ご質問やフィードバックがある場合は、Kubernetesコミュニティまでお気軽にお問い合わせください。

**キャッシュからの整合性のある読み込みによって、あなたのKubernetes体験がどう変わったか、ぜひ教えてください！**

この機能への貢献に対して、@ah8ad3 と @p0lyn0mial に特別な感謝を捧げます。

