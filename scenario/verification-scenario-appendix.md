# 検証シナリオ 付録：手順と根拠の詳細

[本文](verification-scenario.md) の詳細です。実施する人が手元で参照するためのもので、判断する人は本文だけを読めば足ります。

| 付録 | 内容 |
|------|------|
| A | ゲートと構築の確かめ方 |
| B | 遅延の測り方と判定の詳細 |
| C | 記録とログの設定 |
| D | 追加テスト（必要なときだけ） |
| E | 公式ドキュメントで確かめたこと（出典つき） |
| F | 前提と範囲 |

---

## A. ゲートと構築の確かめ方

### A.1 作るもの

- 提供側：デュアルスタックの VPC、ECS サービス（Fargate、タスク 2 つ、AZ ごとに 1 つ）、NLB、エンドポイントサービス、VPC Lattice のサービスとターゲットグループ
- 呼び出し側：デュアルスタックの VPC、クライアント（AZ ごとに 1 台）、インターフェイスエンドポイント、サービスネットワークへの VPC の関連付け
- 付録 C.2 のログ

IaC で作り、同じ構成を作り直せるようにします。

**確認用アプリのポート**
ECS タスクのアプリは 2 つのポートで待ち受けます。Proxy Protocol v2 を有効にしたターゲットグループに、Proxy Protocol を解釈しないポートをつなぐと、すべてのリクエストが失敗するためです。

| ポート | 受けるもの | つなぐ先 | 使うテスト |
|--------|------------|----------|------------|
| 8080 | 平文の HTTP | 基準（直接）、NLB のターゲットグループ A、Lattice | L1〜L3、S1、S2、S4、S5 |
| 8081 | Proxy Protocol v2 ＋ HTTP | NLB のターゲットグループ B（Proxy Protocol v2 あり） | S3〜S5 の PrivateLink 側 |

アプリは、受け取ったリクエストの送信元 IP、すべてのヘッダー、Proxy Protocol v2 の中身、受信時刻、処理時間をログに出します。

### A.2 ゲートの確かめ方

| ゲート | 確かめ方 |
|--------|----------|
| G1 | 呼び出し側のクライアントで `dig AAAA <Lattice のサービス名>` を実行し、AAAA が返ることを見る。続けて `curl -6 -v` で HTTP 200 が返り、アプリのログで送信元が IPv6 であることを見る |
| G2 | G1 と同じことを、PrivateLink のエンドポイントの DNS 名（AZ ごとの名前を含む）で行う。返らなければ、エンドポイントの IP アドレスタイプと DNS レコードの IP タイプがデュアルスタックかを見る（E の Q1） |
| G3 | ECS サービスに、NLB 用の `loadBalancers` と Lattice 用の `vpcLatticeConfigurations` を両方設定して作成できるかを試す。できなければ、ECS タスクの IPv6 アドレスを Lattice のターゲットグループに IP ターゲットとして登録する。この場合、タスクが入れ替わると登録が古くなるので、測定中はタスクを入れ替えない |
| G4 | G1・G2 で送ったリクエストが、付録 C.2 の 4 種類のログすべてに出ていることを見る。VPC フローログに IPv6 のアドレスが入っていることも見る |

### A.3 そのほかの確かめ項目

ゲートに加えて、次も確かめます。結果は `results/build-check.md` に残します。

- [ ] 両方の VPC とサブネットに IPv6 の CIDR と、IPv6 のルートがある（`describe-vpcs`、`describe-subnets`、`describe-route-tables`）
- [ ] ECS タスクが IPv6 アドレスを持ち、ターゲットグループに IPv6 で登録されている（`describe-tasks`、`describe-target-health`）
- [ ] すべてのターゲットが healthy（`describe-target-health`、`list-targets`）
- [ ] ターゲットがクライアントと同じ AZ ID だけに登録され、NLB のクロスゾーン負荷分散が無効（B.3）
- [ ] ポート 8080 と 8081 が、それぞれ正しく応答する
- [ ] ECS のオートスケーリングが無効で、タスク数が 2 のまま（`describe-services`）
- [ ] Lattice の認証タイプが、認証なしでは `NONE`、ありでは `AWS_IAM` で、認証ポリシーが意図どおり（`get-service`、`get-auth-policy`）
- [ ] クライアントと ECS タスクの時刻が合っている（Amazon Time Sync Service）
- [ ] 構成のスナップショットを取った（C.3）

