---
title: "ECS on FargateのAuto Scaling を CDK で試す"
published: true
cssclass: zenn
emoji: "⚖️"
type: "tech"
topics: ["AWS", "AWS CDK"]
url: "https://zenn.dev/yamaren/articles/df1eaa00e46fca"
---


こんにちは。Web バックエンドエンジニアをしている[山梨](https://twitter.com/yama_ren_tw)といいます。  

## 1. 概要

この記事では、CDK を使用して ECS の Auto Scaling を設定する方法を記します。  
また、Auto Scaling の内容として以下の 2 点について紹介します。

- 自動スケーリング
- スケジュールされたスケーリング

### 1.1 自動スケーリング

自動スケーリングとは、ECS サービスが負荷に応じてタスク数を自動的に増減させる機能です。  
今回は、以下の条件に基づいて自動でスケーリングが行われるように設定します

- CPU の平均使用率が 30%以上の場合にタスクを 1 つ増加 (スケールアウト)
- CPU の平均使用率が 20%以下の場合にタスクを 1 つ減少 (スケールイン)

※1. スケールアウト: システムを構成する仮想マシン (今回の場合はタスク) を増やすこと
※2. スケールイン: システムを構成する仮想マシン (今回の場合はタスク) を減らすこと

### 1.2 スケジュールされたスケーリング

スケジュールされたスケーリングとは、あらかじめスケールインあるいはスケールアウトする時間・曜日を指定し、その内容に応じて ECS サービスのタスク数を増減させる機能です。  
今回は、以下のようにタスク数が自動的に調整される設定を行います

- 8 時にスケールアウト (タスク数の最小値を 3 に増やす)
- 18 時にスケールイン (タスク数の最小値を 1 に減らす)

## 2. 実装

:::message
この記事では前提として、TypeScript, CDK, AWS CLI のインストールは済んでいるものとして記述しております。
:::

この記事では、CDK の [`ApplicationLoadBalancedFargateService`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns.ApplicationLoadBalancedFargateService.html) という L3 Construct を使用して構築していきます。

今回作成した Stack 全体のコードは、以下の通りです。

```ts
// lib/server-stack.ts

import * as path from "node:path";
import type { StackProps } from "aws-cdk-lib";
import { CfnOutput, Duration, Stack, TimeZone } from "aws-cdk-lib";
import { Schedule } from "aws-cdk-lib/aws-applicationautoscaling";
import type { Construct } from "constructs";
import { Vpc } from "aws-cdk-lib/aws-ec2";
import { ContainerImage, CpuArchitecture, FargateTaskDefinition, OperatingSystemFamily } from "aws-cdk-lib/aws-ecs";
import { ApplicationLoadBalancedFargateService } from "aws-cdk-lib/aws-ecs-patterns";
import { MetricAggregationType } from "aws-cdk-lib/aws-autoscaling";

export class ServerStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // NOTE: VPCの作成
    const vpc = new Vpc(this, "Vpc", { maxAzs: 2 });

    // NOTE: タスク定義の作成
    const taskDefinition = new FargateTaskDefinition(
      this,
      "TaskDefinition",
      {
        runtimePlatform: {
          operatingSystemFamily: OperatingSystemFamily.LINUX,
          cpuArchitecture: CpuArchitecture.ARM64,
        },
      },
    );
    taskDefinition.addContainer("AppContainer", {
      image: ContainerImage.fromAsset(path.resolve(__dirname, "../")),
      portMappings: [{
        containerPort: 80,
        hostPort: 80,
      }],
    });

    // NOTE: Fargate起動タイプでサービスの作成
    const fargateService = new ApplicationLoadBalancedFargateService(this, "FargateService", {
      taskDefinition,
      vpc,
    });

    // NOTE: オートスケーリングのターゲット設定
    const scaling = fargateService.service.autoScaleTaskCount({
      minCapacity: 1,
      maxCapacity: 5,
    });

    // NOTE: CPU使用率に応じてスケールアウト・スケールイン
    scaling.scaleOnMetric("StepScaling", {
      metric: fargateService.service.metricCpuUtilization({
        period: Duration.minutes(1), // 1分間隔でCPU使用率を取得
      }),
      scalingSteps: [
        { lower: 30, change: +1 }, // CPUの使用率が30%以上の場合にタスクを1つ増加
        { upper: 20, change: -1 }, // CPUの使用率が20%以下の場合にタスクを1つ減少
      ],
      metricAggregationType: MetricAggregationType.AVERAGE, // 平均値に基づいてスケーリングされるように設定
      cooldown: Duration.minutes(1), // スケーリングのクールダウン期間を1分に設定
    });

    // NOTE: 8時にスケールアウト
    scaling.scaleOnSchedule("ScaleOutSchedule", {
      timeZone: TimeZone.ASIA_TOKYO,
      schedule: Schedule.cron({ hour: "8", minute: "0" }),
      minCapacity: 3,
    });

    // NOTE: 18時にスケールイン
    scaling.scaleOnSchedule("ScaleInSchedule", {
      timeZone: TimeZone.ASIA_TOKYO,
      schedule: Schedule.cron({ hour: "18", minute: "0" }),
      minCapacity: 1,
    });

    // NOTE: 出力としてロードバランサーのDNS名を出力
    new CfnOutput(this, "LoadBalancerDNS", {
      value: fargateService.loadBalancer.loadBalancerDnsName,
    });
  }
}

```

::::details 実装内容の補足 (`scaleOnMetric` の使用理由について)

今回は、CPU 使用率に基づいたスケーリングを行うために、`ScalableTaskCount` クラスの `scaleOnMetric` メソッドを使用しました(該当箇所は以下のコード部分)

```ts
// NOTE: CPU使用率に応じてスケールアウト・スケールイン
scaling.scaleOnMetric("StepScaling", {
  metric: fargateService.service.metricCpuUtilization({
    period: Duration.minutes(1), // 1分間隔でCPU使用率を取得
  }),
  scalingSteps: [
    { lower: 30, change: +1 }, // CPUの使用率が30%以上の場合にタスクを1つ増加
    { upper: 20, change: -1 }, // CPUの使用率が20%以下の場合にタスクを1つ減少
  ],
  metricAggregationType: MetricAggregationType.AVERAGE, // 平均値に基づいてスケーリングされるように設定
  cooldown: Duration.minutes(1), // スケーリングのクールダウン期間を1分に設定
});
```

<br />

実は、この方法の他に `scaleOnCpuUtilization` メソッドを使用する方法もあります。  
`scaleOnCpuUtilization` メソッドを使用する場合は以下のようにコードを記述します。

```ts
// NOTE: CPU使用率が30%を超えたらスケールアウト
scaling.scaleOnCpuUtilization("CpuScaling", {
  targetUtilizationPercent: 30,
  scaleInCooldown: Duration.minutes(1),
  scaleOutCooldown: Duration.minutes(1),
});
```

<br />

こちらの方がシンプルに記述できますが、私が調べた限りでは `scaleOnCpuUtilization` を使用してスケールインする CPU 使用率を指定する方法がわかりませんでした。  

おそらく、以下の内容でスケールインの条件が自動的に作成されるのかなと推察しています  
(`scaleOnCpuUtilization` の `disableScaleIn` を true にしなかった場合)

- targetUtilizationPercent に指定した値を **N** とした時、  
CPU 使用率が **N * 0.9** を下回る状態が 15 分連続で続いた場合にスケールイン

<br />

今回は学習目的としてスケールインの設定も手動で行いたかったため、`scaleOnMetric` メソッドを使用しています。

::::

## 3. 動作確認

### 3.1. デプロイ

実装が完了しましたので、デプロイして動作確認してみます。  
以下のコマンドを上から順番に 1 つずつ実行して、デプロイします。

```bash
cdk bootstrap
cdk deploy
```

デプロイが完了するとロードバランサーの DNS 名が出力されるので、それをコピーしておきます。  
(自動スケーリングの動作確認で使用します)

<br />

### 3.2. 設定内容の確認

デプロイ完了後、ECS のマネジメントコンソールを開いて、Auto Scaling の設定内容が適切に反映されているか確認できます。  
(Auto Scaling の設定は、ECS サービスの「設定とネットワーク」タブから確認できます)

![auto scaling setting in management console](https://storage.googleapis.com/zenn-user-upload/ef3679dbe77b-20241025.png)

<br />

### 3.3. 自動スケーリングのテスト

続いて、アプリケーションに対してサービスの負荷を上げるコマンドを実行し、正常にスケーリングするか確認してみます。
※コマンド実行前のタスク数は 1 の状態でテストしています

::::details タスク数を 1 の状態に戻す方法

以下のコマンドを実行します

```sh
# それぞれの環境変数にはあらかじめ値を代入しておく必要があります
aws ecs update-service \
    --cluster $EcsClusterName \
    --service $EcsServiceName \
    --desired-count 1
```

::::

以下のコマンドを上から順番に 1 つずつ実行して、アプリケーションに対して複数のリクエストを送ります。  

```bash
# ロードバランサーのDNS名を環境変数に代入
export LoadBalancerDNS=sample.elb.amazonaws.com # 先程コピーしたロードバランサーのDNS名
# Apache Benchコマンドを使用して負荷をかける(`-n`でリクエスト数を、`-c`で同時接続数を指定)
ab -n 1000 -c 100 http://$LoadBalancerDNS/
```

<br />

上記のコマンドを実行したあと、ECS メトリクスと ECS イベントから以下の内容が確認できました

- `CPUUtilization Average` が 30%を超えている

![cpu utilization average](https://storage.googleapis.com/zenn-user-upload/4d37ff6fc1b0-20241025.png)

- ポリシーがトリガーされ、実行中のタスクが増加した
  - 時間おきに徐々にタスク数が増えている

![scaling policy triggered](https://storage.googleapis.com/zenn-user-upload/c91eddca3d90-20241025.png)

<br />

また、上記のコマンドを停止した後、CPU 使用率が減少し、それに伴いタスクも減少していることが確認できました

![cpu utilization decrease](https://storage.googleapis.com/zenn-user-upload/6c43d7f3ec0a-20241025.png)

<br />

### 3.4. スケジュールによるスケーリングのテスト

スケジュールによるスケーリングも、正常に動作していることが確認できました。

- 8 時にタスク数が 3 になっている

![scale out at 8am](https://storage.googleapis.com/zenn-user-upload/0fabc4615296-20241025.png)

![scale out at 8am](https://storage.googleapis.com/zenn-user-upload/f30f40c719b1-20241025.png)

- 18 時にタスク数が 1 になっている

![scale in at 6pm](https://storage.googleapis.com/zenn-user-upload/569b4201569b-20241025.png)

![scale in at 6pm](https://storage.googleapis.com/zenn-user-upload/8db694eff428-20241025.png)

## 4. まとめ

ここまでの内容で、ECS on Fargate での Auto Scaling を実装する方法について解説しました。  
参考になれば幸いです！

### 今回作成したGitHubリポジトリ

https://github.com/ren-yamanashi/ecs-auto-scaling
