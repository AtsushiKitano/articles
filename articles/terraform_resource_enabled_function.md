---
title: Terraformリソースのon/off機能の実装方法
emoji: 📝
type: tech
topics: [Terraform, gcp,]
published: true
---

# 概要
Terraformのリソースのon/off機能を`for_each`で実現する方法について説明します。

`for_each`と`if`文でTerraformブロックをon/offする機能の実装は、以下の通りです。
`enabled`のbool値でリソースの作成を制御します。
```
locals {
  enabled = true
  
  vpc_name = local.enabled ? ["sample"] : []
}

resource google_compute_network main {
  for_each = toset(local.vpc_name)
  
  name = each.value
  auto_create_subnetworks = false
}
```

# 実装方法
`for_each`に空の値を入力すると、処理を実行しません。たとえば、以下のように`google_compute_network`のブロックに空の配列を入力したとき、`google_compute_network.main`は実行されません。

```
resource google_compute_network main {
  for_each = toset([])
  
  name = each.value
  auto_create_subnetworks = false
}
```

このコードに対してterraformコマンドを実行すると、以下のようになります。

```
$ terraform plan

No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```

ここで、配列に値を入力し以下のようし、terraformコマンドを実行すると以下のようになります。

- 配列に値を入力したコード

```
resource "google_compute_network" "main" {
  for_each = toset(["sample"])

  name                    = each.value
  auto_create_subnetworks = false
}
```

- コマンド実行したときの出力

```
$ terraform plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_network.main["sample"] will be created
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

Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

このように、空配列またはmapを入力したあり、値を入力したりすることでon/offの切り替えが可能になります。この値の入力するしないの切り替えに`if`文を使うことで、実現することができます。

先の例をif文を使い、on/offを切り替えたコードは以下のようになります。

```
locals {
  enabled = true
  
  vpc_name = local.enabled ? ["sample"] : []
}

resource google_compute_network main {
  for_each = toset(local.vpc_name)
  
  name = each.value
  auto_create_subnetworks = false
}
```

上記では、`enabled = true`となったとき、`vpc_name = ["sample"]`となり、`google_compute_network`が実行され、`enabled = false`としたとき、`vpc_name = []`となり、処理が実行されません。

# さいごに
`for_each`を使いリソースのon/off機能の実装方法を紹介しました。今回の例では、`resouce`ブロックについての説明をしましたが、`for_each`が対応してるその他の`module`と`data`でも実装可能です。
このon/off機能は本番環境で作成し、開発環境で作成しないようなリソースの管理に有効です。実装方法も簡便なので活用してみてください。
