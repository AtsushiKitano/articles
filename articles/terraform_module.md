---
title: TerraformのModuleを使ったAWSリソースの作り方
emoji: 🗂
type: tech
topics: [Terraform, AWS]
published: true
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

このモジュールでは以下のリソースをまとめて設定できます。

- VPC
- サブネット
- ネットワークACL
- ルートテーブル
- インターネットゲートウェイ


- main.tfの内容

```
locals {
  subnet_ids = [
    for v in var.subnets : aws_subnet.main[v.name].id
  ]
}

resource "aws_vpc" "main" {
  cidr_block = var.cidr
  tags = {
    Name = var.vpc_name
  }
}

resource "aws_subnet" "main" {
  for_each = { for v in var.subnets : v.name => v }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags = {
    Name = each.value.name
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = var.ig_name
  }
}

resource "aws_route_table" "main" {
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = toset(var.route_internet)
    iterator = _conf

    content {
      cidr_block = _conf.value
      gateway_id = aws_internet_gateway.main.id
    }
  }

  tags = {
    Name = var.route_table_name
  }
}

resource "aws_network_acl" "main" {
  depends_on = [
    aws_subnet.main
  ]
  vpc_id     = aws_vpc.main.id
  subnet_ids = local.subnet_ids

  tags = {
    Name = var.acl_name
  }
}

resource "aws_network_acl_rule" "main" {
  for_each = { for v in var.acl_rules : join("-", [v.rule_action, v.protocol, v.priority, v.from_port, v.to_port]) => v }

  network_acl_id = aws_network_acl.main.id
  rule_number    = each.value.priority
  egress         = each.value.egress
  protocol       = each.value.protocol
  rule_action    = each.value.rule_action
  cidr_block     = each.value.cidr
  from_port      = each.value.from_port
  to_port        = each.value.to_port
}

resource "aws_default_network_acl" "main" {
  default_network_acl_id = aws_vpc.main.default_network_acl_id

  ingress {
    protocol   = -1
    rule_no    = 100
    action     = "allow"
    cidr_block = aws_vpc.main.cidr_block
    from_port  = 0
    to_port    = 0
  }

  egress {
    protocol   = -1
    rule_no    = 32766
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = var.defalut_acl_name
  }
}

resource "aws_default_route_table" "main" {
  default_route_table_id = aws_vpc.main.default_route_table_id

  tags = {
    Name = var.default_route_table_name
  }
}

resource "aws_security_group" "main" {
  name   = var.sg.name
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.sg.ingress_rules
    iterator = _conf

    content {
      from_port   = _conf.value.from
      to_port     = _conf.value.to
      protocol    = _conf.value.protocol
      cidr_blocks = _conf.value.cidrs
    }
  }

  tags = {
    Name = var.sg.name
  }
}
```
- variable.tfの内容

```
variable "vpc_name" {
  type = string
}

variable "cidr" {
  type = string
}

variable "subnets" {
  type = list(object({
    cidr = string
    az   = string
    name = string
  }))
}

variable "ig_name" {
  type = string
}

variable "route_table_name" {
  type = string
}

variable "default_route_table_name" {
  type = string
}

variable "route_internet" {
  type = list(string)
}

variable "acl_name" {
  type = string
}

variable "defalut_acl_name" {
  type = string
}

variable "acl_rules" {
  type = list(object({
    priority    = number
    egress      = bool
    protocol    = string
    cidr        = string
    rule_action = string
    from_port   = number
    to_port     = number
  }))
}

variable "sg" {
  type = object({
    name = string
    ingress_rules = list(object({
      from     = number
      to       = number
      protocol = string
      cidrs    = list(string)
    }))
  })
}
```

- output.tfの内容

```
output "subnet_id" {
  value = { for v in var.subnets : v.name => aws_subnet.main[v.name].id }
}

output "sg_id" {
  value = aws_security_group.main.id
}
```

# Root ModuleからChild Moduleの呼び出し方
ルートモジュールは`module`ブロックを使い、公開モジュールや独自モジュールを呼び出します。以下の6つの項目を`module`ブロックで使いモジュールを呼び出します。

