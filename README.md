## 概要
AmazonECSクラスター環境、AWSCodeシリーズを使用したCI/CD環境を構築するCloudFormationサンプルです。
VPC、Auroraスナップショット、コンテナイメージを指定し実行することで、指定のVPC環境に既存のAmazonECS環境を複製できます。

## 構成
![構成図](./image/構成図_ECS_Sample.png)

## 自動で作成されるリソース
#### Sample_CF_Aurora.yaml
- Aurora
  - AuroraDBクラスター
  - AuroraDBインスタンス

#### Sample_CF_ECS_CICD.yaml
- ネットワーク
  - Network Load Balancer
  - TargetGroup(Blue)
  - TargetGroup(Green)
  - Listener(Blue)
  - Listener(Green)

<br>

- タスク定義
  - コンテナ用LogGroup
  - タスク実行ロール
  - タスク定義

<br>

- ECSクラスター
  - ECSクラスター
  - ECSサービス

<br>

- CI/CD用 IAM Role, Policy
  - CodeBuild IAM Role
  - CodeBuild IAM Policy
  - CodeDeploy IAM Role
  - CodeDeploy IAM Policy

<br>

- カスタムリソースLambda用 IAM Role, Policy
  - Lambda IAM Role
  - Lambda IAM Policy

<br>

- Blue Green Artifact S3バケット
  - S3バケット

<br>

- Code Build
  - CodeBuild用LogGroup
  - CodeBuild

<br>

- Code Deploy
  - アプリケーション
  - デプロイグループ

## 実行時に必要なパラメータ
#### Sample_CF_Aurora.yaml
- Auroraクラスター名
- Auroraインスタンス名
- Auroraクラスタースナップショット名
- インスタンスクラス
- サブネットグループ名
- セキュリティグループ

#### Sample_CF_ECS_CICD.yaml
- NLB サブネット
- ターゲットグループ VPC ID
- ECSサービス作成時に指定するコンテナイメージ
- ECSサービス セキュリティグループ
- ECSサービス サブネット
- CodePipeline Sorce S3バケット名
- CodePipeline Sorce S3オブジェクトキー

## 実行時に必要なリソース
#### Sample_CF_Aurora.yaml
- Auroraクラスタースナップショット
- サブネットグループ
- セキュリティグループ

#### Sample_CF_ECS_CICD.yaml
- VPC/サブネット
- ECR コンテナイメージ
- ECS用 セキュリティグループ
- CodePipeline Sorce S3バケット
- CodePipeline Sorce オブジェクト
- Systems Manager パラメータストア

## 詳細
#### NLB
- コンテナ接続用のNLBです。
- 構築後DNS名でアクセス、またはDNS名をRoute53レコードに登録いただくことで指定のドメインからコンテナにアクセス可能です。
- BGデプロイを行うため、Blue用、Green用のターゲットグループが作成され、それぞれリスナーに登録されます。
- BlueはTCP:80、GreenはTCP:10080のプロトコル:ポートでそれぞれのターゲットグループへ転送されます。

#### ECSタスク定義
- CF実行時に指定したECRイメージがタスク定義に登録されます。
- パラメータストアの情報を参照する設定があり、サンプルでは参照先の値がコードにべた書きされています。「/SAMPLE/DB_HOST」「/SAMPLE/DB_NAME」等DBへの接続情報を参照する設定がありますので、必要な値に置き換えてご利用ください。

#### ECSクラスター
- ECSクラスターとECSサービスが作成されます。
- 初回構築時は前項で作成したタスク定義のECSタスクがデプロイされます。
- ECSサービスはNLB作成後に構築を開始する必要があるため、サンプルではNLBのリスナーにターゲットグループが登録されたのちに構築が開始されるよう依存設定を追加しています。

#### CodeBuild
- CodeBuild用ロググループとCodeBuildが作成されます。
- サンプルでは以下の実行環境設定をしています。
  - 環境イメージ：aws/codebuild/amazonlinux2-x86_64-standard:4.0
  - 環境タイプ：Linux
  - コンピューティング：3 GB メモリ、2 vCPU

#### CodeDeploy
- 