## B. 遅延の測り方と判定の詳細

### B.1 測り方

- **ツール**：一定のリクエスト率で送り続けられ、パーセンタイルを出せるもの（例：`wrk2`、`fortio`、`vegeta`）を使います。一定の率で送れないツールは、遅れた分が測定値に表れない（協調的省略）ため使いません。
- **測る値**：p50、p90、p99、最大値、エラー率
- **測る場所**：クライアント側。アプリも処理時間をログに出し、差し引けるようにします。
- **接続の内訳**（L3 用）：`curl -w` で、名前解決・TCP 接続・最初の 1 バイトまでの時間を分けて記録します。

### B.2 そろえる条件

| 条件 | 値 |
|------|----|
| リージョン | 1 つに固定（実施時に決めて記録する） |
| クライアント | 同じインスタンスタイプを AZ ごとに 1 台 |
| 応答 | 固定の JSON（1 KB 未満） |
| リクエスト率 | 50 リクエスト/秒と 500 リクエスト/秒 |
| 1 回の測定 | ウォームアップ 60 秒の後、本測定 300 秒。測定ごとに接続を張り直す（Lattice の接続は最長 10 分のため） |
| 繰り返し | 3 回。時間帯を変える |
| 順番 | 経路を測る順を、回ごとに入れ替える |

### B.3 AZ のそろえ方

- AZ は名前ではなく AZ ID（例：`apne1-az1`）でそろえます。アカウントが違うと、同じ名前の AZ が別の場所を指すことがあるためです。
- ターゲットは、クライアントと同じ AZ だけに登録します。Lattice からターゲットへの振り分けは、AZ が固定されないためです（E の Q8）。PrivateLink 側もそろえます。
- PrivateLink では、クライアントと同じ AZ のエンドポイントの DNS 名を使い、NLB のクロスゾーン負荷分散を無効にします。

### B.4 判定の数値と見直し

- **「差がある」の条件**：3 回の測定それぞれの値で、p50 で 0.5 ms 以上、または p99 で 2 ms 以上の差があり、3 回とも同じ向き。
- **数値の理由**（依頼者から一任）：同じ AZ 内の往復時間を 0.1〜0.3 ms と見込み（実測ではない）、その 2 倍程度を超える上乗せを差とみなします。p99 は揺れが大きいため 4 倍の幅をとります。
- **見直し**：最初に基準を 3 回測り、p50 の幅（最大 − 最小）が 0.25 ms を超えたら、条件をその幅の 2 倍に引き上げます。**ほかの経路を測る前に** 決め、理由を `results/notes.md` に書きます。
- **無効とする測定**：エラー率が 0.1% を超えたもの。原因を記録して測り直します。
- L2 の「1 ms 未満」は「気にしなくてよい上乗せ」の目安で、「差がある」の条件とは別のものです。

### B.5 セキュリティのテストの具体的な試し方

