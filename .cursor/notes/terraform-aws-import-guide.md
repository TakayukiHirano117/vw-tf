# AWSリソースのTerraformインポート手順メモ

**最終更新**: 2026-09-24

## 要点

- Terraform v1.5.0以降で追加された `import` ブロックとコード自動生成（`-generate-config-out`）を使うのが最も安全。
- VPC、Subnet、Route Table、Security Group、EC2、ALBなどのAWSリソースはTerraformで管理可能。
- **EC2内部で動いているDockerコンテナはAWS Providerの管理対象外**のため、Terraformでは直接importできない。
- 初期導入時は **module（モジュール化）は使わず、フラットな構成（単一ディレクトリ）** で進めるのが安全（アドレス解決の複雑化を防ぐため）。

## 元の文脈

- 質問: VPC、Subnet、EC2とDocker、ルートテーブル、SG、ALBを作った状態からTerraformでimportする手順と、module等の基礎知識について。

---

## 前提条件

- Terraform CLI: `v1.5.0` 以上
- AWS Provider: `~> 5.0`
- インポート対象のAWSリソースID（マネジメントコンソールやAWS CLIで確認）

---

## インポート手順（4ステップ）

### 1. Provider 設定ファイルの作成 (`main.tf`)

AWS Providerを定義し、初期化します。

```terraform
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1" # 対象リソースが存在するリージョン
}
```

ターミナルで初期化コマンドを実行します：

```bash
terraform init
```

### 2. インポート対象の定義 (`imports.tf`)

取り込みたいリソースと対応するIDを `import` ブロックで指定します。
`to` には将来のTerraformリソースアドレス（任意のローカル名）、`id` にはAWSリソースの識別子を指定します。

```terraform
# VPC (ID: vpc-xxxx)
import {
  to = aws_vpc.main
  id = "vpc-0123456789abcdef0"
}

# パブリックサブネット (ID: subnet-xxxx)
import {
  to = aws_subnet.public
  id = "subnet-0123456789abcdef0"
}

# ルートテーブル (ID: rtb-xxxx)
import {
  to = aws_route_table.public
  id = "rtb-0123456789abcdef0"
}

# セキュリティグループ (ID: sg-xxxx)
import {
  to = aws_security_group.app
  id = "sg-0123456789abcdef0"
}

# EC2インスタンス本体 (ID: i-xxxx)
# ※ 注意: EC2内部のDockerは取り込めません
import {
  to = aws_instance.app
  id = "i-0123456789abcdef0"
}

# Application Load Balancer (ID: ALBのARN)
import {
  to = aws_lb.main
  id = "arn:aws:elasticloadbalancing:ap-northeast-1:123456789012:loadbalancer/app/my-alb/1234567890abcdef"
}
```

### 3. コード自動生成 (`terraform plan`)

既存のインフラ設定を読み取り、対応するHCLコードを新規ファイル `generated.tf` に出力します。

```bash
terraform plan -generate-config-out=generated.tf
```

#### 注意点と調整
- 出力先ファイル（例: `generated.tf`）は事前に存在しないファイル名を指定する必要があります（既存ファイルがあるとエラーになります）。
- AWSリソース仕様により、排他的なパラメータ（例: `aws_instance` の `ipv6_address_count` と `ipv6_addresses` など）が両方出力されてプラン時に競合エラーが出ることがあります。その場合は `generated.tf` を開き、不要な属性を削除して調整します。

### 4. インポートの確定 (`terraform apply`)

内容を確認後、インポートを状態ファイル（`terraform.tfstate`）に反映します。

```bash
terraform apply
```

インポートが成功すると、`Plan: X to import, 0 to add, 0 to change, 0 to destroy` のように表示され、インフラ実体を変更せずにTerraform配下に組み込まれます。インポート完了後は `imports.tf` を削除またはコメントアウトしても問題ありません。

---

## 1次ソース（公式ドキュメント）

- **Terraform公式: インポート概要**  
  [Import resources overview | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import)
- **Terraform公式: コード自動生成 (`-generate-config-out`)**  
  [Import - Generating Configuration | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/generating-configuration)
- **AWS Provider公式: 各リソースのImport仕様**  
  - VPC: [aws_vpc | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#import)
  - Subnet: [aws_subnet | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet#import)
  - Route Table: [aws_route_table | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table#import)
  - Security Group: [aws_security_group | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group#import)
  - EC2 Instance: [aws_instance | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#import)
  - Application Load Balancer: [aws_lb | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb#import)
