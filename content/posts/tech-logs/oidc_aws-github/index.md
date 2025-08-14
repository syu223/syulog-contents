---
title: "AWSとGithub ActionsをOpenID Connectで連携させる"
date: 2025-08-01T22:39:04Z
tags:
  - AWS
  - CI/CD
draft: true
---

CI/CDを構築する際にOpenID Connectによる連携をしました。  
その際に得た知識をアウトプットします。

<!--more-->

{{< toc >}}

## OpenID Connect (OIDC) とは

参考：  
<https://docs.github.com/ja/actions/concepts/security/openid-connect>

### 利点

・対象のサービス（AWSなど）へのアクセスに、効期間が短いトークンを一時的に発行して使用する
　→**トークンは保存や管理の必要がなく、万が一漏洩しても同じものを使わない**

・どのサービス（Github Actionsなど）から、何をできるかなど、**細かく権限の制御ができる**


## 設定方法

参考：  
<https://docs.github.com/ja/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws>

### 前提

AWSへのアクセスを許可するGithubリポジトリを作成しておく必要があります。

### AWS側での設定

AWS側ではIAMにて設定が必要となります。  
大まかな内容は以下です。

①OIDCプロバイダーにGitHubを登録する

②IAMロールを作成し、信頼ポリシーを定義・ポリシーをアタッチする

#### ①OIDCプロバイダーにGitHubを登録する

IAM>ID プロバイダ>プロバイダーを追加  
　より情報を追加します。

・IDプロバイダの追加とは、「このサービスが発行するトークンなら信用してアクセスを許可してよい」とAWSに設定することです。  
　これにより、AWSへのアクセスに都度発行するトークンを利用するため、安全な認証が可能になります。

・設定項目

　プロバイダのタイプ：**OpenID Connect**  
　　→SAMLは主にSSO連携などで使用します。  
　　　今回は外部サービスからのアクセスのためOIDCを選択します。

　プロバイダ : https://token.actions.githubusercontent.com  
　　→GitHub ActionsがOIDCトークンを発行するURLです。

　対象者：**sts.amazonaws.com**  
　　→外部トークンを受け入れる窓口としてSTS（Security Token Service）を指定します。

#### ②IAMロールを作成し、信頼ポリシーを定義する

IAM>ロール>ロールを作成  
　より情報を追加します。

・「カスタム信頼ポリシー」を選択し、ステートメントには以下などを参考に構成します。  
https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub

　"**token.actions.githubusercontent.com:aud**"で、  
　　先ほどのAWS STS(sts.amazonaws.com)の指定と
 
　"**token.actions.githubusercontent.com:sub**"で、  
　　アクセスを許可する(ワークフローが配置される)Githubリポジトリの指定をします。

・「ポリシーの追加」で必要な権限のポリシーを選択します。


### Github側での設定

Github側では、最低限ワークフローを作成すればOKです。

IAM ロールを引き受けるステップでは、以下のようにaws-actions/configure-aws-credentialsを使用します。

````yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::<AWSアカウントID>:role/<ロール名>
    aws-region: ap-northeast-1
````

この際、IAMロールを直接書かず秘匿しておく方がセキュリティ上よろしいということで  
IAMロールのARNをSecrets に登録しました。

登録するには、githubのリポジトリにアクセスして  
settings → Secrets and variables → actions → New Repository Secret

「name」にワークフローで使う名前（例としてOIDC_ROLE）、  
「secret」に「arn:aws:iam::<AWSアカウントID>:role/<ロール名>」

上記のように登録すると、ワークフローでは以下のように記述できます。

````yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: ${{ secrets.OIDC_ROLE }}
    aws-region: ap-northeast-1
````
  
  
  
  

アウトプットは以上です。

