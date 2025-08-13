## はじめに

「個人学習でEC2を気軽に立てては壊したい」って時、ありますよね。
この記事は、エンジニア見習いでも簡単に編集できるCSVファイルで、検証環境が手に入るようにしたサーバレスアプリケーションの紹介です。

**[GitHubリポジトリへのリンク](https://github.com/RT-art/csv-to-ec2.git)** 

## プロジェクトの概要とアーキテクチャ

シェル1コマンド実行するだけで、必要なAWSリソース一式(VPC,Subnet,s3,Lambda等)をデプロイ。
CSVをアップロードするだけでEC2を起動できます。

![アーキテクチャ図](https://github.com/RT-art/csv-to-ec2/raw/main/images/%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3%E5%9B%B3.png)

## 使い方
レポジトリをクローン
↓
必須ツールをインストール（AWS CLI、AWSSAMCLI、Python3.12）
↓
deployコマンドを打つ
↓
s3 pushコマンドを打つ
↓
完成

deleteコマンドで作成したすべてのAWSリソースを一括削除


### 1\. 初期設定 (初回のみ)

#### プロジェクトのダウンロード

ターミナルで以下のコマンドを実行してください。

```bash
git clone https://github.com/RT-art/csv-to-ec2.git
cd csv-to-ec2
```

必須ツールのインストール

AWS CLI, AWS SAM CLI, Python 3.12 が必要です。お使いのOSに合わせてインストールしてください。
※準備ができている方は、飛ばしていただいて構いません。
#### macOS (Homebrewを使用)

ターミナルを開き、以下のコマンドを実行します。

```bash
# Homebrewが未導入の場合は先にインストール
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 各ツールを一括でインストール
brew install awscli aws-sam-cli python@3.12
```
#### Windows (Git Bash等)

1.  [AWS CLI公式ダウンロードページ](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html)からインストーラー（.msi）をダウンロードし、実行します。
2.  [Python 3.12公式サイト](https://www.python.org/downloads/windows/)からインストーラーをダウンロードしてインストールします。**このとき、必ず「Add Python 3.12 to PATH」のチェックボックスをオンにしてください。**
3.  [Git for Windows](https://git-scm.com/download/win)をインストールし、Git Bashを起動します。
4.  Git Bash上で、pipを使いAWS SAM CLIをインストールします。

    ```bash
    pip install aws-sam-cli
    ```
5.  以下のコマンドで各ツールのバージョンが表示されれば成功です。
    ```bash
    aws --version
    sam --version
    python --version
    ```

#### Windows (WSL2)

ターミナルを開き、以下のコマンドを順番に実行します。

```bash
# パッケージリストを更新し、Pythonの前提パッケージをインストール
sudo apt update
sudo apt install -y software-properties-common

# Python 3.12 をインストール (deadsnakes PPA を使用)
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt install -y python3.12

# AWS CLI をインストール
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws/

# AWS SAM CLI をインストール
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
rm -rf aws-sam-cli-linux-x86_64.zip sam-installation/
```

#### インストール後、以下設定を行う
```bash
# AWSのアクセスキーとシークレットアクセスキーなどを設定
aws configure

# スクリプトに実行権限を付与
chmod +x manage.sh
```
-------

### 2\. デプロイスクリプトの実行

以下のコマンドで、AWS上に必要なリソース（VPC, S3, Lambda等）を一括作成します。

```bash
./manage.sh deploy
```
-----

### 3\. EC2インスタンスの作成

`sample.csv` を編集し、作成したいインスタンスのAMI IDとインスタンスタイプを記述します。

```csv:sample.csv
ami_id,instance_type
ami-0dc279da1d8643e74,t2.micro
```

その後、以下のコマンドでCSVをアップロードすれば、EC2の作成が自動で開始されます。

```bash
# カレントディレクトリの sample.csv を自動でアップロード
./manage.sh upload

# ファイル名を指定してアップロードも可能
./manage.sh upload my_servers.csv
```

-----

### 4\. クリーンアップ

作成したEC2インスタンスやVPCなど、すべてのリソースを一発で削除します。

```bash
./manage.sh delete
```

## つまづいた点と学び

### 1\. 循環参照エラーとの戦い

当初、S3バケットの作成と、そのバケットをトリガーにするLambda関数の権限設定で、深刻な循環参照エラーに陥りました。

  * **S3バケット**は、自身が変更されたときに通知する**Lambda関数**を知る必要がある。
  * **Lambda関数**は、トリガーとなる**S3バケット**から読み取るためのIAM権限を付与される必要がある。

`DependsOn` 属性で依存関係を明示しようとしても解決できず、AWS SAMの `Events` プロパティを使っても同様のエラーが発生しました。

**【エラーの核心】**
AWS SAMの`Events`プロパティは、S3イベントをトリガーにLambdaを起動するための`lambda:InvokeFunction`権限と、S3バケットの通知設定を自動で作成してくれます。しかし、Lambda関数コード内で`s3:GetObject`を実行するための**読み取り権限**までは面倒を見てくれません。この読み取り権限を`AWS::Serverless::Function`リソース内の`Policies`で定義しようとすると、「S3 ←→ Lambda」の相互参照が発生してしまう、というのがエラーの核心でした。


**【解決策】**
最終的に、**LambdaのS3読み取り権限を定義するIAMポリシー (`S3ReadPolicyForLambda`) を、`AWS::Serverless::Function` リソースから完全に分離して定義する**ことで、この依存関係のループを断ち切ることに成功しました。これによりCloudFormationは明確な順序でリソースを作成できるようになり、エラーは解消されました。

```yaml:template.yaml
# (抜粋)
# EventsプロパティでS3トリガーとLambdaの実行権限は自動作成されるが、
# S3からのGetObject権限は循環参照を避けるため別に定義する

S3ReadPolicyForLambda:
  Type: AWS::IAM::Policy
  Properties:
    PolicyName: !Sub "${AWS::StackName}-S3ReadPolicyForLambda"
    Roles:
     # CsvToEc2FunctionリソースからSAMが自動で作成する実行ロール名を参照
      # samが自動で作ったロールに、s3の読み取り権限を追加
      - !Ref CsvToEc2FunctionRole
    PolicyDocument:
      Statement:
        - Effect: Allow
          Action: "s3:GetObject"
          Resource: !Sub "arn:aws:s3:::${CsvFileBucket}/*"
```

### 2\. 設計のシンプル化：Step Functionsからの脱却

実は当初、AWS Step Functionsを導入し、EC2作成後の状態監視や通知など、より多機能なワークフローを構想していました。しかし、設計を進めるうちにLambda関数が複数必要になったり、CFnの呼び出しが複雑化したりと、**「やりたいこと」に対して仕組みがとんでもなく複雑になってしまいました。**

そこで一度立ち止まり、「本当に必要な要件は何か？」を自問しました。答えは「CSVからEC2を手軽に作りたい」という一点です。この原点に立ち返り、Step Functionsの導入をやめ、S3トリガーと単一Lambdaという現在のシンプルな構成に舵を切りました。

この経験から、**技術は目的を達成するための手段であり、常にシンプルな解決策を模索する重要性**を改めて学びました。

## 今後の展望

このプロジェクトはまだ発展途上です。今後、以下のような機能追加を考えています。

  * **セキュリティグループの動的設定**: CSVにポート番号などを記述し、インスタンスごとに適切なSGを割り当てる。
  * **UserDataの活用**: インスタンス起動時に実行したいシェルスクリプトをCSVで指定できるようにする。
  * **SQSの導入**: Lambdaの前にSQSを挟むことで、大量のEC2インスタンス作成リクエストを非同期に処理し、Lambdaのタイムアウトを気にせずスケールできるようにする。

## おわりに

このプロジェクトは、「学習コストを下げて、もっと気軽にAWSに触れる環境を作りたい」という、自分自身の課題感からスタートしました。結果として、コマンド一つでインフラを管理できる手軽さは、想像以上に学習効率を高めてくれました。

この記事が、AWSの学習を始めたばかりの方や、日々の検証作業を効率化したいと考えているエンジニアの方の助けになれば、これほど嬉しいことはありません。