| 項目        | 機能                             | 必須 |
| --          | --                               | --   |
| source      | モジュールの格納先               | ○    |
| version     | モジュールのバージョン           | ×    |
| count       | `module`ブロックの反復           | ×    |
| for_each    | `module`ブロックの反復           | ×    |
| proviers    | モジュールで使うprovider         | ×    |
| depeonds_on | `module`ブロックの実行の依存関係 | ×    |

 `source`はモジュールが格納されているパスを定義します。相対パスや他のホストなどを指定することができます。このとき指定するパスはモジュールを定義したディレクトリのパスとなります。他のホストを指定する場合は格納先に応じて、呼び出しかたが異なります。`source`は`module`ブロックに必須の項目となります。

 `version`はモジュールのバージョンを示します。最新バージョンであったり、特定のバージョンを指定しモジュールのコードを固定させます。`version`はモジュールの格納先が`Terraform Registry`であるときに使えます。その他の格納先である場合には、この`version`は機能しません。そのときは、格納先の機能に応じて、バージョンを指定します。`source`は`module`ブロックで必ず設定する必要はありません。

 `count`、`for_each`は`module`ブロックを繰り返し実行を命令します。`resource`ブロックと同じように配列やmapを渡し実行すると、その個数分`module`ブロックが実行されます。`source`は`module`ブロックで必ず設定する必要はありません。

 `providers`はモジュールで利用する`provider`を渡します。`provider`はルートモジュールで定義したproviderを使いますが、特定のモジュールでルートモジュールで定義した`provider`以外のものが必要になったとき、当該項目に定義しモジュールに渡します。`source`は`module`ブロックで必ず設定する必要はありません。

 `depends_on`は`module`ブロックの実行順序の依存関係を定義します。モジュールAとモジュールBがあったとき、モジュールBがモジュールAよりも後に実行する必要があるとします。このときモジュールBに`depends_on`を定義し、モジュールAを記載します。`source`は`module`ブロックで必ず設定する必要はありません。


`module`ブロックは上記の項目を使い、定義していきます。定義は以下のようにおこないます。

```
module ec2 {
  source = "../modules/ec2"
  
  name   = "sample"
  subnet = module.vpc.subnet_id
  ...
}
```

この例はec2モジュールを呼び出す例となっています。`module`ブロックの定義は、`module`を宣言した後に`module`オブジェクトの名前を記載します。ここでは、`ec2`としています。
この値を使い、他の`module`ブロックが`module`オブジェクトを参照します。参照方法はsubnetで定義しているように`module.vpc.subnet_id`のように記載します。ここで、`subnet_id`は、モジュール内で定義した`output`名となります

`module`ブロック内の`name`,`subnet`などの値は、モジュール内で定義した`variable`となります。`module`内で定義したデフォルト値のない`variable`はすべて定義しなければなりません。name,subnetなどの値は例として記載しています。

