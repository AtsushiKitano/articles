---
title: Terraformのモジュール設計の3つのベストプラクティス
emoji: 🗂
type: tech
topics: [Terraform, AWS, GCP]
published: true
---

# 概要
 Terrafomが公式として紹介しているTerraform Moduleの要素のベストプラクティスとそのプラクティスを使った具体例を説明します。
 
 ここで紹介するサンプルのコードは、[このリポジトリ](https://github.com/AtsushiKitano/terraform-module-bestpractice-sample)のコードとなっています。

# Terraformモジュールのベストプラクティス
Terraformの[公式ページ](https://www.terraform.io/docs/language/modules/develop/composition.html)にはモジュール設計の以下の3つのベストプラクティスが紹介されています。

- 依存関係の反転 (Dependency Inversion)
- マルチクラウドの抽象化 (Multi-cloud Abstractions)
- dataブロックのみのモジュール (Data-only Modules)

## 依存関係の反転
モジュールの依存関係はモジュールブロック内で、`module.<module name>.<resource name>`と記述することでTerrafromが解決してくれます。例えば、以下のようにルートモジュールで`ec2`と`vpc`を定義し、`ec2`の内部で、`module.vpc.vpc_id`, `module.vpc.subnet_id`と記述したとします。すると、Terraformの処理では、`vpc`モジュールを作成、作成したリソースの`vpc`と`subnet`のIDを取得し`ec2`モジュールの`vpc`と`subnet`の変数に代入します。

```
module ec2 {
  source = "xxx"
  
  name = "sample"
  vpc = module.vpc.vpc_id
  subnet = module.vpc.subnet_id
}

module vpc {
  source = "xxx"
  
  name = "sample"
  cidr = "192.168.10.0/24"
}
```

このように依存関係を記述すると、Terraform側で依存関係を解決し、システムを構築します。
しかし、開発環境・ステージング環境・本番環境とシステムを構築する運用をしていると、開発・ステージング環境では作らず既存のシステムを共有したりすることがあります。そのようなとき、moduleの出力を直接入力するのではなく、dataブロックなどで値を代入することで依存関係の問題は解決できます。

### サンプル
ここでは、GCP上でVPCネットワークを作成し、作成したVPCネットワークにGCEを作成する例を使い、依存関係の反転を説明します。この例では開発環境と本番環境を想定し、開発環境では既存のVPCネットワークを使い、本番環境では新しくVPCネットワークを作成するとします。
開発環境と本番環境はworkspaceを用い`prd`,`dev`として切り替えるものとします。コードは以下のようになります。

```
locals {
  network = {
    prd = terraform.workspace == "prd" ? module.vpc[local.name].subnetwork_self_link[local.name] : null
    dev = terraform.workspace == "dev" ? data.google_compute_subnetwork.main[terraform.workspace].self_link : null
  }

  network_enable = {
    prd = true
    dev = false
  }

  network_conf = local.network_enable[terraform.workspace] ? [
    {
      name = local.name
      subnetworks = [
        {
          name   = local.name
          cidr   = "192.168.10.0/24"
          region = "asia-northeast1"
        }
      ]
      firewall = []
    }
  ] : []

  name                = "sample"
  default_subnet_name = "research"
}

variable "project" {
  type = string
}

data "google_compute_subnetwork" "main" {
  for_each = terraform.workspace == "dev" ? toset([terraform.workspace]) : []
  name     = local.default_subnet_name
  region   = "asia-northeast1"
}

module "vpc" {
  for_each = { for v in local.network_conf : v.name => v }
  source   = "github.com/AtsushiKitano/assets/terraform/gcp/modules/network/vpc_network"

  vpc_network = {
    name = each.value.name
  }
  subnetworks = each.value.subnetworks
  firewall    = each.value.firewall

  project = var.project
}

module "gce" {
  source = "github.com/AtsushiKitano/assets/terraform/gcp/modules/compute/gce"

  gce_instance = {
    name         = local.name
    machine_type = "f1-micro"
    zone         = "asia-northeast1-b"
    subnetwork   = local.network[terraform.workspace]
    tags         = []
  }

  boot_disk = {
    name      = local.name
    size      = 20
    interface = null
    image     = "ubuntu-os-cloud/ubuntu-2004-lts"
  }

  project = var.project
}
```

ベストプラクティスの依存関係の反転は、`gce`モジュールの`gce_instance.subnetwork`の箇所になります。ここでは、本番環境(`prd`)では`module.vpc["sample"]`の出力を代入し、開発環境(`dev`)では`data.google_compute_subnetwork.main["dev"]`を代入しています。このように直接入力せずにlocal変数を介して代入することで、環境依存で作成しないリソースが存在しないというエラーを起こさず環境の切り替えを実施できます。
SQLインスタンスなど開発では共通して使うようなリソースの管理には、依存関係の反転を使い管理することで、コストをうまく下げながら本番環境と同じコードでのリソース管理が可能になります。

## マルチクラウドの抽象化
各クラウドは類似機能を提供しています。Terraformのproviderの各コードは各クラウドプロバイダーが提供するリソースの機能をすべて実行できるよう設計されています。そのため、Terraformのルートモジュールの入力を共通させるためには抽象化レイヤーが必要となります。モジュール定義で`variable`を共通化させることで、ルートモジュールの参照先のみを代えるだけで、別のクラウドへのデプロイが可能になります。この設計は入力を共通化させることで、各ベンダ固有の機能を使えなくすることになるので注意が必要です。

### サンプル
ここでは、AWSのVPCとGCPのVPCの機能を抽象化するモジュールの例を示します。ここでは、入力として以下を許可するように設計します。

- vpc名
- vpc cidr
- subnet名
- subnet cidr
- subnet region/zone
- project

上記のvpc cidr,projectはそれぞれAWS,GCPで必要となる入力となるので、他方で不要ではありますが定義しています。それぞれのコードは以下のようになります。

- AWSのvpc モジュールのコード

```
locals {
  subnet_ids = [
    for v in var.subnets : aws_subnet.main[v.name].id
  ]
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc.cidr
  tags = {
    Name = var.vpc.name
  }
}

resource "aws_subnet" "main" {
  for_each = { for v in var.subnets : v.name => v }

  vpc_id            = aws_vpc.main.id
  availability_zone = each.value.location
  cidr_block        = each.value.cidr
  tags = {
    Name = each.value.name
  }
}

variable "vpc" {
  type = object({
    name = string
    cidr = string
  })
}

variable "subnets" {
  type = list(object({
    cidr     = string
    name     = string
    location = string
  }))
}

variable "project" {
  type = string
}
```

- GCPのvpc モジュールのコード

```
resource "google_compute_network" "main" {
  name                    = var.vpc.name
  auto_create_subnetworks = false

  project                         = var.project
  delete_default_routes_on_create = false
}

resource "google_compute_subnetwork" "main" {
  for_each = { for v in var.subnets : v.name => v }
  provider = google-beta

  name          = each.value.name
  ip_cidr_range = each.value.cidr
  network       = google_compute_network.main.self_link
  region        = each.value.location

  project = var.project
}

variable "vpc" {
  type = object({
    name = string
    cidr = string
  })
}

variable "subnets" {
  type = list(object({
    name     = string
    cidr     = string
    location = string
  }))
}

variable "project" {
  type = string
}
```

- 各コードを呼び出すルートモジュールのコード

```
locals {
  location = {
    aws = "us-east-2c"
    gcp = "asia-northeast1"
  }
}

variable "project" {
  type = string
}

module "vpc" {
  # source = "../../modules/multi-cloud_abstractions/aws_vpc" # AWSのときはこちらを有効化
  # source = "../../modules/multi-cloud_abstractions/gcp_vpc" # GCPのときはこちらを有効化

  vpc = {
    name = "sample"
    cidr = "192.168.10.0/24"
  }

  subnets = [
    {
      name     = "sample"
      cidr     = "192.168.10.0/28"
      location = local.location[terraform.workspace]
    }
  ]

  project = var.project
}
```

awsのvpcモジュールとgcpのvpcモジュールのvariableを比べると両方とも同じ定義になっているため、ルートモジュールの呼び出す値の定義を共通化させることができます。このようにすることで、ルートモジュールの`source`の値を代えるだけで、別のクラウドへのデプロイが可能になります。

この設計は先にも述べたとおり、特定の機能を使えなくしてコードを共通化させています。そのため、制限が発生するので、利用には注意が必要です。しかし、マルチクラウドが浸透しつつある世の中ではうまく活用することで、管理が簡便になると思われます。
また、マルチクラウドの抽象化と、クラウド機能を抽象化する紹介でしたが`resouce`ブロックの項目も抽象化が可能です。例えば、`google_compute_firewall`では、アクセスを拒否するか許可するかでブロックを定義します。この拒否・許可をうまく抽象化することで入力を簡便化することが可能です。

## dataブロックのみのモジュール
モジュールはTerraformコードを集約します。そこで、複雑な参照ロジックが必要となるdataブロックのみをまとめて管理することで、複雑性を隠蔽しリソースへの参照を簡便にすることができます。

### サンプル
ここではGCPのKMSの復号化処理をまとめるdataブロックの例をしめします。
GCPのKMSを使い復号化するには、[google_kms_secret](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/kms_secret)のデータブロックを使います。`google_kms_secret`のデータブロックは、KMSの鍵(`google_kms_crypto_key`)のself_linkを指定する必要があります。また、KMSの鍵のself_linkは、[google_kms_crypto_key](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/kms_crypto_key)を使い参照します。また、`google_kms_crypto_key`の参照には、[google_kms_key_ring](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/kms_key_ring)が必要になります。以上より、KMSで暗号化したデータを復号化するには、以下の3つのdataブロックが必要になります。

- google_kms_key_ring
- google_kms_crypto_key
- google_kms_secret

ここでは、3つのdataブロックを合わせた`secret`モジュールを作成します。`key_ring`、`key`と暗号化した文字列をモジュールに入力し、復号化した文字列を出力します。文字列の暗号化は以下のコマンドで実行します。

```
echo -n $PLAINTEXT | gcloud kms encrypt --project $(gcloud config get-value core/project) --keyring $KEYRING --key $KEY --location $LOCATION  --plaintext-file - --ciphertext-file - | base64
```

- PLAINTEXT : 暗号化する文字列
- KEYRING : key ring名
- KEY : key名
- LOCATION : key ringのロケーション

モジュールのコードは次のようになります。

```
data "google_kms_key_ring" "main" {
  name     = var.kms_info.key_ring
  location = var.location
}

data "google_kms_crypto_key" "main" {
  name     = var.kms_info.crypto_key
  key_ring = data.google_kms_key_ring.main.self_link
}

data "google_kms_secret" "main" {
  crypto_key = data.google_kms_crypto_key.main.self_link
  ciphertext = var.ciphertext
}

variable "kms_info" {
  type = object({
    key_ring   = string
    crypto_key = string
  })
}

variable "ciphertext" {
  type = string
}

variable "location" {
  type    = string
  default = "global"
}

output "plaintext" {
  value = data.google_kms_secret.main.plaintext
}
```

モジュールを呼び出すルートモジュールのコードは以下のようになります。

```
module "secret" {
  source = "../../modules/secret"

  kms_info = {
    key_ring   = "test"
    crypto_key = "bestpractice-sample"
  }

  ciphertext = "CiQA3KEfa06yazAYoXGyuX0ZRX4MjluESCBTQPWhEgRzeK4HB3ASLQCk29aHH/XgLZDZTGAVOGyQpveN33SWVlGTY6qMqIiFATkCGYOZpSgbHv5Y7A=="
}

output "secret" {
  value = module.secret.plaintext
}
```

`ciphertext`の値は、`key_ring`,`bestpractice-sample`で `test` という文字列を暗号化した値になります。ルートモジュールを実行した結果は次のようになります。

```
$ terraform apply

Already have image: alpine/terragrunt
time=2021-03-28T11:12:13Z level=info msg=Stack at /workspace/src/secret:
  => Module /workspace/src/secret (excluded: false, dependencies: [])

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

secret = "test"
```

このようにdataブロックをまとめることで、複雑な処理も隠蔽することで簡便に利用できるようになります。

# さいごに
Terraformの公式ページで紹介されているモジュール設計のベストプラクティスを3つ紹介しました。依存関係の反転やdataブロックのみのモジュールは非常に使える設計と思います。マルチクラウドの抽象化は機能が制限されることもあり慎重に利用する必要ですが、`resource`内部での抽象化は可能なので抽象化はうまく活用できる場合は利用してみてください。
今後、モジュールを使いシステムを構築するにあたりこれらプラクティスも導入すると、ルートモジュールのコードが簡便になり管理しやすくなると思います。