| ID | PrivateLink | VPC Lattice |
|----|-------------|-------------|
| S1 | 許可していないアカウントから、エンドポイントの作成を試みる。「承認が必要」を有効にして、承認の前後で通信できるかも見る | サービスネットワークを共有（AWS RAM）していないアカウントから、関連付けを試みる |
| S2 | IAM・パス単位の制御ができないことを確かめる（エンドポイントポリシーは効かない：E の Q9）。セキュリティグループで特定のクライアントに絞れることは確かめる | 認証ポリシーで、ロール A は許可・ロール B は拒否、`GET /read` は許可・`POST /write` は拒否にして呼ぶ。VPC の関連付けのセキュリティグループでも絞る |
| S3 | ポート 8081 経由で呼び、Proxy Protocol v2 の送信元 IP（IPv6）とエンドポイント ID を見る。提供側アカウントでエンドポイント接続の一覧を取り、アカウント ID までたどる | 認証なし・ありの両方で呼び、`x-amzn-lattice-network`、`x-amzn-lattice-identity` などのヘッダー（E の Q6）を見る。認証なしでも VPC のヘッダーが付くかを見る |
| S4 | ECS タスクのセキュリティグループを NLB からだけに絞り、NLB を通らない通信が拒否されることを確かめる | クライアントが `x-amzn-lattice-identity`、`x-amzn-lattice-network`、`x-forwarded-for` を自分で付けて送り、提供側に何が届くかを見る |
| S5 | VPC フローログとアプリのログ（Proxy Protocol v2 から取り出した値）で、呼び出し元を特定できるかを見る。NLB のアクセスログは平文では出ない（E の Q10） | Lattice のアクセスログで、呼び出し元の VPC やプリンシパルを特定できるかを見る |

すべてのセキュリティのテストで、IPv6 でも制御が効くことを確かめます。セキュリティグループのルールは、IPv6 の CIDR（クライアントのサブネットなど）か、セキュリティグループの参照で書きます（`::/0` では絞り込みの確認にならない）。Lattice 側は IPv6 用のプレフィックスリスト（`com.amazonaws.<region>.ipv6.vpc-lattice`）を使います。

## C. 記録とログの設定

### C.1 記録の置き場所

| 記録 | 置き場所 |
|------|----------|
| 構築の確かめ結果 | `results/build-check.md` |
| 測定の生データ | `results/raw/<テストID>/<経路>-<回>.json` |
| 測定の集計 | `results/latency-summary.csv`（列：`test_id, route, keepalive, rps, run, p50_ms, p90_ms, p99_ms, max_ms, error_rate, started_at`） |
| セキュリティの結果 | `results/security-checks.md`（テストごとに、経路別の判定 ◯ / △ / ✗ と根拠） |
| セキュリティの証跡 | `results/evidence/<テストID>/`（`curl -v` の出力、該当するログの行、関係する設定の `describe-*` の出力） |
| 実行ログ | `results/run-log.md`（実行したコマンド、開始・終了の時刻（UTC と JST）、クライアントの AZ ID、結果の要約） |
| 構成 | `results/environment/`（測定の前後の `describe-*` の出力。前後で差がないことを確かめる） |
| 気づいたこと | `results/notes.md` |
| 持ち出したログ | `results/logs/`（**リソースを削除する前に** 出力する） |

### C.2 AWS 側で有効にするログ

| ログ | 設定 | 使うテスト |
|------|------|------------|
| VPC フローログ | 両方の VPC。送り先は S3。`pkt-srcaddr`、`pkt-dstaddr`、`az-id` を含むカスタム形式にし、IPv6 のアドレスが残ることを確かめる。S3 に置けば Athena で集計できる | S2、S4、S5 |
| VPC Lattice のアクセスログ | サービスネットワークとサービスの両方。送り先は CloudWatch Logs | S2、S5 |
| ECS タスクのアプリログ | `awslogs` で CloudWatch Logs へ | S3〜S5、L1〜L3 の補助 |
| AWS CloudTrail | 両方のアカウント。管理イベントを S3 へ。接続の承認、認証ポリシー、セキュリティグループの変更を残す | S1、S2 |

### C.3 時刻と伏せ字

- クライアントは Amazon Time Sync Service で時刻を合わせ、記録はすべて UTC で残します。
- `results/` には、アカウント ID、VPC の ARN、IP アドレスが含まれます（`x-amzn-lattice-network` の値にもアカウント ID が入る）。外に出す前に伏せ字にします。

## D. 追加テスト（必要なときだけ）

本文のテストで判断がつかないときや、要件があるときに行います。