## sourceの種類
sourceはTerraformモジュールの保存先の指定となります。呼び出し方は保存先により異なります。ここではその代表的なsourceの保存先と呼び出し方を紹介します。すべての内容は、[公式ページ](https://www.terraform.io/docs/language/modules/sources.html)'を確認ください。

| 種類               | 指定方法                           |
| --                 | --                                 |
| Local Path         | 相対/絶対パス                      |
| Terraform Registry | <NAMESPACE>/<NAME>/<PROVIDER>  |
| GitHub             | github.com/<Repository Path> |
| S3                 | s3::https://<S3 Path>              |
| GCS                | gcs::https://www.googleapis.com/storage/v1/<BUCKET_NAME>/PATH |

`Local Path`の相対/絶対パスはモジュールを定義したディレクトリのパスを記述します。例えば、下記のようなディレクトリ構成であれば、`source`は次のように記述します。

- Local Pathのモジュール定義の例

```
module ec2 {
  source = "../modules/ec2"
  
  ...
}
```

- ディレクトリ構成の例

```
.
├── modules
│   ├── ec2
│   │   ├── main.tf
│   │   └── variable.tf
│   └── vpc
│       ├── main.tf
│       ├── output.tf
│       └── variable.tf
└── src
    ├── main.tf
    ├── provider.tf
    └── terraform.tf
```

`Terraform Registry`は[公式のregistryページ](https://registry.terraform.io/browse/modules)から目的のモジュールを探し、`<NAMESPACE>/<NAME>/<PROVIDER>`の順で定義します。例えば、[awsのVPCモジュール](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)を使う場合は次の様に記述します。

```
module vpc {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "sample"
  ...
}
```

定義すべき設定項目は、モジュールの[Inputページ](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest?tab=inputs)を確認し定義します。

`GitHub`はGitHub上に登録したモジュールを参照します。参照方法は`github.com/<Repository Path>/` となります。ここで、著者が作った [VPC Networkのモジュール](https://github.com/AtsushiKitano/assets/tree/master/terraform/aws/modules/network/vpc)の参照方法は下記の通りです。GitHubでの実行ではリポジトリへの参照権限が必要となります。

```
module vpc {
  source = "github.com/AtsushiKitano/assets/terraform/aws/modules/network/vpc"
  
  config = {
     vpc = {
       name = "sample"
       cidr = "192.168.10.0/24"
     }
    subnets = [
      {
        name = "sample"
        cidr = "192.168.10.0/24"
        az   = "us-east-2a"
      }
    ]
  }
}
```

`S3`はAWSのS3バケット上に保存したモジュールを参照します。GitHubの参照は先に記述した通りで、参照権限が必要となります。そのため、案件固有のモジュールをGitHubで管理している場合、クラウド上に構築したCICDのパイプラインから参照できないことがあります。例えば、AWS上での構築パイプラインを構築した場合、パイプラインのエージェントにS3への権限を付与し、モジュールを参照させます。
参照方法は、`s3::https://<S3 Path>`となります。ここでは、us-east-2に作成した`terraform-module-sample`バケットを参照させています。

```
module "sample" {
  source = "s3::https://s3-us-west-1.amazonaws.com/terraform-module-sample/network/vpc/"

  config = {
    vpc = {
      name = "sample"
      cidr = "192.168.0.0/16"
    }

    subnets = [
      {
        name = "sample"
        cidr = "192.168.10.0/24"
        az   = "us-east-2a"
      }
    ]
  }
}
```
`terraform-module-sample`バケットの構成は次のようになっています。

```
aws s3 ls terraform-module-sample/network/vpc/ --region us-west-1
2021-03-27 14:22:02          0
2021-03-27 14:22:20        598 main.tf
2021-03-27 14:22:20        676 output.tf
2021-03-27 14:22:21        619 variable.tf
```

`gcs`も同様にGCP上でCICDのパイプラインを構築し、エージェントがGCP上のリソースであるとき、GitHubへアクセスができないため利用します。
参照は `gcs::https://www.googleapis.com/storage/v1/<BUCKET_NAME>/PATH` です。ここで、`terraform-module-sample`に登録したモジュールを参照する例を以下に示します。

```
module "sample" {
  source = "gcs::https://www.googleapis.com/storage/v1/terraform-module-sample/network/vpc/"

  config = {
    vpc = {
      name = "sample"
      cidr = "192.168.0.0/16"
    }

    subnets = [
      {
        name = "sample"
        cidr = "192.168.10.0/24"
        az   = "us-east-2a"
      }
    ]
  }
}
```

`terraform-module-sample`バケット内の構成は、以下の通りです。

```
$ gsutil ls gs://terraform-module-sample/network/vpc

gs://terraform-module-sample/network/vpc/main.tf
gs://terraform-module-sample/network/vpc/output.tf
gs://terraform-module-sample/network/vpc/variable.tf
```

## Child ModuleのAWSのVPCモジュールの呼び出し例
モジュールのときに紹介したAWSのVPCモジュールの呼び出し方を紹介します。今回はモジュールが下記ディレクトリ構成のようにローカルディレクトリに保存されているとします。

```
.
├── README.md
├── modules
│   └── vpc
│       ├── main.tf
│       ├── output.tf
│       └── variable.tf
└── src
    ├── main.tf
    └── provider.tf
```

ルートモジュールのVPCの呼び出しは以下のようになります。

```
locals {
  name = "sample"
  az   = "us-east-2a"
}

module "vpc" {
  source = "../modules/vpc"

  vpc_name                 = local.name
  ig_name                  = local.name
  acl_name                 = local.name
  route_table_name         = local.name
  defalut_acl_name         = "default-${local.name}"
  default_route_table_name = "default-${local.name}"

  cidr = "10.10.0.0/24"

  subnets = [
    {
      name = local.name
      cidr = "10.10.0.0/28"
      az   = local.az
    }
  ]

  route_internet = [
    "0.0.0.0/0"
  ]

  acl_rules = [
    {
      priority    = 200
      egress      = false
      protocol    = "tcp"
      rule_action = "allow"
      cidr        = "0.0.0.0/0"
      from_port   = 22
      to_port     = 22
    }
  ]

  sg = {
    name = local.name
    ingress_rules = [
      {
        from     = 22
        to       = 22
        protocol = "tcp"
        cidrs    = ["0.0.0.0/0"]
      }
    ]
  }
}
```

# モジュールを使ったシステムの構築
独自モジュールを使い、AWS上にEC2を作成する例を示します。
システム構成図は以下の通りです。

![system arch](https://storage.googleapis.com/zenn-user-upload/g8ki5q7slref9wl81ohwfk6kwoky)

ディレクトリ構成は以下の通りです。下記コードは[当該リポジトリ](https://github.com/AtsushiKitano/terraform-module-sample)に格納されています。

```
.
├── README.md
├── modules : モジュール
│   ├── ec2
│   │   ├── main.tf
│   │   └── variable.tf
│   └── vpc
│       ├── main.tf
│       ├── output.tf
│       └── variable.tf
└── src : ルートモジュール
    ├── main.tf
    └── provider.tf
 ```

`modules`がモジュールを格納しているディレクトリとなっています。`ec2`,`vpc`がそれぞれのモジュールを定義しています。

`src`はルートモジュールとなっています。ルートモジュールで下記terraformコマンドを実行すると、構成図のシステムが構築されます。

```
terraform init
terraform plan
terraform apply
```

下記コマンドで構築したシステムを削除することができます。

```
terraform destroy
```

# さいごに
Terraform Moduleについて、使い方について例を使いながら説明しました。モジュールを使うことでまとまった処理をまとめることができるため、コードの再利用とコードの複雑性を解消ができます。モジュールを活用することで素早くかつ簡便なリソース管理が可能になります。是非活用を検討してみてください。
