---
title: TerratestによるTerraformコードの単体テスト
emoji: 📝
type: tech
topics: [Terraform, gcp, CICD]
published: true
---

# 概要
Terratestを使ったTerraformで構築したインフラのテストと方法について。

# [Terratestとは](https://terratest.gruntwork.io/docs/getting-started/introduction/#watch-how-to-test-infrastructure-code)
TerratestはインフラをテストするGoライブラリです。
提供している機能としては、以下の機能があります。

- Terraformコードのテスト
- Packerテンプレートのテスト
- Dockerイメージのテスト
- Helm Chartのテスト
- SSH経由のホストへのコマンド実行
- AWS,Azure,GCP,k8sのAPI
- HTTPリクエストの作成
- shellコマンドの実行

Goのライブラリであり、HTTPリクエストを使いテストの記述もできるため、インフラの設定値のみだけではなく、自動化できる機能テスト、結合テストも一括して書くことが可能となります。
また、テストの記法はGoのテストの記法をそのまま利用するため、Goを書きなれていれば新たに覚えることはありません。

# 使い方
## 環境の要件
TerratestはGoライブラリであり、実行環境は1.13以上のバージョンである必要があります。

Terratestのライブラリは、以下のように取得します。
以下では、terratestのライブラリの取得方法を示します。

- terratestライブラリ
```
go get github.com/gruntwork-io/terratest
```

また、これらライブラリに依存するするライブラリを取得する必要があります。
実行環境で不足する場合は、取得してください。

## テストの書き方
Terratestの利用はテストで利用するモジュールをインポートして利用します。
また、Terraformのテスト方法はtfstateのoutputブロックの値をテストします。  

