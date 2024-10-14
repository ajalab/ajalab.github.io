---
title: "VLDB 2024 拾い読み"
date: 2024-10-14T20:30:00+09:00
draft: false
---

[VLDB](https://vldb.org/) はデータベースやデータ処理に関するカンファレンスです。休暇を利用して、今年出た論文 [PVLDB Volume 17](https://vldb.org/pvldb/volumes/17) をいくつか拾い読みしました。

## トレンド

以下のトピックが印象的だった。

- **LLM**
- **Disaggregated database**
- **Serverless database**

### LLM

LLM はあちらこちらで流行しているが、データベースも例外ではない。[D-Bot: Database Diagnosis System using Large Language Models](https://vldb.org/pvldb/volumes/17/paper/D-Bot%3A%20Database%20Diagnosis%20System%20using%20Large%20Language%20Models) は、データベース管理者（DBA）がやるトラブルシューティングを LLM エージェントに実行させる試みだ。[Text-to-SQL Empowered by Large Language Models: A Benchmark Evaluation](https://vldb.org/pvldb/volumes/17/paper/Text-to-SQL%20Empowered%20by%20Large%20Language%20Models%3A%20A%20Benchmark%20Evaluation) は、自然言語テキストから SQL クエリを生成するタスク（Text-to-SQL）において、LLM のプロンプトの与え方を比較・分析し、新しい手法 [DAIL-SQL](https://github.com/BeachWang/DAIL-SQL) を提案している。

### Disaggregated database

Disaggregated database とは、Compute、Memory、Storage といったリソースタイプによって層を分離し、各層ごとに役割に特化したノードで構成されたデータベースのことを指す。Compute+Memory 層と Storage 層の2層構造は既存のデータベース製品にも見られたが、最近は Compute 層と Memory 層も分離した3層構造が注目されているらしい。

- Compute: 計算性能に特化（例：100 CPU コアを持つ）
- Memory: メモリ容量に特化（例：100 GB のメモリ容量を持つ）
- Storage: ディスク性能に特化（例：xTB のディスクを持つ）

特に Compute 層と Memory 層の間の通信には、高速化のため [RDMA](https://ja.wikipedia.org/wiki/Remote_Direct_Memory_Access) を採用している例が多い。またメモリ空間とディスク空間には、いわゆる shared-nothing ではなく、空間を共有する shared-storage や shared-memory モデルが採用される。

このようなリソースタイプによる分離によって、disaggregated database は高い弾力性（必要なリソースを独立に拡張できる）および高いリソース利用効率を実現できるとされている。詳細は [Jianguo Wang](https://www.cs.purdue.edu/homes/csjgwang/) による[講義](https://www.cs.purdue.edu/homes/csjgwang/CS592DisaggregatedDB/)や [SIGMOD 2023 チュートリアルのスライド](https://www.cs.purdue.edu/homes/csjgwang/pubs/SIGMOD23_DisaggregatedDB_Slides.pdf)を参照のこと。

### Serverless database

サーバーレスといってもサーバーがなくなるわけではなく、トラフィックなどに応じてリソース容量をダイナミックに拡張もしくは縮小できることを言うらしい。[Amazon Aurora](https://aws.amazon.com/jp/rds/aurora/serverless/) や [Azure SQL](https://learn.microsoft.com/ja-jp/azure/azure-sql/database/serverless-tier-overview)、[TiDB](https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c) など、様々なクラウドデータベースサービスがサーバーレス対応を謳っている。

TiDB (TiDB Cloud Serverless) を例として、少しだけ具体的に見てみる。TiDB は大きくわけて TiDB、TiKV、PD の3つのコンポーネントで構成される。[Secrets Behind TiDB Serverless Architecture](https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c) や [How PingCAP transformed TiDB into a serverless DBaaS using Amazon S3 and Amazon EBS | AWS Storage Blog](https://aws.amazon.com/jp/blogs/storage/how-pingcap-transformed-tidb-into-a-serverless-dbaas-using-amazon-s3-and-amazon-ebs/) によれば、3つのコンポーネントのうち、Compute 層に属し SQL の処理を担当している TiDB が動的割り当ての対象になっている。すなわち、あるテナントにおいて TiDB へのトラフィックが低い場合、その TiDB インスタンスを他のテナントに割り当てることができる。これにより、Compute リソースのオーバーコミットを実現し、コストが削減できる。また永続性を S3（shared storage）によって担保することで、コンパクションや分析といった重い処理をユーザーのリクエスト処理から分離することも可能になる。

## 拾い読み

前置きが長くなったが、VLDB の論文の中で気になったものを挙げてみる。

### [D-Bot: Database Diagnosis System using Large Language Models](https://vldb.org/pvldb/volumes/17/paper/D-Bot%3A%20Database%20Diagnosis%20System%20using%20Large%20Language%20Models)

先に紹介した、LLM を用いてデータベースのトラブルシューティングを自動化する D-Bot の提案。[Hacker News でも少し話題になっていた](https://news.ycombinator.com/item?id=41150275)。

単純に LLM に投げて終わりではなく、かなり凝った作りをしているのが面白い。まず、LLM が参照するナレッジベースを、データベースのドキュメントを章ごとに分割した木をベースに作る。章（ノード）ごとにサマリーを LLM によって生成した後、各章ごとにトラブルシューティングに関する知識を LLM で抽出する。このとき、LLM は子ノードもしくは内容の近いサマリーを持つ他のノードも参照するように指示される。抽出された知識チャンクは数値化され、IO や CPU といったトピックごとにクラスタリングされる。

また、D-Bot がデータベースの状態を観測するためのツール（`vmstat` や `iostat` など）や、設定を変更するためのツールなどを LLM に認識させる必要がある。これらは機能やパラメータ等の説明とともに、カテゴリによって分類された階層構造によって整理する。

D-Bot は事前構築された問題分析やツールのナレッジベースを元に、各ステップの調査結果によって分岐する問題分析のための探索木を生成する。また、CPU やディスクなど、特定のトピックに特化したエキスパート LLM も併用する。

LLM の導入方法が参考になりそう。

### [GaussDB: A Cloud-Native Multi-Primary Database with Compute-Memory-Storage Disaggregation](https://www.vldb.org/pvldb/volumes/17/paper/GaussDB%3A%20A%20Cloud-Native%20Multi-Primary%20Database%20with%20Compute-Memory-Storage%20Disaggregation)

Huawei が開発した multi-primary な分散データベース [GaussDB](https://www.huaweicloud.com/intl/ja-jp/product/gaussdb.html) について。

先に紹介した Compute、Memory、Storage の3層 disaggregated database（shared memory, shared storage）アーキテクチャを採用している。データはページ単位で扱われる。

- Compute ノード：SQL 最適化、実行、およびトランザクション処理を担当する。あるページは単一の Compute ノードが所有し、ページを所有するノードだけがそのページにデータを書き込むことができる。トランザクション処理において、書き込み対象のページを所有していない場合は、Memory ノードに問い合わせてその所有権を得る。Compute ノードの障害時には Memory ノードまたは Storage ノードの状態を参照して復旧できる（two-tier recovery）。Compute ノードはこのためのチェックポイントも管理している。
- Memory ノード：ページの所有権やロックの管理、ページのキャッシュを担当する。Memory ノードそのものはステートレスであり、Compute ノードの状態をスキャンすることで復元できる。
- Storage ノード：ページやログの永続化を担当する。特に、Redo ログは Compute ノードごとに保存されるが、Undo ログは Compute レイヤー全体で共有されている。ノード共通のブロックベースなファイルシステム上で動作する。

このアーキテクチャによって、GaussDB は two-tier の高速なリカバリや高いスケーラビリティを実現している。

Storage レイヤーがどのように耐障害性を保証しているかが気になったが、詳細は読んだ限りではわからなかった。[Amazon EBS](https://aws.amazon.com/jp/ebs/) や [OpenStack Block Storage](https://docs.openstack.org/cinder/latest/) のようなものを使っているのかもしれない。3層構造の disaggregated database がすでに実用レベルに達しているというのが印象的だった。

### [Resource Management in Aurora Serverless](https://vldb.org/pvldb/volumes/17/paper/Resource%20Management%20in%20Aurora%20Serverless)

[Amazon Aurora Serverless](https://aws.amazon.com/jp/rds/aurora/serverless/) がどのようにして自動スケーリングを実現しているかを紹介している。著者の一人 [Marc Brooker](https://brooker.co.za/blog/) による[解説記事](https://brooker.co.za/blog/2024/07/29/aurora-serverless.html)がわかりやすい。

Aurora Serverless は、複数台のハイパーバイザ（ホスト）上で動く複数の DBMS エンジン（インスタンス）によって構成されている。自動スケーリングを実現する上で、どのインスタンスをどのホストに配置するか、また各インスタンスにどれくらいのリソースを割り当てるかがポイントになる。このリソースは ACU（Aurora Capacity Unit）という単位で表現されており、1 ACU は 2GB のメモリおよびある一定の CPU、ネットワーク、ブロックデバイス IO スループットに対応する。各インスタンスは ACU に関する2つの閾値を持つ。

- **ACU limit**：インスタンスが拡張可能な ACU の上限。通常時は、ユーザーが指定した ACU の最大容量と同じ値に設定されている。
- **reserved ACU**：インスタンスがただちに拡張できる ACU の上限。近い将来に必要だと想定されるリソース量よりも少しだけ大きな値に設定される。

スケーリングはホスト間もしくはホスト内で行われる。ライブマイグレーションやインスタンスへのリソースの動的割り当て、メモリ使用量の最適化といったテクニックが使われており、ACU limit および reserved ACU も同時に調整される。

- **ライブマイグレーション**：あるホストにおけるインスタンスの reserved ACU の合計が閾値を超えた際、そのホスト上で動作しているインスタンスの一部を別のホストにライブマイグレーションする。移行先ホストの選び方として、最も容量が余っているホストを選ぶのではなく、インスタンスが必要としているリソース容量とホスト残容量がフィットするようなホストを選ぶことで、他のホストに容量を残す戦略をとっている（ビンパッキング問題のヒューリスティック）。ライブマイグレーション中は ACU limit を一時的に reserved ACU と同じ値に設定することで、マイグレーションがトラフィックの増加に間に合わなくなるのを防いでいる。
- **インスタンスへのリソースの動的割り当て**：[AWS Nitro System](https://aws.amazon.com/jp/ec2/nitro/) がインスタンスへの動的な CPU およびメモリのプロビジョニングを可能にしている。Reserved ACU の範囲内でのスケーリング（in-place scaling）、または reserved ACU 自体の再計算（boundary management）が行われる。Reserved ACU にはトークンバケットを用いたレートリミットが設定されており、ライブマイグレーションへの影響が軽減されている。ACU の上限は、cgroups や CPU/memory on/offlining によって実際のリソースに適用される。
- **メモリ使用量の最適化**：Linux カーネルやデータベースエンジンは、空いているメモリを使い尽くすような挙動をする。これはデータベースが単一ノードで動く状況なら[正しい動作](https://www.linuxatemyram.com/)だが、自動スケーリングをするにあたっては好ましくない。そこで、memory offlining や cold page の検出等をゲストカーネル上で行うことで、インスタンスにメモリを積極的に解放させるようにしている。

仮想化技術をふんだんに利用していて面白い。一方で、リソース管理には複雑な予測アルゴリズムを用いず、主要なところではシンプルでリアクティブな実装で済ましているのが素晴らしいバランス感覚だと思う。また、急激なトラフィックの増加を可能な限りサポートするのではなく、レートリミットを適用して予測可能性を高めるという実装方針も示唆的だった。

### [Cloud-Native Database Systems and Unikernels: Reimagining OS Abstractions for Modern Hardware](https://www.vldb.org/pvldb/volumes/17/paper/Cloud-Native%20Database%20Systems%20and%20Unikernels%3A%20Reimagining%20OS%20Abstractions%20for%20Modern%20Hardware)

DBMS をプロセスや権限の分離をしない [unikernel](http://unikernel.org/) 上で動かすことで、権限昇格のコストを回避したり、ハードウェアやページテーブルを直接管理できるようにするというアイディアの紹介。

例として、データベース内で OLTP と OLAP ワークロードを分離する際に、メモリの Copy-on-Write スナップショットをとるケースが紹介されている。[`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html) の代わりに、unikernel から専用の機能を提供できるよねということらしい。Unikernel が使えそうなことには異論はないが、これからより面白い使い方や実例が現れることを期待している。

### [X-Stor: A Cloud-native NoSQL Database Service with Multi-model Support](https://vldb.org/pvldb/volumes/17/paper/X-Stor%3A%20A%20Cloud-native%20NoSQL%20Database%20Service%20with%20Multi-model%20Support)

キーバリュー、時系列、グラフ、ドキュメントといった種々の NoSQL データベースのモデルを統一して扱えるようにするデータベース X-Stor の紹介。

Data Plane と Control Plane に分かれたアーキテクチャ。

- Data Plane: Access 層、Cache 層、Storage 層からなる。各 plane はサポートするモデルごとに異なる Storage ノードを Storage 層に持つ。
- Control Plane: メタデータを管理する。またテーブル作成等の管理操作を workflow として統一的に扱う。

Aurora Serverless と同様に、X-Stor もリソース管理のために Request Unit (RU) という統一された指標を元にリソース分離やロードバランシングを行う。

NoSQL を対象として、インターフェースの簡素化やマイグレーションのために抽象化したレイヤを導入するパターンをたまにみかける。これは管理者向けの抽象化だけど、Netflix が NoSQL データベースクライアント向けに抽象化レイヤを導入した事例（[キーバリュー](https://netflixtechblog.medium.com/introducing-netflixs-key-value-data-abstraction-layer-1ea8a0a11b30)、[時系列](https://netflixtechblog.com/introducing-netflix-timeseries-data-abstraction-layer-31552f6326f8)）が記憶に新しい。

### [Caerus: Low-Latency Distributed Transactions for Geo-Replicated Systems](https://vldb.org/pvldb/volumes/17/paper/Caerus%3A%20Low-Latency%20Distributed%20Transactions%20for%20Geo-Replicated%20Systems)

地理分散された DBMS 上の分散トランザクションを、1回のラウンドトリップで効率良く順序づけするプロトコルの提案、およびデータベース実装 Caerus の紹介。

[Calvin](https://github.com/yaledb/calvin)（[Database Internals](https://www.databass.dev/) でも紹介されていた）は、トランザクションの順序づけに Paxos を使っている。一方で、Caerus は各リージョンごとにトランザクションを部分的に順序づけしたあとに、それらをマージするという方式らしい。

### [Timestamp as a Service, not an Oracle](https://www.vldb.org/pvldb/volumes/17/paper/Timestamp%20as%20a%20Service%2C%20not%20an%20Oracle)

分散トランザクションに必要な論理タイムスタンプを SPoF なしで提供する TaaS（timestamp as a service）の提案。

原子時計や TrueTime が使えない TiDB のような DBMS は、中心化された[タイムスタンプオラクル](https://docs.pingcap.com/ja/tidb/stable/tso)を用いることが多い。しかし、リーダーがクラッシュするとその間にタイムスタンプが発行できず可用性が下がってしまう問題がある。 そこで、論文では合意アルゴリズムを用いずに論理タイムスタンプを独立に生成するサービスを提案している。

比較のベースラインが TiDB になっている。ただ TiDB もサーバーレス化のために [PD から TSO サービスを分離](https://dataturbo.medium.com/secrets-behind-tidb-serverless-architecture-8b277f00cb7c)しているので、似たようなことはやっていそう。

### [An Empirical Evaluation of Columnar Storage Formats](https://www.vldb.org/pvldb/volumes/17/paper/An%20Empirical%20Evaluation%20of%20Columnar%20Storage%20Formats)

列志向のデータフォーマットである [Apache Parquet](https://parquet.apache.org/) や [Apache ORC](https://orc.apache.org/docs/) の比較。

ベンチマークのやり方が非常にしっかりしていて良い。パラメータやワークロードの選定の仕方が、自分でベンチマークを組む上で参考になるかもしれない。

特に示唆的だったのは以下の点。

- 実世界のデータは、データタイプにかかわらず（含 float 型）ユニークなデータの割合が低い。なので基本的には dictionary encoding を採用するべき。ベンチマークで Parquet のほうが圧縮効率が良いのも、Parquet が dictionary encoding をより積極的に使っているからかもしれない。
- Snappy や gzip といったブロック圧縮は原則不要。現代的なハードウェアにおけるボトルネックは CPU に移っているため。

## 感想

オープンソースの DBMS をオンプレで運用している身としては、クラウドデータベース大手の resource disaggregation やサーバーレス化によるコスト削減に張り合っていくのはちょっとしんどいなと感じる。ただデータベースの仕事には上（分散システムにおける並行性、インターフェイス）から下（OS、ハードウェア）まで面白い課題がたくさんあり、やっている分には非常に楽しい。LLM に仕事が奪われないことを祈るばかりである。
