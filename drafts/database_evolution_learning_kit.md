# 0) 進化の年表（1ページ）

- **1970 relational model**: データを「関係（relation）」として扱い、宣言的に問い合わせる枠組みが提示された。これにより、物理格納の都合と論理モデルを分離できる土台ができた。([Codd 1970](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf))
- **1970s–80s SQL / optimization / recovery (System R era)**: SQL 実装・コストベース最適化・アクセスパス選択の実務が確立し、同時に障害回復を WAL/ログ中心に体系化する流れが進んだ。([System R 1981](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p632-chamberlin.pdf), [ARIES 1992](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf))
- **1990s–2000s web-scale + distributed reality**: 単一ノード前提では遅延・部分故障・ネットワーク分断を避けられず、分散前提の設計語彙（複製、クォーラム、非同期修復）が必須化した。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), [Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf))
- **2000s Bigtable/Dynamo (NoSQL patterns)**: wide-column と key-value の実用パターンが確立し、「高可用・水平分割・運用継続」を重視する設計が拡大した。([Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf), [Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf))
- **2010s Spanner/NewSQL (strong consistency at global scale)**: グローバル分散でも外部整合（実運用上の強い整合性）を提供する方向が現実化し、時刻 API と分散トランザクション実装の統合が進んだ。([Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf))
- **2010s–2020s cloud DWH (separation, elasticity), columnar/vectorization, serverless**: ストレージとコンピュートの分離、クラウド弾力性、列指向＋ベクトル化での高スループット分析が主流化した。([Snowflake 2016](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf), [ClickHouse PVLDB](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf), [Dremel 2010](https://research.google/pubs/pub36632/))
- **recent: cloud-native high performance & HTAP patterns**: 高速 OLAP と低遅延更新の両立（HTAP）は「単一エンジン統合」「役割分離」「ログ/レプリカ offload」など複数流派に分岐している。([SAP HANA](https://sites.computer.org/debull/A12mar/hana.pdf), [DuckDB CIDR 2019](https://www.cidrdb.org/cidr2019/papers/p29-raasveldt-cidr19.pdf), [ClickHouse PVLDB](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf))

---

# 1) A: 基礎思想（単一ノードRDBの地面）

## A1 relational model (keys, constraints)
- relation は「属性の集合を持つタプル集合」として定義し、論理設計を物理実装から分離する。([Codd 1970](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf))
- **主キー**は行同定、**外部キー**は参照整合性、**制約**は「入れてはいけない状態」を宣言で表現する。

## A2 normalization (what it prevents; 3NF/BCNF intuition)
- 正規化の本質は、更新・挿入・削除の異常を関数従属性にもとづいて減らすこと。
- 直感:
  - **3NF**: 非キー属性の不要な従属連鎖を避ける。
  - **BCNF**: 決定項が候補キーであることをより厳密に要求する。
- 代償: JOIN コストが増える可能性。

## A3 indexes (B+Tree, access paths)
- B+Tree は範囲検索に強く、葉ノード連結により順次走査効率を確保。
- インデックスは「I/O を減らすための追加構造」であり、更新時には保守コストが発生する。([System R 1981](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p632-chamberlin.pdf))

## A4 query optimization intuition (stats, join order, cost model)
- 最適化器は統計情報を使い、結合順序・アクセスパスを探索し、推定コスト最小計画を選ぶ。([System R 1981](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p632-chamberlin.pdf))
- 誤推定があると実行計画が悪化するため、統計更新は運用上重要。

## A5 transactions (ACID, isolation anomalies; examples)
- ACID は「壊れない更新」の約束。
- 分離レベルが低いと dirty read / non-repeatable read / phantom などが起き得る。
- MVCC 系は読取りと書込みの干渉を減らすが、バージョン管理コストを払う。

## A6 recovery (WAL, redo/undo mental model; ARIES as anchor)
- WAL: データページより先にログ永続化。
- ARIES の直感:
  1. **Analysis**: どこから再開するか把握
  2. **Redo**: 反映漏れ更新を再適用
  3. **Undo**: 未コミット更新を打消し
- 「steal/no-force」を許す実装現実に整合する回復手順が ARIES の核。([ARIES 1992](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf))

## A7 A→Bブリッジ (partitioning, replication basics, quorum vocabulary)
- 単一ノード限界を超えるには、
  - **partitioning**: データを分割
  - **replication**: コピーで可用性を向上
  - **quorum (R/W/N)**: 何台の応答で読書きを成立させるか
- ここから分散の失敗モデル（遅延・分断・部分故障）を前提にする必要が出る。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf))

### 用語集（Stage A）
- **Functional Dependency**: X が決まると Y が一意に決まる関係。
- **Candidate Key**: 行を一意に識別できる最小属性集合。
- **BCNF**: 全 FD の決定項がスーパーキー。
- **Access Path**: どの索引/走査でデータに到達するか。
- **Cost Model**: I/O, CPU, ネットワーク等の推定コスト。
- **WAL**: 先にログ、後でデータ。
- **Redo/Undo**: 再適用 / 打消し。
- **Isolation Level**: 同時実行時に許容する見え方。

## == Explanation Test T1 ==
**3分スクリプト骨子**「RDBとは何か。なぜ正規化/索引/トランザクション/リカバリが必要か」
1. RDB は「表」ではなく、関係代数に基づく論理モデル（30秒）
2. 正規化が異常更新を防ぐ理由（40秒）
3. 索引が I/O を減らすが更新コストを増やす（40秒）
4. トランザクションが同時更新の破壊を防ぐ（40秒）
5. WAL+ARIES がクラッシュ後の整合復元を保証（30秒）

---

# 2) B: 分散・NoSQL（壊れ方と分岐）

## B1 what breaks in distributed (latency, partial failure, partitions)
- 分散では「落ちる」だけでなく「遅い」「届かない」「一部だけ見える」が起きる。
- そのため、ローカル ACID の感覚をそのまま適用できない。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), [Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf))