| ID | 内容 | 予想 | 行うとき |
|----|------|------|----------|
| D1 | AZ をまたいで呼ぶ | AZ をまたぐことによる増え方は、経路どうしの差より大きい | 呼び出し側と提供側の AZ をそろえられない場合 |
| D2 | IPv4 で呼んで IPv6 と比べる | 差は見られない | IPv4 との混在を続ける場合 |
| D3 | TLS ありで比べる（PrivateLink は ECS まで TLS、Lattice は HTTPS リスナーで終端） | Lattice のほうが遅い差が広がる | 本番で TLS を使う場合。終端の場所が違うことを結果に必ず書く。NLB のアクセスログはこのときだけ出る |
| D4 | IPv6 のみのサブネットで構築する | 構築できるかどうか自体が不明 | 前提を「IPv6 のみ」に切り替える場合（E の Q7）。Lattice は、サービスネットワーク型の VPC エンドポイントで入る方法も比べる |
| D5 | NLB 自体の上乗せを、提供側 VPC の中から NLB を直接呼んで測る | 小さい | L1 で PrivateLink の上乗せが予想より大きかった場合の切り分け |

## E. 公式ドキュメントで確かめたこと

2026-11-24 に AWS の公式ドキュメントで確かめました。確かめたのは「ドキュメントにどう書かれているか」までで、実際の動きはゲートとテストで確かめます。

判定：確認済み＝明記あり／一部確認＝一部だけ明記、または読み取りによる／未確認＝見つからない、または資料が食い違う

| ID | 確かめたこと | 判定 | 結果 | 本文への反映 |
|----|--------------|------|------|--------------|
| Q1 | PrivateLink の IPv6 対応 | 確認済み | NLB をデュアルスタックにし、エンドポイントサービスで IPv6 を許可すれば対応。エンドポイントの IPv6 タイプは IPv6 のみのサブネット専用なので、デュアルスタックを選ぶ | G2 |
| Q2 | NLB に ECS を IPv6 で登録できるか | 一部確認 | ECS のドキュメントから、できると読める | 構成 |
| Q3 | Lattice の IPv6 対応 | 一部確認 | ターゲットは IPv6 に対応。IPv6 用のプレフィックスリストもある。**クライアントが IPv6 で接続できるかは明記なし。資料が食い違う** | G1 |
| Q4 | PrivateLink で送信元がどう見えるか | 確認済み | そのままでは NLB の IP。Proxy Protocol v2 で送信元とエンドポイント ID（TLV `0xEA` / `0x01`）が渡る。ヘッダーを信じるなら、ECS を NLB からだけに絞る | S3、S4 |
| Q5 | ECS を Lattice に直接つなげるか | 確認済み | つなげる | 構成 |
| Q6 | Lattice が付けるヘッダー | 確認済み | 下の表のとおり。`x-amzn-lattice-*` は偽装できない（Lattice が取り除く）。`x-forwarded-*` は明記なし | S3、S4 |
| Q7 | IPv6 のみのサブネット | 一部確認 | エンドポイントは可。Fargate は資料が食い違う。Lattice の関連付けは明記なし | D4 |
| Q8 | Lattice の AZ | 確認済み | クライアント → Lattice は同じ AZ。Lattice → ターゲットは複数の AZ に分かれうる。Lattice 内の AZ 間転送は無料 | B.3 |
| Q9 | PrivateLink のエンドポイントポリシー | 一部確認 | AWS 以外のサービス向けはフルアクセスになり、絞れないと読める | S2 |
| Q10 | NLB のアクセスログ | 確認済み | TLS リスナーのときだけ出る | S5、D3 |
| Q11 | Lattice の制限 | 一部確認 | アイドルタイムアウトは既定で 60 秒、接続は最長 10 分。1 つの VPC が関連付けられるサービスネットワークは 1 つ | B.2 |
| Q12 | ECS に NLB と Lattice を同時につなげるか | 未確認 | どちらのパラメーターもあるが、組み合わせの明記なし | G3 |

**Lattice が提供側に付けるヘッダー（Q6）**

