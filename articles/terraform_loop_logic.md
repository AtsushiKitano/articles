---
title: Terraformのループ処理(for_each,for)について
emoji: 🗂
type: tech
topics: [Terraform, GCP]
published: false
---

# 概要
Terraformのループ処理の`for_each`,`for`について説明します。

記事で使っているコードは[このリポジトリ]()を使っています。

# はじめに
Terraformではループ処理として、次の2つの処理を提供しています。

- Terraformリソースのループ処理
- データのループ処理

Terraformリソースのループ処理は、`resource`ブロックなどを繰り返し実行します。例えば、GCP内のVPCネットワークに複数のsubnetworkを作成するとき、`google_compute_subnetwork`リソースにループ処理を適用すると、1つのリソース定義で済みます。

データのループ処理は、`map`や`list`などのCollection型をもっています。これらの値を1つずつ取り出し別のデータを作成する必要があるときがあります。Terraformでは`map`や`list`を入力し、新たな`map`,`list`を生成する機能を提供しています。

# Terraformリソースのループ処理
Terraformのリソースのループ処理としては以下の機能を提供しています。

- for_each
- count

またこの機能でループ処理できるTerraformリソースは以下の通りです。

- resource
- module
- data

## `for_each`によるリソースのループ処理
`for_each`のループ処理は、Terraform v0.12から`resource`ブロックで提供しはじめました。また、その他のブロックは順次対応していきました。現在、2021年3月時点のGAとなっているv0.14では、`resource`,`module`,`data`ブロックで機能を提供しています。そのため、古いバージョンを使っている場合は利用できないので、注意してください。

### 入力可能なデータ型
`for_each`に入力できるデータ型は、`map`と`strings`の2つのCollection型のみとなっています。`for_each`の機能は、これらCollection型のサイズ分ループ処理を実施します。

`map`型は`key = { var1 = value }`という形式のデータです。このデータの`key`は重複しないように定める必要があります。例えば、以下のような`subnetworks`というデータに、各サブネットワークの名前を`key`に`cidr`と`region`を定義しているようなデータが`map`型です。

```
locals {
  subnetworks = {
    tokyo-network = {
       cidr   = "192.168.10.0/24"
       region = "asia-northeast1"
    },
    osaka-network = {
       cidr   = "192.168.20.0/24"
       region = "asia-northeast2"
    },
  }
}
```
`map`型の出力は以下のようになります。

```
map_type = {
  "oska-network" = {
    "cidr" = "192.168.20.0/24"
    "region" = "asia-northeast2"
  }
  "tokyo-network" = {
    "cidr" = "192.168.10.0/24"
    "region" = "asia-northeast1"
  }
}
```

`strings`型は配列にprimitive型(`string`,`number`,`bool`)を格納したデータを`toset`関数で変換したデータ型です。例えば、以下のような`subnetworks`という各サブネットワークの名前を格納した配列を`toset`関数で変換したデータが`strings`型です。

```
locals {
  subnetworks = toset([
    "tokyo-network", "osaka-network"
  ])
}
```

`strings`型の出力は以下のように、`toset()`が間に挟まります。

```
strings_type = toset([
  "osaka-network",
  "tokyo-network",
])
```
### `for_each`の定義場所
`for_each`はブロック全体のループ処理と、ブロック内部のブロックのループ処理の2つの機能を提供しています。これらの機能は`for_each`の定義場所によって異なります。

定義可能な場所は下記の2箇所になります。

- ブロックの内部
- リソースブロックのループさせるブロックの直前

#### ブロックの内部への定義
ブロックの内部への定義はブロック全体をループ処理します。例えば、`google_compute_subnetwork`のリソースブロックの内部に、先のmap型の`subnetworks`を入力すると2回リソースを実行します。
また、`map`型のデータのキー値には`each.key`で、内部の値は`each.value.<変数名>`で参照します。

以下では、GCP上で`sample`のVPCネットワークを作成し、そのVPCネットワークに`tokyo-network`と`osaka-network`のサブネットワークを`for_each`を使い作成します。

