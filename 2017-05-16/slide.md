slidenumbers: true
autoscale: true

# Lambdaと上手に付き合う
## クラスメソッドのMEETUP
### 〜プログラミングが好きだ！Python、Java、Ruby、Swiftをやってるエンジニア達が語るAWSどっぷりな会社での働き方〜
### 2017/05/16 中山 幸治

---
# 自己紹介

- 中山 幸治
- クラスメソッド AWS事業部 ソリューションアーキテクト
  - AWSを利用したインフラの設計/構築/コンサルティング
- GitHub: [knakayama](https://github.com/knakayama)
- 経歴
  - オンプレサーバの運用3年
  - AWS 1年

![70%, right](images/self-introduce.png)

---
# アジェンダ

1. AWS Lambdaって何(3分)
1. AWS Serverless Application Modelって何(3分)
1. ソースコードはどうやって管理すべきか(3分)
1. Dockerと組み合わせる(3分)
1. まとめ(1分)

---
# 1. Lambdaって何

---
# Lambdaの概要

![100%](images/lambda.png)

- コードを実行できるコンピューティングサービス
  - サーバやプロセスを開発者側で管理する必要がない
  - → アプリケーションの抽象化
- リクエスト/実行時間に応じた従量課金制
  - → 基本的にEC2/ECSより安価になる場合が多い
- 現時点で公式サポートしている言語はNode.js/Python/Java/C#
  - → やろうと思えば別言語も実行可能
- Lambdaを中心としたシステムをサーバレスアーキテクチャと呼ぶ

---
# Lambdaのサンプル

```javascript
module.exports.handler = (event, context, callback) => {
  const response = {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello, Lambda!',
    }),
  };
  callback(null, response);
};
// 結果 {"statusCode":200,"body":"{\"message\":\"Hello, Lambda!\"}"}
```

---
# 2. AWS Serverless Application Modelって何

---
# AWS Serverless Application Modelの概要

![200%](images/sam.jpg)

- サーバレスアーキテクチャを管理するためのモデル
- 略してAWS SAMと呼ばれることが多い
- LambdaなどのAWSサービスをコードで管理可能
  - → Gitで管理可能
  - → GitHubと連携してCI/CDパイプラインを作れる
- 実態はCloudFormationの拡張機能
  - → 学習コストが低い
- デプロイなどにはAWS CLIを利用する

---
# AWS Serverless Application Modelのサンプル

```yaml
---
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31
Description: Hello Lambda

Resources:
  Func1:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/func1
      Handler: index.handler
      Runtime: nodejs6.10

Outputs:
  Func1Name:
    Value: !Ref Func1
```

---
# AWS Serverless Application Modelのデプロイ

```bash
# LambdaのコードなどをS3にアップロード
$ aws cloudformation package \
  --template-file sam.yml \
  --s3-bucket <_YOUR_S3_BUCKET_> \
  --output-template-file .sam/packaged.yml \
# デプロイ
$ aws cloudformation deploy \
  --template-file .sam/packaged.yml \
  --stack-name <_YOUR_STACK_NAME_> \
  --capabilities CAPABILITY_IAM
```

---
# 3. ソースコードはどうやって管理すべきか

---
# ソースコードを管理する際の考え方

- Lambdaのデプロイメントパッケージは必要なもののみ含める
  - → Lambdaの起動時間を短縮するため
  - AWS SAMはパッケージのinclude/excludeが弱いのでテストコードも分離
- リポジトリに全ての情報を含める
  - → The twelve-factor app
- AWS SAMは専用のコマンドが用意されてないのでAWS CLIのラッパースクリプトで頑張る
  - `bin` 以下に配置して `package.json` などから呼び出す
- `aws cloudformation deploy` の `--parameter-overrides` で環境毎のパラメータを分離

---
# これだ！

```bash
.
├── .eslintrc # Lint
├── .gitignore # node_modulesなどを除外
├── .sam # パッケージ化されたAWS SAMテンプレート
│   └── packaged-dev.yml
├── bin # package.jsonのscriptから呼び出すシェルスクリプト
│   └── deploy.sh
├── package.json # 必要なモジュールを管理
├── params # 環境毎のパラメータ
│   └── param.dev.json
├── requirements.txt # AWS CLIもバージョン管理する
```

---
# こうだ！

```bash
├── sam.yml # AWS SAMのテンプレート
├── src # ソースコードなどを配置
│   ├── api # API Gateway用Swaggerファイル
│   │   └── swagger.yml
│   └── handlers # Lambdaのコード
│       └── func1
│           ├── index.js # Lambdaのコード
│           ├── lib
│           │   └── func1.class.js # テストしやすくするためにクラス化してLambda本体とは分離
│           └── package.json # 非標準モジュールを使う場合
└── test # テストコード
    └── func1.spec.js
```

---
# 4. Dockerと組み合わせる

---
# Dockerと組み合わせると嬉しいこと

- Lambdaの実態はAmazon Linuxベースのコンテナ
- 開発環境にMacを利用している場合にモジュールが動かない場合あり
  - → コンパイルが必要なモジュールの場合
  - → 画像処理ライブラリとか
- Amazon LinuxのDockerイメージ使えば簡単にモジュールをインストール可能
  - → コンテナのボリュームを `node_modules` にマウントしてローカルにインストールする
- とはいえ、CI/CDでパッケージインストールする仕組みであれば別にいらないかも
  - → あくまで開発環境で利用する

---
# Dockerfile

```docker
FROM amazonlinux

ADD etc/nodesource.gpg.key /etc
ADD package.json /tmp

WORKDIR /tmp

RUN yum -y install gcc-c++ && \
    rpm --import /etc/nodesource.gpg.key && \
    curl --location --output ns.rpm https://rpm.nodesource.com/pub_6.x/el/7/x86_64/nodejs-6.10.1-1nodesource.el7.centos.x86_64.rpm && \
    rpm --checksig ns.rpm && \
    rpm --install --force ns.rpm && \
    npm install -g npm@latest && \
    npm cache clean && \
    yum clean all && \
    rm --force ns.rpm

ENTRYPOINT npm install --production
```

---
# Dockerコマンド

```bash
# ECRの認証情報取得
$ aws ecr get-login
# ECRにログインしてpull/pushできるようにする
$ docker login -u AWS https://<_YOUR_AWS_ACCOUNT_ID_>.dkr.ecr.ap-northeast-1.amazonaws.com
# イメージのビルド
$ docker build -t amazonlinux:nodejs .
# コンテナの実行
$ docker run --rm -v $PWD/node_modules:/tmp/node_modules amazonlinux:nodejs
```

---
# 5. まとめ

---
# まとめ

- Lambdaヤバイ
- サーバレスアーキテクチャは夢がある
- ハマるとヤバイ
- とはいえ全てのシステムにマッチするわけではない
- でもハマるとヤバイ
- AWSでアーキテクチャを考慮する際に一度は導入を検討してみては

---
# おわり