## B2 consistency vocabulary (strong/eventual, linearizable, quorum)
- **Strong consistency**: 最新の確定書込みを直ちに観測できるモデル（文脈依存で定義差あり）。
- **Linearizable**: 実時間順序と整合する単一オブジェクト操作の見え方。
- **Eventual consistency**: 更新停止後に最終的に収束。
- **Quorum**: R+W>N で古い値確率を下げるが、衝突解決・遅延悪化の運用論点が残る。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf))

## B3 replication patterns
- leader-follower, multi-leader, leaderless(Dynamo 型)で設計点が異なる。
- 非同期複製は遅延と分岐解決戦略（LWW, version vector 等）が必要。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf))

## B4 why distributed transactions are hard (intuition-first)
- 参加ノード増加で失敗点が増える。
- 原子性・整合性・可用性・遅延を同時最大化できないため、プロトコル選択は要件依存。
- Spanner は時刻 API と 2PC/レプリケーション協調で外部整合性を狙う。([Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf))

## B5 storage branch: LSM vs B+Tree (write/read/space amplification)
- **LSM**: 書込みは順次追加中心で高スループット。ただし compaction による read/space/write amplification が発生。
- **B+Tree**: 点読取り・範囲走査の安定性が高い一方、ランダム更新 I/O が重くなる場面がある。

## B6 NoSQL patterns: KV, wide-column, document (strengths/weaknesses)
- **KV**: 単純・高速・水平分割しやすいが複雑問合せは弱い。
- **wide-column**: スパース大規模データに適し、時系列/ログ系に強い。([Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf))
- **document**: 柔軟スキーマで開発速度が高いが、複雑整合制約の管理が難化しやすい。

## B7 ops view: hotspots, tail latency, compaction
- 熱いキー偏り、遅い末尾遅延、バックグラウンド compaction 干渉が主要運用課題。
- 運用では p99 遅延、再バランス、修復帯域を監視軸に置く。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), [Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf))

### Mini Thought Labs
- **Lab B1 (amplification)**: 書込み比率 90% ワークロードで、LSM の compaction 帯域が飽和したら何が先に壊れるか（p99 書込み遅延、ディスク使用量、read miss）を順に推論。
- **Lab B2 (Dynamo choices)**: N=3 で W=1/R=1 を選ぶ利点と、2 ノード障害時の読み結果の不確実性を説明。hinted handoff で回復後何を確認するか。
- **Lab B3 (consistency scenario)**: 同一キーへ連続更新 U1,U2 の直後に地域別読取りが食い違う理由を、複製遅延とクォーラム不足で説明。

## == Explanation Test T2 ==
**3分スクリプト骨子**「なぜ NoSQL が生まれたか、可用性/整合性/性能のトレードオフ」
1. Web-scale で単一ノード仮定が崩れた（40秒）
2. 分断時に“何を守るか”を決める必要（40秒）
3. Dynamo/Bigtable の設計選択（60秒）
4. LSM・クォーラム・運用負債の要点（40秒）