| ヘッダー | 中身 | 付く条件 |
|----------|------|----------|
| `x-forwarded-for` | 送信元 IP | 明記なし |
| `x-forwarded-port`、`x-forwarded-proto` | 宛先のポート、`http` / `https` | 明記なし |
| `x-amzn-lattice-network` | `SourceVpcArn`（呼び出し元 VPC の ARN。アカウント ID を含む） | 明記なし |
| `x-amzn-lattice-target` | サービス、サービスネットワーク、ターゲットグループの ARN | 明記なし |
| `x-amzn-lattice-identity` | `Principal`、`PrincipalOrgID` など | AWS の認証に成功したときだけ |
| `x-amzn-lattice-identity-tags` | プリンシパル ID とタグ | 明記なし |

**出典（2026-11-24 に参照）**

- [Share your services through AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-share-your-services.html)（Q1）
- [Access an AWS service using an interface VPC endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)（Q1）
- [Create a service powered by AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/create-endpoint-service.html)（Q1）
- [Edit target group attributes for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html)（Q4）
- [Access logs for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-access-logs.html)（Q10）
- [Control access to VPC endpoints using endpoint policies](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-access.html)（Q9）
- [Target groups in VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/target-groups.html)、[Document history for the Amazon VPC Lattice User Guide](https://docs.aws.amazon.com/vpc-lattice/latest/ug/doc-history.html)（Q3）
- [Control traffic in VPC Lattice using security groups](https://docs.aws.amazon.com/vpc-lattice/latest/ug/security-groups.html)（Q3）
- [HTTP targets in VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/http-targets.html)（Q6）
- [Control access to VPC Lattice services using auth policies](https://docs.aws.amazon.com/vpc-lattice/latest/ug/auth-policies.html)（本文の S2。認証ポリシーの存在）
- [Access logs for Amazon VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/monitoring-access-logs.html)（本文の S5。アクセスログに `sourceVpcId` が含まれる）
- [How VPC Lattice works](https://docs.aws.amazon.com/vpc-lattice/latest/ug/how-it-works.html)（Q8）
- [Quotas for Amazon VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/quotas.html)（Q11）
- [Create a service that uses VPC Lattice（Amazon ECS）](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-vpc-lattice-create-service.html)（Q5）
- [Use Amazon VPC Lattice to connect, observe, and secure your Amazon ECS services](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-vpc-lattice.html)、[CreateService（Amazon ECS API）](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_CreateService.html)（Q12）
- [Amazon ECS task networking options for Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)（Q2、Q7）
- [AWS services that support IPv6](https://docs.aws.amazon.com/vpc/latest/userguide/aws-ipv6-support.html)（Q3、Q7）
- [Access a service network through a service-network endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/access-with-service-network-endpoint.html)（Q7）

## F. 前提と範囲

**前提**

| 前提 | 理由 |
|------|------|
| 呼び出し側と提供側は、同じリージョン・同じ AWS Organizations 配下の別アカウント | 提供側から呼び出し元のアカウントを識別できるかを確かめるため |
| 「IPv6 を前提」は、デュアルスタック VPC で IPv6 で通信すること | 依頼者と決めた（2026-11-24、「デュアルスタックでいったん」）。IPv6 のみは D4 |
| ECS は Fargate、`awsvpc` で動く HTTP API。応答は固定の小さな JSON | アプリの処理時間の揺れを除くため |
| 比べるのは PrivateLink と VPC Lattice だけ（VPC ピアリングや Transit Gateway は比べない） | 依頼の範囲どおり |

**範囲外**：AWS 上での実行、IaC のコード、スループットの上限やスケール時の挙動、料金の比較、複数リージョン、gRPC と HTTP/2

**費用のメモ**：NLB、インターフェイスエンドポイント、VPC Lattice（時間料金と処理量の料金）、Fargate、クライアント、AZ 間のデータ転送（Lattice 内は無料）に料金がかかります。実施の前に見積もり、終わったらすぐ削除します。