Terratestのterraformライブラリの使い方は、ライブラリがterraformを呼び出す設定を有するOptions typeにテスト対象のterraformコードが格納されている[パスなど](https://pkg.go.dev/github.com/gruntwork-io/terratest/modules/terraform#Options)を渡し、ライブラリのOutputやApplyなどの各種メソッドを実行します。そうすると、ライブラリがOptionsの値を元にterraformコマンドを実行します。
Optionsオブジェクトは、Terraformコードが存在するパスを設定する `TerraformDir` が必須となります。この値は、testコードからの相対パスでを代入します。

例えば、下記ディレクトリの構成としたとき、 `TerraformDir` に `../test` を渡して `terraform.Options`オブジェクトを生成します。
```
.
├── README.md
├── src # terraformコードのディレクトリ
│   ├── main.tf
│   ├── output.tf
│   ├── provider.tf
│   └── terraform.tf
└── test  # terratestコードのディレクトリ
    └── network_test.go
```

## HelloWorldのテスト
ここでは、Hello Worldの文字列を出力するTerraformをテストする方法について説明します。
この説明のディレクトリ構成は、以下のとおりです。
```
helloworld
├── helloworld_test.go  Hello WorldをテストするTerratestコード
└── main.tf             Hello Worldを出力するTerraformコード
```
Hello Worldを出力するTerraformコードは、以下の通りです。
```
output "helloworld" {
  value = "Hello World"
}
```

outputブロックの参照は、`terraform.Output(t, terraformOption, <block name>)`で参照します。
今回、TerratestとTerraformのコードは同一ディレクトリにあるので、`TerraformDir` に `.` を渡します。

`block name` は、outputブロック名を記述します。上記のHello Worldのテストサンプルでは、`helloworld`となります。
また、テストの実施はgoの `testing` ライブラリなどユニットテストで使われるライブラリを使いテストをおこないます。以上の内容から`Hello World`のテストコードは以下のようになります。

```
package helloworld

import(
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestHelloWorld(t *testing.T) {
    t.Parallel()

    terraformOption := &terraform.Options{
        TerraformDir: ".",
    }

    actual := terraform.Output(t, terraformOption, "helloworld")
    expected := "Hello World"

    assert.Equal(t, expected, actual)
}
```

## テスト実行の方法
テストの実行方法は、下記の通りです。

1. Terraformコマンドで、`terraform apply`の実行
1. go コマンドでTerratestのコードの実行

上記の内容のコマンドは以下の通りです。

```
terraform apply
go test -v
```

## Webシステムのテスト
次に、下記のような構成のシステムについてテストをします。

![system_arch](https://storage.googleapis.com/zenn-user-upload/g8hbgck0z52xngmoevo1n1fysgl6)

このシステムは、GCP上にVPCネットワークを作成し、GCEを作成し、当該システムにNginxをインストールしたWebシステムです。
テストでは、GCPの設定とWebシステムへの通信を確認するテストを実施します。
GCPの設定項目およびWebシステムへの通信要件は、以下の通りで以下の内容をテストします。

- VPC ネットワークの設定

| vpc name | subnetwork name | subnetwork region | subnetwork cidr |
| ---      | ---             | ---               | ---              |
| sample   | sample          | asia-northeast1   | 192.168.10.0/24  |

- ファイアーフォールの設定

| name           | direction | target                    | source address | priority | rule type | protocol | port      |
| ---            | ---       | ---                       | ---            | ---      | ---       | ---      | ---       |
| ingress-sample | INGRESS   | VPC network上のすべてのVM | 0.0.0.0/0      | 1000     | allow     | tcp      | 80 |

- GCEの設定

| name   | machine type | zone              | subnetwork | tags | boot disk size | mashine image                   |
| ---    | ---          | ---               | ---        | ---  | ---            | ---                             |
| sample | f1-micro     | asia-northeast1-b | sample     | -    | 20             | ubuntu-os-cloud/ubuntu-2004-lts |

- ネットワーク疎通の設定

| Web IP address  | access port | HTTP Request | Response Status Code |
| ---             | ---        | ---          | ---                  |
| GCEのIPアドレス | 80         | GET          | 200                  |

これらのシステムの構築するTerraformコードおよびTerratestのコードは、この[リポジトリ](https://github.com/AtsushiKitano/terratest_sample)を参照してください。

リポジトリのディレクトリ構成は、以下の通りです。

```
.
├── README.md
├── helloworld # 先のhello worldサンプル
│   ├── helloworld_test.go
│   └── main.tf
├── src #システム構築用のTerraformコード
│   ├── main.tf
│   ├── output.tf
│   ├── provider.tf
│   ├── terraform.tf
│   └── terragrunt.hcl
└── test #システムをテストするTerratest コード
    ├── functional_test.go  #nginxの疎通テストのコード
    ├── gce_test.go  #GCE設定テストのコード
    ├── network_test.go # VPC,Subnetwork,Firewallの設定テストのコード
    └── terraform_options.go # terraform.Optionsを定義したコード
```

ここで利用しているTerraformモジュールは、 [こちらのモジュール](https://github.com/AtsushiKitano/assets/tree/master/terraform/gcp/modules) を参照しています。

Terraformで構築したVPC、Subnetwork、Firewallのテスト(network_test.go)は下記の通りです。
```
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestNetwork(t *testing.T) {
	t.Parallel()

	actualNetworkName := terraform.Output(t, terraformOptions, "network_name")
	assert.Equal(t, "sample", actualNetworkName)

	actualSubnetName := terraform.Output(t, terraformOptions, "subnetwork_name")
	assert.Equal(t, "sample", actualSubnetName)

	actualSubnetRegion := terraform.Output(t, terraformOptions, "subnetwork_region")
	assert.Equal(t, "asia-northeast1", actualSubnetRegion)

	actualSubnetCidr := terraform.Output(t, terraformOptions, "subnetwork_cidr")
	assert.Equal(t, "192.168.10.0/24", actualSubnetCidr)
}

func TestFirewall(t *testing.T) {
	t.Parallel()

	actualFirewallName := terraform.Output(t, terraformOptions, "firewall_name")
	assert.Equal(t, "ingress-sample", actualFirewallName)

	actualFirewallDirection := terraform.Output(t, terraformOptions, "firewall_direction")
	assert.Equal(t, "INGRESS", actualFirewallDirection)

	actualFirewallPriority := terraform.Output(t, terraformOptions, "firewall_priority")
	assert.Equal(t, "1000", actualFirewallPriority)

	actualFirewallProtocol := terraform.Output(t, terraformOptions, "firewall_allow_rules_protocol")
	assert.Equal(t, "tcp", actualFirewallProtocol)

	expectedSrcRanges := []string{"0.0.0.0/0"}
	actualFirewallSrcRanges := terraform.OutputList(t, terraformOptions, "firewall_src_ranges")
	assert.Equal(t, expectedSrcRanges, actualFirewallSrcRanges)

	expectedFirewallPorts := []string{"80"}
	actualFirewallPorts := terraform.OutputList(t, terraformOptions, "firewall_allow_rules_ports")
	assert.Equal(t, expectedFirewallPorts, actualFirewallPorts)
}

```
 上記の注意点としては、outputブロックの値がlistとなっている場合、list型を変数に入力するには `terraform.OutputList`を使う必要があります。その他の型を変数に入力する必要がある場合は、[公式ページ](https://pkg.go.dev/github.com/gruntwork-io/terratest/modules/terraform#Output)を参照してください。


次にGCEのテストコード(gce_test.go)は、次のようになります。

```
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestGCE(t *testing.T) {
	t.Parallel()

	actualGCEName := terraform.Output(t, terraformOptions, "gce_name")
	assert.Equal(t, "sample", actualGCEName)

	actualGCEMachineType := terraform.Output(t, terraformOptions, "gce_machine_type")
	assert.Equal(t, "f1-micro", actualGCEMachineType)

	actualGCEZone := terraform.Output(t, terraformOptions, "gce_zone")
	assert.Equal(t, "asia-northeast1-b", actualGCEZone)

	actualGCEDiskSize := terraform.Output(t, terraformOptions, "gce_boot_disk_size")
	assert.Equal(t, "20", actualGCEDiskSize)

	actualGCEDiskImage := terraform.Output(t, terraformOptions, "gce_boot_disk_image")
	assert.Contains(t, "ubuntu-os-cloud/ubuntu-2004-lts", actualGCEDiskImage)
}

```

最後にNginxとの疎通テスト(functional_test.go)をするコードは、以下のようになります。

```
package test

import (
	"fmt"
	"testing"

	"github.com/gruntwork-io/terratest/modules/http-helper"
	"github.com/gruntwork-io/terratest/modules/terraform"
)

func TestConnection(t *testing.T) {
	t.Parallel()

	ipAddressList := terraform.OutputList(t, terraformOptions, "gce_global_ip")
	url := fmt.Sprintf("http://%s:80", ipAddressList[0])

	statusCode, _ := http_helper.HTTPDo(t, "GET", url, nil, nil, nil)
	if expectedCode := 200; statusCode != expectedCode {
		t.Errorf("handler returned wrong status code: got %v want %v", statusCode, expectedCode)
	}
}
```
疎通確認のテストは、`http-helper`ライブラリを使います。このライブラリの`HTTPDo`メソッドはHTTPリクエストを生成し、対象のホストに対してHTTPリクエストを実行しステータスコードとbodyの値を返します。
[リクエスト実行の引数](https://pkg.go.dev/github.com/gruntwork-io/terratest/modules/http-helper#HTTPDo)は、リクエストメソッド、ホスト、body情報、ヘッダ、tls設定を渡します。

疎通確認テストの注意点はTerraformのNginx構築にstartupスクリプトでnginxをインストールするようにしているため、疎通テストは時間をおく必要があります。よって、構築後すぐにテストすると、Nginxの疎通テストは失敗します。

# さいごに
Terratestを使ったTerraformの設定内容のテスト方法をNginxのシステムをテストする例を使い説明しました。TerratestはGoのライブラリでありかつ、HTTPリクエストを実行するライブラリを提供しているため、ユニットテスト以上の機能テストも実施可能です。
Terraformの設定内容のテストは、outputの値を参照して実施するため、Terraformコードにテストすべき項目を記載する必要があります。また、Terratestのライブラリの制約により、outputを整形しないとうまくテストできません。
ユニットテスト以外にも使える点としては、その他のインフラのテストツールより優れていますが、Terraformのstatefileを参照している点や、出力の制約があるために導入にはその点を考慮する必要があります。