---

# 3) C: 現代設計思想（クラウド前提の設計判断の地図）

## C1 scaling strategy (scale-up/scale-out; shared-nothing/shared-storage)
- scale-up は単機性能、scale-out は台数で伸ばす。
- shared-nothing は独立性高いが再分散が課題、shared-storage は運用柔軟だがメタデータ/ネットワークが律速になりやすい。([Snowflake 2016](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf))

## C2 storage/compute separation + elasticity + cost model
- 分離により「保存容量」と「計算ピーク」を独立課金・独立伸縮しやすい。([Snowflake 2016](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf))

## C3 execution engine ideas (row/column, vectorization, pushdown)
- 分析は列指向＋ベクトル化で CPU 効率が高い。
- pushdown はデータ移動前に絞り込み・集約してネットワークコストを減らす。([Dremel 2010](https://research.google/pubs/pub36632/), [ClickHouse PVLDB](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf), [DuckDB CIDR 2019](https://www.cidrdb.org/cidr2019/papers/p29-raasveldt-cidr19.pdf))

## C4 HTAP philosophies (unified vs dual-store vs offload; freshness vs interference)
- unified: 1エンジンで鮮度高いが干渉制御が難しい。
- dual-store: 役割分離で安定だがデータ移送遅延。
- offload: 本系負荷を逃がすが可観測性/整合窓の設計が必要。([SAP HANA](https://sites.computer.org/debull/A12mar/hana.pdf))

## C5 strong-consistent distributed DB principles (Spanner lineage)
- グローバル整合性は、複製合意・分散トランザクション・時刻整合の同時設計で成立。([Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf))

## C6 serverless / multi-tenant isolation / observability
- serverless は自動スケールを得る代わりに、起動遅延・近隣ノイズ抑制・課金可視化が重要。
- multi-tenant は資源隔離（CPU/メモリ/IO）と公正スケジューリングが中核。

## C7 hardware co-design (offload, appliance, disaggregation; network bottlenecks)
- 現代ボトルネックは CPU 単体でなく、ネットワーク/ストレージ階層/メタデータ制御面へ移る。
- disaggregation は弾力性を上げるが、遠隔アクセス遅延を設計で吸収する必要。

## C8 bottleneck dictionary (log, network, metadata, compaction, cache)
- **log**: fsync 待ち/ログ帯域。
- **network**: shuffle/replication 帯域。
- **metadata**: カタログ/ロック集中。
- **compaction**: バックグラウンド I/O 競合。
- **cache**: 局所性崩壊によるヒット率低下。

### 思想マップ（ASCII）

```text
要求
├─ OLTP低遅延/高整合 ──> NewSQL(Spanner系) ──> 合意/時刻/2PC設計
├─ OLAP高スループット ─> Columnar+Vectorized(Dremel/ClickHouse/DuckDB)
├─ 可用性最優先 ─────> Dynamo系(KV/クォーラム/収束)
└─ 弾力運用最優先 ───> Cloud DWH(Snowflake系の分離設計)
          │
          └─ HTAP分岐: Unified / Dual-store / Offload
```

### 比較表

| アプローチ | workload | data model | storage engine | transactions/consistency | scaling model | main performance levers | ops/cost levers |
|---|---|---|---|---|---|---|---|
| 伝統RDB | 混在OLTP | relational | B+Tree中心 | ACID/強整合 | scale-up主体+部分scale-out | index, optimizer, buffer | HA設計, ログ運用 |
| Dynamo系KV | 高可用KV | key-value | LSM/append系 | eventual〜調整可能 | shared-nothing | quorum調整, hinted handoff | repair/compaction |
| Bigtable系 | 時系列・ログ・分析前段 | wide-column | SSTable/LSM | 行/範囲単位の制御 | タブレット分割 | locality, scan設計 | hotspot分散 |
| Spanner系 | グローバルOLTP | relational+KV内部 | 複製ログ+分散ストレージ | external consistency | geo-distributed scale-out | data placement, txn shaping | レイテンシ予算/地域配置 |
| Cloud DWH | 大規模OLAP | 列指向 | object storage + cache | 主に分析整合 | 分離＋elastic | vectorization, pruning | 自動停止/スケール課金 |

## == Explanation Test T3 ==
**3分スクリプト骨子**「現代の思想マップを描き、主要アプローチを分類する」
1. 要件軸（整合性/可用性/コスト/運用）を先に置く（40秒）
2. 4系統（RDB, Dynamo, Spanner, Cloud DWH）へ分類（80秒）
3. HTAP 3流派と干渉対策を説明（60秒）

---

# 4) D: 製品（思想→具体例として読む）

> 使い方: 暗記ではなく **C1–C8 で読解**する。

## Card 1: Oracle Database
- **Primary**: [Oracle Database Concepts 23ai](https://docs.oracle.com/en/database/oracle/oracle-database/23/cncpt/introduction-to-oracle-database.html)
- **Philosophy mapping (C1–C8)**: C1☑ C2△ C3☑ C4△ C5☑ C6△ C7△ C8☑
- **for / not for**
  - for: 厳密整合OLTP、複雑SQL、成熟運用
  - not for: 低運用負荷の完全サーバレス分析特化
- **performance levers**: 実行計画/索引設計/統計、メモリ領域、ログI/O、パーティショニング、並列実行
- **ops & failure modes**: ログ詰まり、統計劣化、ロック競合、I/O飽和、バックアップ窓
- **exercise**
  1. 更新遅延が急増したら最初に待機イベントをどう切るか？
  2. 統計が古い疑い時、計画安定化をどう行うか？

## Card 2: Microsoft SQL Server (Hekaton)
- **Primary**: [Hekaton SIGMOD 2013](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/hekaton-sigmod2013-final.pdf)
- **Mapping**: C1☑ C2△ C3☑ C4☑ C5☑ C6△ C7☑ C8☑
- **for / not for**: 高TPS OLTP / 超大規模分散リージョン一貫性単体では限定
- **levers**: インメモリ表、ネイティブコンパイル、ロック回避設計、ログ最適化、ワークロード分離
- **ops/failure**: メモリ圧迫、チェックポイント遅延、ログボトルネック、温度差、GC系負荷
- **exercise**: p99悪化時にメモリ/ログどちら先に見る？ スキーマ変更影響をどう見積もる？

## Card 3: SAP HANA
- **Primary**: [SAP HANA architecture](https://sites.computer.org/debull/A12mar/hana.pdf)
- **Mapping**: C1☑ C2△ C3☑ C4☑ C5☑ C6△ C7☑ C8☑
- **for / not for**: 低遅延分析＋トランザクション近接 / 安価なコールド長期保存のみ
- **levers**: 列圧縮、in-memory、並列実行、辞書符号化、デルタマージ
- **ops/failure**: メモリ枯渇、マージ干渉、長大トランザクション、NUMA偏り、バックアップ時間
- **exercise**: デルタ肥大時の対処は？ 圧縮率低下時にどこを見る？

## Card 4: Google Spanner
- **Primary**: [Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf)
- **Mapping**: C1☑ C2☑ C3△ C4☑ C5☑ C6☑ C7☑ C8☑
- **for / not for**: グローバル整合OLTP / 超低コスト単純ローカルDB
- **levers**: インターリーブ配置、地域選択、トランザクション形状、セッション再利用、二次索引
- **ops/failure**: 地域間遅延、ホットキー、スキーマ変更待機、バックフィル長時間、クォータ制約
- **exercise**: 地域追加で遅延増なら何を再配置？ 書込み競合時にキー設計をどう修正？

## Card 5: FoundationDB
- **Primary**: [FoundationDB SIGMOD 2021](https://www.foundationdb.org/files/fdb-paper.pdf)
- **Mapping**: C1☑ C2☑ C3△ C4☑ C5☑ C6☑ C7☑ C8☑
- **for / not for**: 厳密トランザクションKV基盤 / 生SQLそのまま実行
- **levers**: レイヤード設計、トランザクションサイズ管理、キー空間設計、バッチ化、ロール分離
- **ops/failure**: トランザクション競合、ホットレンジ、再配置コスト、遅延スパイク、監視不足
- **exercise**: conflict rate 上昇時の分解手順は？ range hotspot をどう分散？

## Card 6: Snowflake
- **Primary**: [Snowflake 2016](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf)
- **Mapping**: C1☑ C2☑ C3☑ C4△ C5△ C6☑ C7☑ C8☑
- **for / not for**: 弾力OLAP/多チーム同時分析 / 超低遅延OLTP
- **levers**: 仮想ウェアハウス分離、マイクロパーティション、結果キャッシュ、クラスタリング、同時実行スケール
- **ops/failure**: クレジット過消費、小クエリ過多、スキュー、メタデータ増大、権限複雑化
- **exercise**: コスト急増時に停止/サイズ/並行度をどう調整？ キャッシュ不発の原因は？

## Card 7: BigQuery / Dremel lineage
- **Primary**: [Dremel 2010](https://research.google/pubs/pub36632/)
- **Mapping**: C1☑ C2☑ C3☑ C4△ C5△ C6☑ C7☑ C8☑
- **for / not for**: 大規模対話分析 / 高頻度更新OLTP
- **levers**: 列投影、ネスト表現、木構造集約、スキャン削減、シャッフル最適化
- **ops/failure**: スキャン肥大、スキーマ乱立、ジョブ待ち行列、権限管理、コスト予測難
- **exercise**: 同じSQLでコスト2倍の原因は？ パーティション戦略をどう見直す？

## Card 8: ClickHouse
- **Primary**: [ClickHouse PVLDB](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf)
- **Mapping**: C1☑ C2△ C3☑ C4△ C5△ C6☑ C7☑ C8☑
- **for / not for**: 高速集計・ログ分析 / 厳密多行トランザクション
- **levers**: MergeTree設計、主キー順序、圧縮コーデック、マテビュー、データスキッピング
- **ops/failure**: compaction圧、パーツ過多、レプリカ遅延、メモリ超過、クエリ暴走
- **exercise**: parts 爆増時に merge policy をどう調整？ 遅いクエリの読み取り量をどう削る？

## Card 9: DuckDB
- **Primary**: [DuckDB CIDR 2019](https://www.cidrdb.org/cidr2019/papers/p29-raasveldt-cidr19.pdf)
- **Mapping**: C1△ C2△ C3☑ C4△ C5△ C6△ C7☑ C8☑
- **for / not for**: 組込みOLAP・ローカル分析 / 大規模分散OLTP
- **levers**: ベクトル化、列指向、遅延実行、ファイル直接読取、SIMD活用
- **ops/failure**: 単機メモリ限界、同時実行制約、巨大JOIN一時領域、I/O帯域、依存環境差
- **exercise**: ノートPCでOOM時の対処は？ Parquet直読と取り込みの境界は？

## Card 10: Apache Cassandra
- **Primary**: [Cassandra LADIS 2009](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf)
- **Mapping**: C1☑ C2☑ C3△ C4△ C5△ C6☑ C7△ C8☑
- **for / not for**: 高可用書込み・地理分散KV/ワイド列 / 複雑JOIN
- **levers**: パーティションキー設計、CL調整、圧縮、TTL、コンパクション戦略
- **ops/failure**: tombstone肥大、repair負荷、ホットパーティション、read修復遅延、GC停止
- **exercise**: 読み遅延悪化時に tombstone/repair をどう切り分け？ CL変更の副作用は？

## Card 11: Apache HBase
- **Primary**: [Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf), [Apache HBase Reference](https://hbase.apache.org/book.html)
- **Mapping**: C1☑ C2△ C3△ C4△ C5△ C6☑ C7△ C8☑
- **for / not for**: 大規模時系列・スパース表 / ad-hoc複雑SQL
- **levers**: rowkey設計、リージョン分割、Bloom filter、BlockCache、compaction制御
- **ops/failure**: hotspot rowkey、リージョン偏り、compaction競合、GC/heap問題、WAL詰まり
- **exercise**: 単調rowkeyで集中したらどう再設計？ compaction backlog の緩和策は？

## Card 12: Alibaba PolarDB
- **Primary**: [PolarDB PVLDB 2021](https://www.vldb.org/pvldb/vol14/p431-cao.pdf)
- **Mapping**: C1☑ C2☑ C3☑ C4△ C5☑ C6☑ C7☑ C8☑
- **for / not for**: クラウドRDB互換＋弾力拡張 / 完全サーバレス分析専用
- **levers**: 共有ストレージ、読みスケール、ログ最適化、キャッシュ階層、並列化
- **ops/failure**: ストレージ遅延波、メタデータ負荷、読書き分離の整合窓、昇格切替、コスト管理
- **exercise**: 読みレプリカ増で逆に遅い時どこを見る？ 書込みピーク時のログ経路最適化は？

## Card 13: OceanBase
- **Primary**: [OceanBase PVLDB 2022](https://www.vldb.org/pvldb/vol15/p3389-xu.pdf)
- **Mapping**: C1☑ C2☑ C3☑ C4☑ C5☑ C6☑ C7☑ C8☑
- **for / not for**: 分散SQLで高整合OLTP/HTAP志向 / 単純ローカル軽量用途のみ
- **levers**: 分区設計、ローカリティ、並列実行、ログ複製、リソース隔離
- **ops/failure**: 交差分区トランザクション、再均衡コスト、地域間遅延、メタデータ圧、バックアップ窓
- **exercise**: cross-partition 比率増加時の設計見直しは？ 地域障害時の優先復旧順序は？

## == Explanation Test T4 ==
**3分スクリプト骨子**「1製品を選び mapping → fit → pitfalls を説明」
1. C1–C8 で設計思想を30秒で地図化
2. 典型 workload と非適合 workload を対比（60秒）
3. 性能レバー3点 + 運用故障モード2点（60秒）
4. “障害時に最初に何を見るか”で締める（30秒）

---

# 5) 1枚で覚える「説明テンプレ」（必須）

## Template 1: なぜ生まれたか
- **constraints**: 既存方式で何が詰まったか（例: 単機限界、遅延、分断）
- **new requirements**: 何を守る必要が出たか（可用性/整合性/コスト）
- **bottleneck**: 実ボトルネックはどこか（ログ、ネットワーク、メタデータ）
- **approach**: どの設計に振ったか（分離、クォーラム、列指向など）

## Template 2: どう成立しているか
- **data layout**: 行/列、キー配置、パーティション
- **consistency**: トランザクション境界、複製モデル、隔離レベル
- **levers**: 性能を動かすつまみ（索引、圧縮、並列度、キャッシュ）
- **costs**: 代償（運用複雑性、増幅、コスト予測性、干渉）

## Template 3: いつ選ぶか
- **workload → approach**: OLTP/OLAP/時系列/グローバル更新を先に分類
- **constraints → exceptions**: コンプライアンス、リージョン要件、予算、組織運用力で例外処理
- **final choice**: 「捨てる要件」を明示して選択

---

# 6) 最終レビュー

## If you only remember 20 things
1. RDB の本質は表ではなく relational model。([Codd 1970](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf))
2. 論理と物理を分離すると進化しやすい。
3. 正規化は更新異常を減らす設計規律。
4. 索引は read を速くし write を重くする。
5. 最適化器は統計が命。([System R 1981](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/p632-chamberlin.pdf))
6. ACID は「同時更新でも壊さない」契約。
7. WAL は durability の実務中核。([ARIES 1992](https://web.stanford.edu/class/cs345d-01/rl/aries.pdf))
8. ARIES は analysis/redo/undo で復旧を分解。
9. 分散では部分故障が常態。
10. 可用性・整合性・遅延は同時最大化できない。
11. Dynamo は可用性重視の代表。([Dynamo 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf))
12. Bigtable は wide-column 実用化の代表。([Bigtable 2006](https://research.google.com/archive/bigtable-osdi06.pdf))
13. LSM は書込み強いが compaction 負債を持つ。
14. Spanner は強整合グローバル分散の代表。([Spanner 2012](https://research.google.com/archive/spanner-osdi2012.pdf))
15. Cloud DWH は storage/compute 分離が中核。([Snowflake 2016](https://www.cs.cmu.edu/~15721-f24/papers/Snowflake.pdf))
16. 列指向＋ベクトル化は分析で効く。([Dremel 2010](https://research.google/pubs/pub36632/), [ClickHouse PVLDB](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf))
17. HTAP は万能解でなく設計流派。
18. serverless は運用削減と観測難の交換。
19. 実運用は p99 と再配置とメタデータが勝負。
20. 製品名でなく「思想地図(C1–C8)」で選ぶ。

## 12-question mixed quiz

### A（基礎）
1. BCNF が 3NF より厳しい点を 1 文で説明せよ。
2. WAL が「データ先書き」でない理由は何か。
3. 最適化器の結合順序誤りが起きる主因を挙げよ。

### B（分散/NoSQL）
4. N=3, R=2, W=2 の直感的意味を説明せよ。
5. eventual consistency で「不整合が永続しない」条件は何か。
6. LSM の compaction が p99 を悪化させる経路を述べよ。

### C（現代設計）
7. storage/compute 分離がコスト最適化に効く理由は？
8. 列指向＋ベクトル化で CPU 効率が上がる理由は？
9. HTAP の unified と dual-store の最大トレードオフは？

### D（製品適用）
10. Spanner系を選ぶべき条件を 2 つ挙げよ。
11. ClickHouse が不向きなトランザクション要件を 1 つ挙げよ。
12. Cassandra で hotspot が出たときのキー設計見直し方針を述べよ。

