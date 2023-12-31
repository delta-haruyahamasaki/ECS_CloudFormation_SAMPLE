## 概要
AmazonECSクラスター環境、AWSCodeシリーズを使用したCI/CD環境を構築するCloudFormationサンプルです。
VPC、Auroraスナップショット、コンテナイメージを指定し実行することで、指定のVPC環境に既存のAmazonECS環境を複製できます。

## 構成
![構成図](./image/構成図_ECS_Sample.png)

## CloudFormation テンプレートファイル
#### リソース構築用ファイル
スタックのネストを利用し、機能別にファイルを分けています。
ネストされたスタックファイルをS3にアップロードし、ルートスタックファイルでS3のファイルを指定します。
<br>参考：<br>https://dev.classmethod.jp/articles/create-own-verification-environment-using-cfn-nested-stacks/
- ルートスタック
  - Sample_CF_Root.yaml
- ネストされたスタック
  - Sample_CF_Aurora.yaml
  - Sample_CF_Network.yaml
  - Sample_CF_TaskDefinition.yaml
  - Sample_CF_ECS.yaml
  - Sample_CF_CICD_Role.yaml
  - Sample_CF_CodeBuild.yaml
  - Sample_CF_CodeDeploy.yaml
  - Sample_CF_CodePipeline.yaml
  - Sample_CF_SSM.yaml

## テンプレートファイル別 自動作成リソース
#### Sample_CF_Root.yaml
- CloudFormation実行時に使用するルートスタック。すべてのネストされたスタックを作成します。

#### Sample_CF_Aurora.yaml
- Aurora
  - AuroraDBクラスター
  - AuroraDBインスタンス

#### Sample_CF_Network.yaml
- ネットワーク
  - Network Load Balancer
  - TargetGroup(Blue)
  - TargetGroup(Green)
  - Listener(Blue)
  - Listener(Green)
  - Route53 レコード

#### Sample_CF_TaskDefinition.yaml
- タスク定義
  - コンテナ用LogGroup
  - タスク実行ロール
  - タスク定義

#### Sample_CF_ECS.yaml
- ECSクラスター
  - ECSクラスター
  - ECSサービス

#### Sample_CF_CICD_Role.yaml
- CI/CD用 IAM Role, Policy
  - CodeBuild IAM Role
  - CodeBuild IAM Policy
  - CodeDeploy IAM Role
  - CodeDeploy IAM Policy
- カスタムリソースLambda用 IAM Role, Policy
  - Lambda IAM Role
  - Lambda IAM Policy

#### Sample_CF_CodeBuild
- Code Build
  - CodeBuild用LogGroup
  - CodeBuild

#### Sample_CF_CodeDeploy
- Code Deploy
  - アプリケーション
  - デプロイグループ

#### Sample_CF_CodePipeline
- Blue Green Artifact S3バケット
  - S3バケット
- Code Pipeline

#### Sample_CF_SSM
- カスタムリソースLambda用 IAM Role
- SSMパラメータストア

## 実行時に必要なパラメータ
#### Sample_CF_Root.yaml
- 環境(hotfix/candidate)
- ネストされたスタックファイル格納 S3バケット
- Auroraクラスター名
- Auroraインスタンス名
- Auroraクラスタースナップショット名
- インスタンスクラス
- サブネットグループ名
- セキュリティグループ
- NLB サブネット
- ターゲットグループ VPC ID
- Route53 ホストゾーンName
- Route53 ホストゾーンID
- ECSサービス作成時に指定するコンテナイメージ
- ECSサービス セキュリティグループ
- ECSサービス サブネット
- CodePipeline Sorce S3バケット名
- CodePipeline Sorce S3オブジェクトキー

## 実行時に必要なリソース
#### Sample_CF_Root.yaml
- Auroraクラスタースナップショット
- サブネットグループ
- セキュリティグループ
- ネストされたスタックファイル格納用 S3バケット
- VPC/サブネット
- ECR コンテナイメージ
- ECS用 セキュリティグループ
- CodePipeline Sorce S3バケット
- CodePipeline Sorce オブジェクト
- Systems Manager パラメータストア
- Route53 ホストゾーン

## 詳細
#### Auora
- パラメータに指定したAuroraスナップショットが復元されます。

#### NLB
- コンテナ接続用のNLBです。
- NLB作成後、指定のRoute53ホストゾーンにDNS名のエイリアスレコードが追加されます。
- BGデプロイを行うため、Blue用、Green用のターゲットグループが作成され、それぞれリスナーに登録されます。
- BlueはTCP:80、GreenはTCP:10080のプロトコル:ポートでそれぞれのターゲットグループへ転送されます。

#### ECSタスク定義
- CF実行時に指定したECRイメージがタスク定義に登録されます。
- パラメータストアの情報を参照する設定があり、サンプルでは参照先の値がコードにべた書きされています。「/SAMPLE/DB_HOST」「/SAMPLE/DB_NAME」等DBへの接続情報を参照する設定がありますので、必要な値に置き換えてご利用ください。

#### ECSクラスター
- ECSクラスターとECSサービスが作成されます。
- 初回構築時は前項で作成したタスク定義のECSタスクがデプロイされます。

#### CodeBuild
- CodeBuild用ロググループとCodeBuildが作成されます。
- サンプルでは以下の実行環境設定をしています。
  - 環境イメージ：aws/codebuild/amazonlinux2-x86_64-standard:4.0
  - 環境タイプ：Linux
  - コンピューティング：3 GB メモリ、2 vCPU

#### CodeDeploy
- ECS BGデプロイメントグループはCloudFormationでサポートされていないことから、CodeDeployはAWS Lambda-backed カスタムリソースを利用しています。
- CFコード内のLambda関数にはCFスタック作成時にCodeDeployを作成、CFスタック削除時にCodeDeployを削除する処理が記載されています。
  - 参考：<br>https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-custom-resources-lambda.html<br>https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-custom-resources.html

#### CodePipeline
- アーティファクト用S3バケットとCodePipelineが作成されます。
- パラメータで指定したS3バケット、オブジェクトキーが、Sourceステージに登録されます。
- CFスタックを削除する際、アーティファクト用S3バケット内のオブジェクトを手動ですべて削除する必要があります。

#### SSMパラメータストア
- 構築したDBのエンドポイント名が「/SAMPLE/DB_HOST」と命名されたSSMパラメータストアに追加されます。
- SecureString パラメータタイプがCloudFormationでサポートされていないことから、CodeDeployはAWS Lambda-backed カスタムリソースを利用しています。


## 使用の流れ
1. ネストされたスタックファイルをS3バケットにアップロード。
2. Sample_CF_Root.yamlを実行し、新規環境を構築。
3. 環境構築後、S3バケットにソースコードのZIPファイルをアップロード。パイプラインが起動しコンテナがデプロイされる。
4. 指定のドメインで構築した環境にアクセス。