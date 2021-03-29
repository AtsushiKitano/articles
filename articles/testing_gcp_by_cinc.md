---
title: CINCを使ったGCPの設定内容のテスト
emoji: 📝
type: tech
topics: [gcp, InSpec, CINC]
published: true
---

# 概要
CINCを使ったGCPの設定値をテストする方法について。詳細設計に沿ってGCPが設定されているかの確認にCINCを使う方法について説明します。
CINCはChef Softwareが提供するソフトウェアのパッケージをOSSで提供しています。CINCを利用することで、ライセンス購入せずにInSpecの利用が可能となります。

# [CINCとは](https://cinc.sh/goals/)
[Chef Softwareは2019年4月2日より、全製品をソフトウェア配布物に商用ライセンス条項の適用を開始しました。](https://blog.chef.io/chef-software-announces-the-enterprise-automation-stack)そのため、現在、商用利用するには有償ライセンスの購入が必要となります。そこで、CINC(CINC is not Chefの略)はChef Softwareが提供しているソフトウェアを自由に使えるように活動しています。

CINCプロジェクトはChef Softwareのリポジトリのコードをビルドし、パッチ適用したCINCバイナリを配布しています。このバイナリのライセンスがApache2.0となっており商用利用可能となっています。(EULAやライセンス不要)
CINCはChef Softwareのソフトウェアのみを対象にしているため、その他のソフトウェアは対象にしていません。(test-kitchenなど)しかしながら、Chef Softwareのソフトウェアとの互換のあるサードパーティツールツールとの互換も保証するよう活動しているとのことです。

Chef Softwareが提供する各ソフトウェアごとのCINCバイナリは、以下の通りとなっています。

- Cinc Workstation: Chef Workstation
- Cinc Auditor: Chef InSpec
- Cinc Dashboard: Chef Automate
- Cinc Packager: Chef Habitat

この記事では、InSpecを使ったクラウドの設定をテストする例を示すので、`Cinc Auditor`の使い方を説明します。

# [CINC Auditorのインストール方法](https://cinc.sh/start/auditor/)
 
 スクリプトによる各OSでのインストール方法は下記の通りです。
 
 - UNIX/Linux/MacOS
 
 ```
$ curl -L https://omnitruck.cinc.sh/install.sh | sudo bash -s -- -P cinc-auditor -v 4
 ```
 
 - Windows
 
 ```
 . { iwr -useb https://omnitruck.cinc.sh/install.ps1 } | iex; install -project cinc-auditor -version 4
 ```
 
 bundlerによるインストール方法は、下記の通りです。
 
 ```
source "https://packagecloud.io/cinc-project/stable" do
  gem "cinc-auditor-bin"
end
 ```

各バージョンのバイナリは、下記ページのOpen Source Labより、実行環境のOSに対応するバイナリを取得してください。

- Cinc Open Source Lab Mirrors: http://downloads.cinc.sh/files/stable/cinc/

インストールが完了すると、下記のように `cinc-auditor` コマンドを実行することができます。

```
$ cinc-auditor version
4.24.28
```

また、CINCのDockerイメージはDockerHubの[cincproject/auditor](https://hub.docker.com/r/cincproject/auditor)で提供されています。


# InSpec
InSpecはシステムの現在状態を確認し、詳細設計などのあるべき状態であるかを確認します。状態の確認はSDKにより実装されたメソッドを使い稼動中のシステムの状態値を取得し確認します。
各クラウドの状態を確認するメソッドは、[公式ページのリソースレファレンス](https://docs.chef.io/inspec/resources)を参照してください。

## InSpecの書き方
InSpecはRSpecベースの言語であるため、RSpecに準拠した文法となります。InSpecのディレクトリ構成は以下の通りとなります。

```
./
  ├── controls
  │   └── main.rb
  ├── files
  │   └── lamb_actual.yaml
  ├── attribute.yaml  
  ├── inspec.yml
  └── inspec.lock
```

- controls: テスト内容のコードが記述されたコード
- files: テスト内容のコードが呼び出す設定値などを定義したファイル
- attribute.yaml: テスト実行時に渡すファイル
- inspec.yml: 実行するinspecの内容を定義するファイル
- inspec.lock: 実行環境のinspecバージョン定義するファイル

上記の中で実行に必須となるのは、 `controls` と `inspec.yml` のみです。
このディレクトリのrootでinspecコマンドを実行すると、定義したテストを実行します。

### inspec.ymlの設定内容
inspec.ymlは、実行するテストの情報(profile)を定義するファイルとなっています。定義する内容は、以下の通りです。

| 項目            | 内容                                      | 必須 |
|-----------------|-------------------------------------------|------|
| name            | profileで一意になる名前                   | ○    |
| title           | profileのnameの読み易い値                 | ×    |
| maintainer      | profileを管理するユーザー名               | ×    |
| copyright       | copyright                                 | ×    |
| copyright_email | profileの管理者のメールアドレス           | ×    |
| license         | profileのライセンス                       | ×    |
| summary         | profileのサマリ                           | ×    |
| description     | profileの具体的な内容                     | ×    |
| version         | profileのバージョン                       | ×    |
| inspec_version  | inspec のバージョン                       | ×    |
| supports        | profileがサポートするplatform             | ×    |
| depends         | profileが依存するライブラリなどを定義する | ×    |
| inputs          | controlsのテストが参照するinputを定義する | ×    |

sshを実行する踏台サーバーのテストのinspec.ymlの例は以下の様になります。

```
name: ssh
title: Basic SSH
maintainer: Chef Software, Inc.
copyright: Chef Software, Inc.
copyright_email: support@chef.io
license: Proprietary, All rights reserved
summary: Verify that SSH Server and SSH Client are configured securely
version: 1.0.0
supports:
  - platform-family: linux
depends:
  - name: profile
    path: ../path/to/profile
inspec_version: "~> 2.1"
```

#### クラウドのテスト方法
AWS,GCP,Azureなどのパブリッククラウドをテストする方法は、inspec.ymlの`depends`に対象のクラウドのライブラリと`supports`に対象のプラットフォームを定義します。
例えばGCPをテストする場合、inspec.ymlは以下のようになります。

```
name: sample
depends:
  - name: inspec-gcp
    url: https://github.com/inspec/inspec-gcp/archive/<tag version>
supports:
  - platform: gcp
```

`tag version` は、GitHubの[tagsページ](https://github.com/inspec/inspec-gcp/tags)から、利用する対象のバージョンを指定します。例えば、1.8.8バージョンを利用する場合、tag versionに `v1.8.8`を指定します。

### InSpecのサンプル (HelloWorld)
ここでは、ローカルにあるyamlファイル(hello_world.yaml)の内容が、`name: HelloWorld`という文字列が含まれているかのテストサンプルを示します。

ディレクトリ構造は、以下の通りとします。

```
helloworld
 ├── controls
 │   └── main.rb
 ├── files
 │   └── hello_world.yaml
 └── inspec.yml
```

main.rb,hello_world.yamlとinspec.ymlの内容は以下の通りです。

- main.rb

```
actual = yaml(content: inspec.profile.file("hello_world.yaml"))

describe actual do
  its(:name) { should eq "HelloWorld" }
end
```
`inspec.profile.file()`メソッドにより、`files`ディレクトリのファイルをファイルパスなしに参照することができます。hello_world.yamlの`name`キーの値が、`HelloWorld`であるかをdescribe内でテストします。


- hello_world.yaml

```
name: HelloWorld
```

- inspec.yml

```
name: "test"
title: "testing hello world yaml file"
```

上記のファイルを作成し、ディレクトリのルートにて下記コマンドを実行しテストを実施します。

```
$ cinc-auditor exec .
```

実行結果は以下のようになります。

```
Profile: testing hello world yaml file (test)
Version: (not specified)
Target:  local://

  YAML content
     ✔  name is expected to eq "HelloWorld"

Test Summary: 1 successful, 0 failures, 0 skipped
```

# GCPの設定内容のテスト
次に、クラウド上に構築したシステムのテストについて説明します。ここではWebシステムをGCP上に構築し、その設定値をテストします。

![system_arch](https://storage.googleapis.com/zenn-user-upload/km1j41dvy0de2ctgp27asvwawjij)

このWebシステムは、VPCネットワーク内にGCEを作成し、その上でNginxが稼動するWebシステムです。
GCPの各設定項目は、以下の通りとなっています。

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

上記のシステムを構築するコードは、この[リポジトリ](https://github.com/AtsushiKitano/cinc-test-sample)のsrcディレクトリを参照してください。
システムの構築は以下のコマンドで作成されます。

```
$ terragrunt run-all apply
```
また、構築したシステムを削除するのは、以下のようにすることで削除ができます。

```
$ terraform destroy
```

CINCのテストは、さきのリポジトリのtestディレクトリに格納されています。コードは以下の通りです。

```
# coding: utf-8
network_expected_info = yaml(content: inspec.profile.file("network.yaml")).params # 想定されるvpc,サブネットワーク,ファイアーフォールの設定項目
gce_expected_info = yaml(content: inspec.profile.file("gce.yaml")).params # 想定されるgceインスタンス、gceディスクの設定項目
project_id = ENV["TF_VAR_project"]


control "network" do
  title "vpc,subnetwork,firewallの設定"

  vpc_expected = network_expected_info["vpc"]
  subnetwork_expected = network_expected_info["subnetwork"]
  firewall_expected = network_expected_info["firewall"]

  vpc_actual = google_compute_network(project: project_id, name: vpc_expected["name"]) # 構築したvpcの設定値
  subnetwork_actual = google_compute_subnetwork(project: project_id, region: subnetwork_expected["region"], name: subnetwork_expected["name"]) # 構築したサブネットワークの設定値
  firewall_actual = google_compute_firewall(project: project_id, name: firewall_expected["name"]) # 構築したファイアーフォールの設定値

  describe vpc_actual do
    it { should exist }
    its("name") { should cmp vpc_expected["name"] }
  end

  describe subnetwork_actual do
    it { should exist }
    its("name") { should cmp subnetwork_expected["name"] }
    its("region") { should match subnetwork_expected["region"] }
    its("ip_cidr_range") { should cmp subnetwork_expected["cidr"] }
  end

  describe firewall_actual do
    it { should exist }
    its("name") { should cmp firewall_expected["name"] }
    its("direction") { should cmp firewall_expected["direction"]}
    its("priority") { should cmp firewall_expected["priority"] }
    its("source_ranges") { should be_in firewall_expected["source_ranges"]}
  end

  firewall_actual.allowed.each do |rule|
    describe rule do
      its("ip_protocol") { should cmp firewall_expected["rule"]["protocol"] }
      its("ports") { should be_in firewall_expected["rule"]["ports"] }
    end
  end
end

control "gce" do
  title "gce instance, diskの設定"

  instance_expected = gce_expected_info["instance"] 
  disk_expected = gce_expected_info["disk"] 

  instance_actual = google_compute_instance(project: project_id, zone: instance_expected["zone"] ,name: instance_expected["name"]) # 構築したgceインスタンスの設定値
  disk_actual = google_compute_disk(project: project_id, name: instance_expected["name"], zone: instance_expected["zone"]) # 構築したgceディスクの設定値

  describe instance_actual do
    it { should exist }
    its("name") { should cmp instance_expected["name"] }
    its("zone") { should match instance_expected["zone"] }
    its("machine_type") { should match instance_expected["machine_type"] }
  end

  instance_actual.network_interfaces.each do |nic|
    describe nic do
      its("subnetwork") { should match instance_expected["interface"]["subnetwork"] }
    end
  end

  instance_actual.disks.each do |disk|
    describe disk do
      its("source") { should match instance_expected["disk_name"]}
    end
  end

  describe disk_actual do
    it { should exist }
    its("name") { should cmp disk_expected["name"] }
    its("zone") { should match disk_expected["zone"] }
    its("source_image") { should match disk_expected["image"] }
    its("size_gb") { should cmp disk_expected["size"]}
  end

end
```
InSpecでは複数のテスト項目である`describe`をまとめて`control`を定義することができます。このcontrolには、titleやimpactを定義することができます。

テストは以下のように実行します。

```
$ cinc-auditor exec . -t gcp://
```

実行コマンドにテスト対象のプラットフォームを指定する必要があり、`-t`で指定します。今回はGCPをテストするため、`gcp://`を渡します。
実行の結果、以下のような結果が出力されます。

```
Profile: nginx web system sample system test (http-system)
Version: (not specified)
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  network: vpc,subnetwork,firewallの設定
     ✔  Network sample is expected to exist
     ✔  Network sample name is expected to cmp == "sample"
     ✔  Subnetwork sample is expected to exist
     ✔  Subnetwork sample name is expected to cmp == "sample"
     ✔  Subnetwork sample region is expected to match "asia-northeast1"
     ✔  Subnetwork sample ip_cidr_range is expected to cmp == "192.168.10.0/24"
     ✔  Firewall ingress-sample is expected to exist
     ✔  Firewall ingress-sample name is expected to cmp == "ingress-sample"
     ✔  Firewall ingress-sample direction is expected to cmp == "INGRESS"
     ✔  Firewall ingress-sample priority is expected to cmp == 1000
     ✔  Firewall ingress-sample source_ranges is expected to be in "0.0.0.0/0"
     ✔  Firewall ingress-sample FirewallAllowed ip_protocol is expected to cmp == "tcp"
     ✔  Firewall ingress-sample FirewallAllowed ports is expected to be in "80"
  ✔  gce: gce instance, diskの設定
     ✔  Instance sample is expected to exist
     ✔  Instance sample name is expected to cmp == "sample"
     ✔  Instance sample zone is expected to match "asia-northeast1-b"
     ✔  Instance sample machine_type is expected to match "f1-micro"
     ✔  Instance sample InstanceNetworkInterfaces subnetwork is expected to match "sample"
     ✔  Instance sample InstanceDisks source is expected to match "sample"
     ✔  Disk sample is expected to exist
     ✔  Disk sample name is expected to cmp == "sample"
     ✔  Disk sample zone is expected to match "asia-northeast1-b"
     ✔  Disk sample source_image is expected to match "ubuntu-2004"
     ✔  Disk sample size_gb is expected to cmp == 20


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 1.8.8
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 2 successful controls, 0 control failures, 0 controls skipped
Test Summary: 24 successful, 0 failures, 0 skipped
```

## ライブラリが準備されていない設定項目のテスト
先の説明ではGCPの設定をテストするために[InSpecのgcpメソッド](https://docs.chef.io/inspec/resources#gcp)を利用しました。しかし、GCPのshared vpcの設定項目などは準備されていなかったりします。そのときの解決策のひとつを紹介します。

実施の流れは以下の通りとなります。

1. インフラの構築
1. SDKを使ったインフラの設定値をファイルに出力
1. 生成したファイルの各項目のテスト

構築したシステム設定値をSDKを用い取得しファイルに出力し、その値が想定したものか確認することで公式の各メソッドを使ったことと同様にテストの実施が可能になります。そのため、システム構築後に設定値を取得するタスクを実行します。

例えば先の例で、VPC、サブネットワーク、ファイアーフォール、GCEインスタンス、GCEディスクを`gcloud`コマンドで以下のように設定値をyaml形式で出力し、InSpecのyamlメソッドを使いyamlの各項目をInSpecでテストします。
この内容のテストは先のリポジトリの[no_lib_test](https://github.com/AtsushiKitano/cinc-test-sample/tree/master/no_lib_test)のディレクトリを参照してください。

- gcloudコマンドで各設定値を確認するコマンドl(scripts/main.sh)

```
#!/bin/bash

CONFIGFILES=files

gcloud compute networks describe sample --format=yaml > $CONFIGFILES/vpc_actual.yaml
gcloud compute networks subnets describe --region asia-northeast1 sample --format=yaml > $CONFIGFILES/subnetwork_actual.yaml
gcloud compute firewall-rules describe ingress-sample --format=yaml > $CONFIGFILES/firewall_actual.yaml
gcloud compute instances describe --zone=asia-northeast1-b sample --format=yaml > $CONFIGFILES/instance_actual.yaml
gcloud compute disks describe --zone=asia-northeast1-b sample --format=yaml > $CONFIGFILES/disk_actual.yaml
```

このスクリプトでは取得したファイルをfilesディレクトリに格納し、テストコードで参照しやすいようしています。

- テストコード

```
# coding: utf-8

network_expected_info = yaml(content: inspec.profile.file("network.yaml")).params
gce_expected_info = yaml(content: inspec.profile.file("gce.yaml")).params
project_id = ENV["TF_VAR_project"]


control "network" do
  title "vpc,subnetwork,firewallの設定"

  vpc_expected = network_expected_info["vpc"]
  subnetwork_expected = network_expected_info["subnetwork"]
  firewall_expected = network_expected_info["firewall"]

  vpc_actual = yaml(content: inspec.profile.file("vpc_actual.yaml"))
  subnetwork_actual = yaml(content: inspec.profile.file("subnetwork_actual.yaml"))
  firewall_actual = yaml(content: inspec.profile.file("firewall_actual.yaml"))

  describe "VPC" do
    it "VPC名が #{vpc_expected["name"]} である" do
      expect(vpc_actual[:name]).to eq vpc_expected["name"]
    end
  end

  describe "subnetwork" do
    it "サブネットワーク名が #{subnetwork_expected["name"]} である" do
      expect(subnetwork_actual[:name]).to eq subnetwork_expected["name"]
    end

    it "リージョンが #{subnetwork_expected["region"]} である" do
      expect(subnetwork_actual[:region]).to match subnetwork_expected["region"]
    end

    it "CIDRが #{subnetwork_expected["cidr"]} であること" do
      expect(subnetwork_actual[:ipCidrRange]).to eq subnetwork_expected["cidr"]
    end
  end

  describe "firewall" do
    it "ファイアーフォール名が #{firewall_expected["name"]} である" do
      expect(firewall_actual[:name]).to eq firewall_expected["name"]
    end

    it "方向が #{firewall_expected["direction"]} である" do
      expect(firewall_actual[:direction]).to eq firewall_expected["direction"]
    end

    it "プライオリティが #{firewall_expected["priority"]} である" do
      expect(firewall_actual[:priority]).to eq firewall_expected["priority"]
    end

    it "source rangeが #{firewall_expected["source_ranges"]} である" do
      expect(firewall_actual[:sourceRanges]).to eq firewall_expected["source_ranges"]
    end

    firewall_actual.allowed.each do |rule|
      it "プロトコルが #{firewall_expected["rule"]["protocol"]} である" do
        expect(rule["IPProtocol"]).to eq firewall_expected["rule"]["protocol"]
      end

      it "許可ポートが #{firewall_expected["rule"]["ports"]} である" do
        expect(rule["ports"]).to eq firewall_expected["rule"]["ports"]
      end
    end
  end
end

control "gce" do
  title "gce instance, diskの設定"

  instance_expected = gce_expected_info["instance"]
  disk_expected = gce_expected_info["disk"]

  instance_actual = yaml(content: inspec.profile.file("instance_actual.yaml"))
  disk_actual = yaml(content: inspec.profile.file("disk_actual.yaml"))

  describe "GCE" do
    it "インスタンス名が #{instance_expected["name"]} である" do
      expect(instance_actual[:name]).to eq instance_expected["name"]
    end

    it "zoneが #{instance_expected["zone"]} である" do
      expect(instance_actual[:zone]).to match instance_expected["zone"]
    end

    it "マシンタイプが #{instance_expected["machine_type"]} である" do
      expect(instance_actual[:machineType]).to match instance_expected["machine_type"]
    end

    instance_actual.networkInterfaces.each do |nic|
      it "サブネットワークの所属が #{instance_expected["interface"]["subnetwork"]} であること" do
        expect(nic["subnetwork"]).to match instance_expected["interface"]["subnetwork"]
      end
    end

    instance_actual.disks.each do |disk|
      it "ブートディスク名が#{instance_expected["disk_name"]}である" do
        expect(disk["source"]).to match match instance_expected["disk_name"]
      end
    end
  end

  describe "GCEDisk" do
    it "ディスク名が #{disk_expected["name"]} である" do
      expect(disk_actual[:name]).to eq disk_expected["name"]
    end

    it "ゾーンが #{disk_expected["zone"]} である" do
      expect(disk_actual[:zone]).to match disk_expected["zone"]
    end

    it "イメージが #{disk_expected["image"]} である" do
      expect(disk_actual[:sourceImage]).to match disk_expected["image"]
    end

    it "サイズが#{disk_expected["size"]}GBであること" do
      expect(disk_actual[:sizeGb]).to eq disk_expected["size"]
    end
  end
end
```

- 出力結果

```
$ cinc-auditor exec . -t gcp://

Profile: nginx web system sample system test (http-system)
Version: (not specified)
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

  ✔  network: vpc,subnetwork,firewallの設定
     ✔  VPC VPC名が sample である
     ✔  subnetwork サブネットワーク名が sample である
     ✔  subnetwork リージョンが asia-northeast1 である
     ✔  subnetwork CIDRが 192.168.10.0/24 であること
     ✔  firewall ファイアーフォール名が ingress-sample である
     ✔  firewall 方向が INGRESS である
     ✔  firewall プライオリティが 1000 である
     ✔  firewall source rangeが ["0.0.0.0/0"] である
     ✔  firewall プロトコルが tcp である
     ✔  firewall 許可ポートが ["80"] である
  ✔  gce: gce instance, diskの設定
     ✔  GCE インスタンス名が sample である
     ✔  GCE zoneが asia-northeast1-b である
     ✔  GCE マシンタイプが f1-micro である
     ✔  GCE サブネットワークの所属が sample であること
     ✔  GCE ブートディスク名がsampleである
     ✔  GCEDisk ディスク名が sample である
     ✔  GCEDisk ゾーンが asia-northeast1-b である
     ✔  GCEDisk イメージが ubuntu-2004 である
     ✔  GCEDisk サイズが20GBであること


Profile: Google Cloud Platform Resource Pack (inspec-gcp)
Version: 1.8.8
Target:  gcp://764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com

     No tests executed.

Profile Summary: 2 successful controls, 0 control failures, 0 controls skipped
Test Summary: 19 successful, 0 failures, 0 skipped
```

テスト対象がyamlファイルの内容なので、先の結果とは違い、すべてが`YAML content`のオブジェクトの値がどのようになっているかと出力されます。そのため、出力結果を分かりやすくなるように`describe`内の書き方に工夫する必要がでてきます。

# まとめ
InspecのOSSであるCINCについて、GCP上に構築したシステムをテストしながら説明をしました。実行内容のすべての実行方法は、リポジトリの[cloudbuildディレクトリのcloudbuild.yaml](https://github.com/AtsushiKitano/cinc-test-sample/blob/master/cloudbuild/cloudbuild.yaml)を参照してください。
Inspecは設定項目すべてをテストするメソッドを揃えていませんが、SDKを使いながら項目のテストをすることができます。
また、OSSであるため、案件でライセンス購入をせずともInSpecの活用ができます。
以上より、CINCを使うことでソフトウェアのようにインフラにもテストを導入することができるので、活用を検討してみてください。
