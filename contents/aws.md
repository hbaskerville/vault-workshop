# AWSのシークレットエンジンを試す

AWSシークレットエンジンではIAMポリシーの定義に基づいたAWSのキーを動的に発行することが可能です。AWSのキー発行のワークフローをシンプルにし、TTLなどを設定することでよりセキュアに利用できます。

サポートしているクレデンシャルタイプは下記の三つです。

* IAM user (Access Key & Secret Key)
* Assumed Role
* Federation Token

## IAMユーザの動的発行

まずシークレットエンジンをenableにします。

・macOS
```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
$ vault secrets enable aws
```
・Windows
```shell
PS > $env:VAULT_ADDR = "http://127.0.0.1:8200"
PS > vault secrets enable aws
```

次にVaultがAWSのAPIを実行するために必要なキーを登録します。

・macOS , Windows
```shell
$ vault write aws/config/root \
    access_key=************ \
    secret_key=************ \
    region=ap-northeast-1
```

`access_key`, `secret_key`, `region`はご自身の環境に合わせたものに書き換えてください。ここでは必ずしもAWSのAadminユーザを登録する必要はなく、ロールやユーザを発行できるユーザであれば大丈夫です。

次にロールを登録します。このロールがVaultから払い出されるユーザの権限と紐付きます。ロールは複数登録することが可能です。今回はまずは`credential_type`に`iam_user`を指定しています。

・macOS , Windows
```shell
$ vault write aws/roles/my-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::*"
    }
  ]
}
EOF
```

別端末を開いて`watch`コマンドでユーザのリストを監視します。

・macOS
```console
$ watch -n 1 aws iam list-users

{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```
・Windows
```shell
PS >  while ($true -eq $true) { aws iam list-users ;  sleep 1 ; clear}
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

>aws cliにログイン出来ていない場合、以下のコマンドでログインしてください。
>
>```console
>$ aws configure
>AWS Access Key ID [****************]: ****************
>AWS Secret Access Key [****************]: ****************
>Default region name [ap-northeast-1]:
>Default output format [json]:
>```

>watchが入っていない場合、以下のコマンドで監視してください。
>```shell
>while true; do aws iam list-users; echo; sleep 1;done
>```
>Windowsなどで実行できない場合は手動で実行して下さい。

ロールを使ってAWSのキーを発行してみましょう。

・macOS , Windows
```console
$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/f3e92392-7d9c-09c8-c921-575d62fe80d8
lease_duration     768h
lease_renewable    true
access_key         ************
secret_key         ************
security_token     <nil>
```

この`watch`の出力結果を見るとユーザが増えていることがわかります。`lease_id`はあとで使うのでメモしておいてください。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
		    "UserName": "vault-root-my-role-1566109640-4907",
		    "Path": "/",
		    "CreateDate": "2019-08-18T06:27:24Z",
		    "UserId": "AIDAZLVKZYEN6HOTBA74D",
		    "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566109640-4907"
        }
    ]
}
```

このユーザを使って動作を確認してみましょう。

・macOS , Windows
```console
$ aws configure

AWS Access Key ID [****************62E7]: ****************
AWS Secret Access Key [****************WF35]: ****************
Default region name [ap-northeast-1]:
Default output format [json]:
```

Vaultから払い出されたユーザのシークレットを入力して下さい。新しい端末を立ち上げて以下のコマンドを実行します。

・macOS , Windows
```console
$ aws ec2 describe-instances

An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.

$ aws s3 ls
2019-08-16 21:41:34 github-image-tkaburagi
2019-05-26 23:31:14 vault-enterprise-tkaburagi
2019-03-08 21:37:21 web-terraform-state-tykaburagi
```

Roleに設定した通りS3に対する操作のみ可能なことがわかります。

## Revokeを試す

aws cliのユーザを元のユーザに切り替えておきます。

・macOS
```console
$ aws configure

AWS Access Key ID [****************62E7]: ****************
AWS Secret Access Key [****************WF35]: ****************
Default region name [ap-northeast-1]:
Default output format [json]:

$ watch -n 1 aws iam list-users

{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
		    "UserName": "vault-root-my-role-1566109640-4907",
		    "Path": "/",
		    "CreateDate": "2019-08-18T06:27:24Z",
		    "UserId": "AIDAZLVKZYEN6HOTBA74D",
		    "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566109640-4907"
        }
    ]
}
```
・Windows
```console
PS > aws configure

AWS Access Key ID [****************62E7]: ****************
AWS Secret Access Key [****************WF35]: ****************
Default region name [ap-northeast-1]:
Default output format [json]:

PS >  while ($true -eq $true) { aws iam list-users ;  sleep 1 ; clear}

{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
		    "UserName": "vault-root-my-role-1566109640-4907",
		    "Path": "/",
		    "CreateDate": "2019-08-18T06:27:24Z",
		    "UserId": "AIDAZLVKZYEN6HOTBA74D",
		    "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566109640-4907"
        }
    ]
}
```

シンプルな手順でユーザが発行できることがわかりましたが、次はRevoke(破棄)を試してみます。Revokeにはマニュアルと自動の2通りの方法があります。

まずはマニュアルでの実行手順です。`vault read aws/creds/my-role`を実行した際に発行された`lease_id`をコピーしてください。

・macOS , Windows
```shell
$ vault lease revoke aws/creds/my-role/<LEASE_ID>
```

`watch`の実行結果を見るとユーザが削除されているでしょう。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

次に自動Revokeです。デフォルトではTTLが`765h`になっています。これは数分にしてみましょう。

・macOS , Windows
```shell
vault write aws/config/lease lease=2m lease_max=10m
```

・macOS , Windows
```console
$ vault read aws/config/lease

Key          Value
---          -----
lease        2m0s
lease_max    10m0s
```

それではこの状態でユーザを発行します。

・macOS , Windows
```console
$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/agnda2uyVWKso4E3HoWlPqY8
lease_duration     2m
lease_renewable    true
access_key         ****************
secret_key         ****************
security_token     <nil>
```

`watch`の実行結果を見るとユーザが増えています。今度は2分後にこのユーザは自動で削除されます。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        },
        {
            "UserName": "vault-root-my-role-1566111715-4258",
            "Path": "/",
            "CreateDate": "2019-08-18T07:01:56Z",
            "UserId": "AIDAZLVKZYENUIPMHTSYJ",
            "Arn": "arn:aws:iam::643529556251:user/vault-root-my-role-1566111715-4258"
        }
    ]
}
```

2分後、再度見てみるとユーザが削除されていることがわかるでしょう。

```json
{
    "Users": [
        {
            "UserName": "tykaburagi",
            "Path": "/",
            "CreateDate": "2019-06-12T07:13:45Z",
            "UserId": "****************",
            "Arn": "****************"
        }
    ]
}
```

## 参考リンク
* [AWS Secret Engine](https://www.vaultproject.io/docs/secrets/aws/index.html)
* [AWS Secret Engine API](https://www.vaultproject.io/api/secret/aws/index.html)
