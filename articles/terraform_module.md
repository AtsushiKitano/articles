---
title: TerraformのModuleを使ったAWSリソースの作り方
emoji: 📝
type: tech
topics: [Terraform, aws]
published: false
---

# 概要
Terraform ModuleはTerraformリソースを集約して1つの機能としたものです。Moduleを使うことでTerraformコードの複雑性を隠したり、過去に作ったシステムの構成の再利用が可能となります。

本記事では、Moduleの使い方についての説明をします。

# [Moduleとは](https://www.terraform.io/docs/language/modules/develop/index.html)
Moduleは`resource`,`variable`などのTerraformファイル(拡張子が`.tf`のファイル)で定義されたTerraformリソースを1つのディレクトリ内にまとめたものを指します。Moduleには、次の2種類があります。

- Child Module
- Root Module

Child Moduleは特定の機能をまとめたものを指します。Child Moduleは、Root Moduleや他のChild Moduleから呼ばれるように設計されています。たとえば、AWSのnetwork Moduleを考えたとき、VPC,サブネット,ネットワーク ACL,ルートテーブル,インターネットゲートウェイなどを含め、ネットワーク機能としてまとめ、Moduleとして定義することが可能です。このように機能をまとめることで、Child Moduleを呼び出す側のコードの複雑性をなくしたり、リソースを再利用しやすくすることが可能になります。
このChild Moduleはクラウドベンダが公開した公式のModuleと、個人で定義した独自のModuleがあります。公開されているModuleは、[Terraform Registry](https://registry.terraform.io/browse/modules)より利用することができます。

Root Moduleは`terraform apply`, `terraform plan`などのTerraformコマンドを実行するディレクトリにまとめられたTerraformリソースを指します。Root ModuleがChild Moduleに具体的な設定値を渡して呼び出すことでリソースの作成を実行します。

以降、タイトル以外は、Child Moduleをモジュール、Root Moduleをルートモジュールと呼びます。

# 独自のChild Module作成方法
公式のモジュールは汎用的で使いにくい場合もあるので、利用用途に合わせてモジュールを作る必要があります。ここでは独自モジュールの作成方法について説明します。

モジュールはディレクトリに次の4つのブロックをTerraformファイルに定義すること作成できます。

- resource
- data
- variable
- output

`resource`ブロックはモジュールが作成するインフラリソースの定義となります。

`data`ブロックは既存のリソースを参照する定義となります。

`variable`ブロックはルートモジュールがモジュールを呼び出す際に渡す値となります。`variable`の値を`resource`や`data`ブロックが受け取り、受け取った値を使いリソースを作成、参照します。

`output`ブロックはモジュールの返値となります。ルートモジュールがモジュールで作成したリソースを他のモジュールに渡したいときに使います。

4つのブロックは1つのTerraformファイルで定義することもできますが、管理のしやすさから別々のファイルに定義して管理します。一般的には以下のようなディレクトリ構成でモジュールを作成します。

```
module
├── README.md
├── main.tf (resource,data ブロックの定義)
├── variables.tf
└── outputs.tf
```

README.mdは処理に関係しませんが、モジュールに必要なvariableや、モジュールの用途などの説明するために作成します。そのため、必須ではありませんが、チームでモジュールを運用していくならばあるとよいでしょう。

## AWSのネットワークのChild Moduleの作り方
ここでは下記のようなAWSリソースを作るモジュールを作成する例を示します。

![vpc network module arch](https://storage.googleapis.com/zenn-user-upload/sw8c70f9wu0kfx26lqd6k7xk52hf)

このモジュールで設定できる項目は次の通りとします。

- VPC

|||

- サブネット
- ネットワークACL
- ルートテーブル
- インターネットゲートウェイ

# Root ModuleからChild Moduleの呼び出し方

## 公開Moduleの呼び出し方

## 独自Moduleの呼び出し方

# さいごに
