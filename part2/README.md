# Part2

## 事前準備

[aws_happy_code - GitHub](https://github.com/ymd65536/aws_happy_code.git)をgitコマンドでDesktop上にcloneします。

## IAMユーザーを作成する

part2ディレクトリにある`iam_user.yml`を使ってハンズオンで利用するIAMユーザーを作成します。
※すでにAdministratorAccessで権限を作成されている場合はこの動作は不要です。

`I acknowledge that AWS CloudFormation might create IAM resources with custom names.`にチェックを入れて`Submmit`をクリックします。

問題なくスタックが作成できましたら`Outputs`からアクセスキーとシークレットアクセスきーをコピーします。

### AWS　CLIをにIAMユーザーを記録する

AWS CLIを設定する為に以下のコマンドを実行します。

```sh
aws configure --profile cicd_handson
```

いくつか質問がなされるので順番に回答します。ここで先ほどのアクセスキーとシークレットアクセスキーを利用します。
`AWS Access Key Id`にアクセスキー、`AWS Secret Access Key`にシークレットアクセスキーを入力します。
リージョンはap-northeast-1、出力形式はjsonで問題ありません。

![9.png](./img/9.png)

最後に設定されているかどうかを確認する為、`credentials`をチェックします。`[cicd_handson]`という項目が追加されていれば問題ありません。

```sh
cat ~/.aws/credentials 
```

### リポジトリを作成する

`aws_happy_code`リポジトリでターミナルを開き、part2にディレクトリを変更します。

```sh
cd ~/Desktop/aws_happy_code/part2
```

以下のコマンドで`codecommit.yml`をCloudFormationで実行します。

```sh
aws cloudformation deploy --stack-name codecommit --template-file ./codecommit.yml --tags Name=cicdhandson --profile cicd_handson
```

### CodeCommitのリポジトリをクローンする

Desktop上にCodeCommitのリポジトリをcloneします。

```sh
cd ~/Desktop
```

```sh
git clone codecommit::ap-northeast-1://cicd_handson@cicdhandson
```

ディレクトリを移動します。

```sh
cd ~/Desktop/cicdhandson
```

### mainブランチを作成

```sh
git checkout -b main
```

```sh
echo "Hello CodeBuild" > README.md
```

```sh
git add .
git commit -m "part2"
git push -u 
```

### code_build_handsonブランチを切る

新しいブランチでビルドを実行する為にCodeBuild用に新しくブランチを切ります。

```sh
git checkout -b code_build_handson
```

### buildspec.yamlを作成する

CodeBuildで利用する設定ファイル(buildspec.yml)を作成します。
Part2ディレクトリにあるbuildspec.ymlを`cicd_handson`リポジトリにコピーします。

```sh
cp ~/Desktop/aws_happy_code/part2/buildspec.yml ~/Desktop/cicdhandson/
```

### dockerfileを作成する

dockerfileを追加します。
`aws_code_happy`リポジトリに戻り、Part2ディレクトリにあるdockerfileを`cicd_handson`リポジトリにコピーします。

```sh
cp ~/Desktop/aws_happy_code/part2/dockerfile ~/Desktop/cicdhandson/
```

### リモートリポジトリを更新する

CodeCommitのリモートリポジトリにdockerfileをpushします。

```sh
git add .
git commit -m "part2"
git push -u 
```

リモートリポジトリにブランチを追加します。

```sh
git push --set-upstream origin code_build_handson
```

### CodeBuild用 S3バケットの作成

以下のコマンドで`s3.yml`をCloudFormationで実行します。

```sh
aws cloudformation deploy --stack-name s3 --template-file ./s3.yml --tags Name=cicdhandson --profile cicd_handson
```

### ECRリポジトリの作成

`aws_happy_code`リポジトリでターミナルを開き、part2にディレクトリを変更します。

```sh
cd ~/Desktop/aws_happy_code/part2
```

以下のコマンドで`ecr.yml`をCloudFormationで実行します。

```sh
aws cloudformation deploy --stack-name ecr --template-file ./ecr.yml --tags Name=cicdhandson --profile cicd_handson
```

### ハンズオンで利用するIAM Roleを作成する

```sh
aws cloudformation deploy --stack-name codebuild-iam-role --template-file ./codebuild-role.yml --tags Name=cicdhandson --capabilities CAPABILITY_NAMED_IAM --profile cicd_handson
```

```sh
aws cloudformation deploy --stack-name event-bridge-iam-role --template-file ./event-bridge-iam-role.yml --tags Name=cicdhandson --capabilities CAPABILITY_NAMED_IAM --profile cicd_handson
```

```sh
aws cloudformation deploy --stack-name pipeline-iam-role --template-file ./pipeline-iam-role.yml --tags Name=cicdhandson --capabilities CAPABILITY_NAMED_IAM --profile cicd_handson
```

### CodeBuildのプロジェクトを作成する

```sh
aws cloudformation deploy --stack-name code-build --template-file ./code-build.yml --tags Name=cicdhandson --profile cicd_handson
```

### CodePipeline の環境構築

CloudFormationでCodePipelineを構築します。
以下のコマンドで`codecommit.yml`をCloudFormationで実行します。

```sh
aws cloudformation deploy --stack-name pipeline --template-file ./pipeline.yml --tags Name=cicdhandson --profile cicd_handson
```

### ビルドされたイメージを確認する

CloudFormationで作成されたリポジトリを確認します。

```sh
aws ecr describe-repositories --profile cicd_handson --output json 
```

コマンドを実行すると以下のようにリポジトリの詳細が出力されます。

```json
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:ap-northeast-1:{アカウント}:repository/cicdhandson",
            "registryId": "{アカウントID}",
            "repositoryName": "cicdhandson",
            "repositoryUri": "{アカウントID}.dkr.ecr.ap-northeast-1.amazonaws.com/cicdhandson",
            "createdAt": "2023-05-24T00:14:05+09:00",
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            },
            "encryptionConfiguration": {
                "encryptionType": "AES256"
            }
        }
    ]
}
```

イメージダイジェストを確認します。

```sh
aws ecr list-images --profile cicdhandson --repository-name cicdhandson --query "imageIds[*].imageDigest" --output table
```

```txt
-----------------------------------------------------------------------------
|                                ListImages                                 |
+---------------------------------------------------------------------------+
|  sha256:2326ba7ae9bff1c55a618051b86c7d71401712898afe1a833734076962a231e5  |
+---------------------------------------------------------------------------+
```

イメージのタグを確認します。

```sh
aws ecr list-images --profile cicd_handson --repository-name cicdhandson --query "imageIds[*].imageTag" --output table
```

```txt
------------
|ListImages|
+----------+
|  latest  |
+----------+
```

## 片付け

### パイプラインを削除

```sh
aws cloudformation delete-stack --stack-name pipline --profile cicd_handson
```

### CodeCommit

```sh
aws cloudformation delete-stack --stack-name codecommit --profile cicd_handson
```

## IAM

```sh
aws cloudformation delete-stack --stack-name pipeline-iam-role --profile cicd_handson
```

```sh
aws cloudformation delete-stack --stack-name code-build --profile cicd_handson
```

```sh
aws cloudformation delete-stack --stack-name code-build-iam-role --profile cicd_handson
```

### S3バケットを空にする

```sh
aws s3 ls --profile cicd_handson | grep cicdhandson | awk '{print $3}'
```

```sh
aws s3 rm s3://cicdhandson-bucket-{アカウント}/ --recursive --profile cicd_handson
```

中身を確認します。

```sh
aws s3 ls s3://cicdhandson-bucket-{アカウントID} --profile cicd_handson
```

```sh
aws s3 rb s3://cicdhandson-bucket-{アカウントID} --force
```

## まとめ

これでハンズオンは以上です。上記の構成でCodeCommit にDockerfileをおくことにより
buildspec.ymlの設定に従ってCodeBuildでイメージをビルドできます。これでイメージをリポジトリにpushしたことをトリガーに
CodeDeployによるデプロイやApp Runnerへのアプリケーションデプロイができます。