```
locals {
  subnetworks = {
    tokyo-network = {
       cidr   = "192.168.10.0/24"
       region = "asia-northeast1"
    },
    osaka-network = {
       cidr   = "192.168.20.0/24"
       region = "asia-northeast2"
    },
  }
}

resource google_compute_subnetwork main {
  for_each = local.subnetworks
  
  name          = each.key
  ip_cidr_range = each.value.cidr
  region        = each.value.region
  network       = google_compute_network.main.id
}

resource google_compute_network main {
  name                    = "sample"
  auto_create_subnetworks = false
}
```

上記のように、`map`型の各値`cidr`と`region`を`each.value.cidr`,`each.value.region`と呼び出しています。`name`は`map`のキー値とするため、`each.key`として代入しています。

上記のコードの処理内容は、以下のようになります。

```
$ terraform plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_network.main will be created
  + resource "google_compute_network" "main" {
      + auto_create_subnetworks         = false
      + delete_default_routes_on_create = false
      + gateway_ipv4                    = (known after apply)
      + id                              = (known after apply)
      + mtu                             = (known after apply)
      + name                            = "sample"
      + project                         = (known after apply)
      + routing_mode                    = (known after apply)
      + self_link                       = (known after apply)
    }

  # google_compute_subnetwork.main["osaka-network"] will be created
  + resource "google_compute_subnetwork" "main" {
      + creation_timestamp         = (known after apply)
      + fingerprint                = (known after apply)
      + gateway_address            = (known after apply)
      + id                         = (known after apply)
      + ip_cidr_range              = "192.168.20.0/24"
      + name                       = "osaka-network"
      + network                    = (known after apply)
      + private_ipv6_google_access = (known after apply)
      + project                    = (known after apply)
      + region                     = "asia-northeast2"
      + secondary_ip_range         = (known after apply)
      + self_link                  = (known after apply)
    }

  # google_compute_subnetwork.main["tokyo-network"] will be created
  + resource "google_compute_subnetwork" "main" {
      + creation_timestamp         = (known after apply)
      + fingerprint                = (known after apply)
      + gateway_address            = (known after apply)
      + id                         = (known after apply)
      + ip_cidr_range              = "192.168.10.0/24"
      + name                       = "tokyo-network"
      + network                    = (known after apply)
      + private_ipv6_google_access = (known after apply)
      + project                    = (known after apply)
      + region                     = "asia-northeast1"
      + secondary_ip_range         = (known after apply)
      + self_link                  = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

```

上記から分かるように`for_each`でループを回すと、Collectionのキー値の配列としてデータが格納されます。(`google_compute_subnetwork.main["osaka-network"]`,`google_compute_subnetwork.main["tokyo-network"]`のようになります。)
また、`strings`型を入力すると、各文字列がキー値として入力されます。そのため、キーおよびデータの値はそれぞれ`each.key`,`each.value`として取得します。

#### リソースブロックのループさせるブロックの直前への定義
リソースブロック内のブロック直前にfor_eachを定義することで、そのブロックをfor_eachに入力されるCollection型のデータサイズ分繰り返し実行されます。この定義は、`resource`ブロックのみとなっています。`output`や`module`では定義できません。

定義の方法は、ループされる前に`dynamic`ブロックを定義し、中身の値を`content`ブロックを内部に定義し値を代入します。
例えば、`google_compute_firewall`のリソース内には`allow`ブロックがあります。このブロックはファイアーフォールで許可するプロトコルとポートを定義します。`icmp`と`tcp:22`, `udp:80,82`を許可するとき、以下のように`map`型でプロトコルとポートを定義します。

```
locals {
  firewall_allow_rules = [
    {
      protocol = "icmp",
      ports = []
    },
    {
      protocol = "tcp",
      ports = ["22"]
    },
    {
      protocol = "udp",
      ports = ["80", "82"]
    }
  ]
}

resource google_compute_firewall main {
  name    = "sample"
  network = google_compute_network.main.id
  
  dynamic allow {
    for_each = local.firewall_allow_rules
    iterator = _conf
    
    content {
      protocol = _conf.value.protocol
      ports    = _conf.value.ports
    }
  }
}

resource google_compute_network main {
  name                    = "sample"
  auto_create_subnetworks = false
}
```
`dynamic <block name>`としてブロック前に定義します。`<block name>`は繰り返し実行する`block`名を記述します。ここでは、`allow`ブロックを繰り返すため、`dynamic allow`と定義します。
そして、ブロック内部で代入する項目は`content`ブロックに記述します。`for_each`は`dynamic`と`content`との間で定義します。`dynamic`と`content`との間には`iterator`も定義することが可能です。
この`iterator`は`for_each`の値を参照するときの`prefix`として参照する値となります。`iterator`を定義しないとき、`<ブロック名.value.変数名>`として参照します。ここで、`iterator`を記述しないと、`allow.value.protocol`という様に値を参照させます。

上記のコードの処理内容は、以下のようになります。

```
$ terraform plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_firewall.main will be created
  + resource "google_compute_firewall" "main" {
      + creation_timestamp = (known after apply)
      + destination_ranges = (known after apply)
      + direction          = (known after apply)
      + enable_logging     = (known after apply)
      + id                 = (known after apply)
      + name               = "sample"
      + network            = (known after apply)
      + priority           = 1000
      + project            = (known after apply)
      + self_link          = (known after apply)
      + source_ranges      = (known after apply)

      + allow {
          + ports    = [
              + "22",
            ]
          + protocol = "tcp"
        }
      + allow {
          + ports    = [
              + "80",
              + "82",
            ]
          + protocol = "udp"
        }
      + allow {
          + ports    = []
          + protocol = "icmp"
        }
    }

  # google_compute_network.main will be created
  + resource "google_compute_network" "main" {
      + auto_create_subnetworks         = false
      + delete_default_routes_on_create = false
      + gateway_ipv4                    = (known after apply)
      + id                              = (known after apply)
      + mtu                             = (known after apply)
      + name                            = "sample"
      + project                         = (known after apply)
      + routing_mode                    = (known after apply)
      + self_link                       = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------
```
上記のように記述することで、1つのブロック定義で複数のブロックを生成することができます。
# データのループ処理
Terraformの型として、配列とmapの2つのCollection型があります。Terraformの処理の中で、管理しやすいように定義したデータからTerraformに沿った型に変換する必要があるときがあります。
例えば、先の`for_each`のブロック全体を繰り返し処理させたいとき、入力データの型をmap型にする必要があります。しかし、管理上配列で管理した方が管理しやすくなることもあります。例えば、subnetworksのmapを以下のようにオブジェクト型配列で定義したとします。

```
locals {
  subnetworks = [
    {
      name   = "tokyo-network",
      cidr   = "192.168.10.0/24",
      region = "asia-northeast1"
    },
    {
      name   = "osaka-network",
      cidr   = "192.168.20.0/24",
      region = "asia-northeast2"
    }
  ]
}
```
上記のようにすると、先程のmapのキーをnameとして明示的に定義することでkeyの意味が分かりやすくなります。しかし、このsubnetworksは配列型のため、for_eachへ代入することができません。そこで、このsubnetworksをmap型に変換する必要があります。こういったときに使う機能として、Terraformは`for`文を提供しています。

`for`文は配列やmapを引数にとり各要素を取り出し処理を施した後、配列またはmapを返します。定義の方法は、配列・mapの内部でforを定義し : の後ろでデータを処理し値を返します。配列を返すときは[]の内部でforを定義し、mapを返すときは{}の内部でforを定義します。

先の例で、subnetworksの配列を配列内のnameの値をkeyとするmap型を返すのは下記のようにおこないます。

```
locals {
  subnetworks = [
    {
      name   = "tokyo-network",
      cidr   = "192.168.10.0/24",
      region = "asia-northeast1"
    },
    {
      name   = "osaka-network",
      cidr   = "192.168.20.0/24",
      region = "asia-northeast2"
    }
  ]
  
  subnetworks_map = { for v in local.subnetworks : v.name => v }
}
```

この例では v.nameがキーとなり v がmapの値となります。つまり、map型のキーと値区切りは =>でおこないます。inの後ろにCollection型のデータを配置し、各要素をforとinの間の変数に代入します。:の後ろの処理で内容を実施します。

for文を使い、配列内のすべての文字列を大文字に変換する例を以下に示します。

```
locals {
  little_strings = [
    "hoge", "fuga", "piyo"
  ]
  
  large_strings = [ for v in local.little_strings : upper(v) ]
}
```

# さいごに
