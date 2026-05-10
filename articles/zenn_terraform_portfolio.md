---
title: "「再現できる気がしない」を卒業する：TerraformでAWSポートフォリオを0から構築した"
emoji: "🏗️"
type: "tech"
topics: ["terraform", "aws", "s3", "cloudfront", "route53"]
published: false
---

## はじめに

AWSでポートフォリオサイトを作ったり、チャットボットを作ったりしてきたけど、「再現できる気がしない」という感覚が拭えなかった。

コンソールをポチポチしたり、Amplifyに丸投げしたりしていたから、**なぜ動くかわからないまま動いていた**という状態だった。

そこでTerraformを使って、S3 + CloudFront + ACM + Route53の構成を0からコードで書いて再現できるところまでやってみた。

---

## 作った構成

```
CloudFront (xxxx.cloudfront.net)
    ↓ OAC（Origin Access Control）
S3（静的サイトホスティング）

ACM（SSL証明書 / us-east-1）
Route53（ホストゾーン / 学習用）
```

:::message
本来はRoute53のNSを使って独自ドメインまで繋げる設計だったが、CloudflareでドメインをRするとNS変更ができないため、今回はCloudFrontの標準ドメインで動作確認している。本来の設計については末尾に記載。
:::

---

## 環境

- Mac（Apple Silicon）
- Terraform v1.15.2
- AWS CLI 2.34.45
- 作業ディレクトリ：`~/terraform-portfolio`

---

## ステップ0：環境構築

```bash
# Terraformインストール
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# AWS CLIインストール
brew install awscli

# 確認
terraform --version
aws --version
```

AWS CLIの認証設定：

```bash
aws configure
# Access Key ID、Secret Access Key、リージョン（ap-northeast-1）を入力
```

設定確認：

```bash
aws sts get-caller-identity
```

アカウント情報が返ってきたらOK。

---

## ファイル構成

```
terraform-portfolio/
├── main.tf          # リソース定義
├── variables.tf     # 変数定義
├── .gitignore
├── .terraform.lock.hcl
├── index.html       # テスト用
└── README.md
```

---

## variables.tf

```hcl
variable "bucket_name" {
  description = "S3バケット名"
  default     = "exobrainlab-portfolio-tftest"
}

variable "domain_name" {
  description = "ドメイン名"
  default     = "exobrainlab.com"
}

variable "aws_region" {
  description = "AWSリージョン"
  default     = "ap-northeast-1"
}
```

---

## main.tf

### プロバイダー設定

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 東京リージョン（メイン）
provider "aws" {
  region = var.aws_region
}

# バージニア北部（ACM用）
# CloudFrontはus-east-1の証明書しか使えない
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}
```

:::message
ACMをus-east-1で作る理由：CloudFrontはグローバルサービスで、証明書をus-east-1からしか取得できない仕様になっている。ここは初見で必ずハマるポイント。
:::

### S3

```hcl
resource "aws_s3_bucket" "portfolio" {
  bucket        = var.bucket_name
  force_destroy = true  # terraform destroy時に中身ごと削除
}

# パブリックアクセスを全部ブロック（CloudFront経由のみ許可するため）
resource "aws_s3_bucket_public_access_block" "portfolio" {
  bucket = aws_s3_bucket.portfolio.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 静的ウェブサイトホスティングの設定
resource "aws_s3_bucket_website_configuration" "portfolio" {
  bucket = aws_s3_bucket.portfolio.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# CloudFrontだけがS3を読めるバケットポリシー
resource "aws_s3_bucket_policy" "portfolio" {
  bucket = aws_s3_bucket.portfolio.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontServicePrincipal"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.portfolio.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.portfolio.arn
          }
        }
      }
    ]
  })
}
```

:::details block_public_acls と ignore_public_acls の違い

ACLには「作る」と「読む」の2段階がある。

| 設定 | 役割 |
|---|---|
| `block_public_acls` | 新規ACLをブロック |
| `ignore_public_acls` | 既存ACLを無視 |
| `block_public_policy` | 新規ポリシーをブロック |
| `restrict_public_buckets` | 既存ポリシーを無効化 |

4つ全部 `true` にするのが「完全にパブリックアクセスを遮断する」定石。
:::

### ACM

```hcl
resource "aws_acm_certificate" "portfolio" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "www.${var.domain_name}"
  ]

  lifecycle {
    create_before_destroy = true
  }
}
```

### Route53（学習用）

```hcl
resource "aws_route53_zone" "portfolio" {
  name = var.domain_name
}

# ACMのDNS検証レコード
resource "aws_route53_record" "acm_validation" {
  for_each = {
    for dvo in aws_acm_certificate.portfolio.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = aws_route53_zone.portfolio.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}
```

:::message
`for dvo in ...` はPythonの内包表記と同じ発想。`dvo` は `Domain Validation Option` の略として慣習的に使われているが、変数名は何でもいい。
:::

### CloudFront

```hcl
resource "aws_cloudfront_origin_access_control" "portfolio" {
  name                              = "${var.bucket_name}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "portfolio" {
  enabled             = true
  default_root_object = "index.html"

  origin {
    domain_name              = aws_s3_bucket.portfolio.bucket_regional_domain_name
    origin_id                = "S3Origin"
    origin_access_control_id = aws_cloudfront_origin_access_control.portfolio.id
  }

  default_cache_behavior {
    target_origin_id       = "S3Origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

---

## デプロイ

```bash
terraform init    # プロバイダーのダウンロード
terraform plan    # 何が作られるか確認
terraform apply   # 実際に作成
```

`terraform plan` の読み方：

- `+` 緑：新しく作られるリソース
- `~` 黄：変更されるリソース
- `-` 赤：削除されるリソース
- `Plan: N to add, N to change, N to destroy.` で全体サマリー

---

## 動作確認

index.htmlをS3にアップロードして確認：

```bash
aws s3 cp ./index.html s3://exobrainlab-portfolio-tftest/
```

CloudFrontのURLを確認：

```bash
terraform state show aws_cloudfront_distribution.portfolio | grep domain_name
```

ブラウザでアクセスしてページが表示されればOK。

---

## 全部消して再現できるか確認

```bash
terraform destroy
aws s3 cp ./index.html s3://exobrainlab-portfolio-tftest/  # 再アップロード
```

再度アクセスして同じページが表示されれば「コードで管理できている」状態。

:::message
`terraform destroy` 前にS3を空にしておく必要がある。または `force_destroy = true` をS3バケットに設定しておくと `terraform destroy` 一発で中身ごと削除できる。
:::

---

## ハマったポイント

### 1. ACMはus-east-1固定

CloudFrontで使うACM証明書は必ずus-east-1で作る必要がある。東京リージョンで作っても使えない。

### 2. 全角文字混入

日本語入力モードでコードを書くとハイフンが全角（`－`）になる。`us-east-1` が `us－east－1` になってエラーになることがある。コードを書くときは常に半角モードで。

### 3. S3バケットが空でないと削除できない

`terraform destroy` 時にS3にファイルが残っていると `BucketNotEmpty` エラーになる。`force_destroy = true` を設定しておくか、先に `aws s3 rm s3://バケット名/ --recursive` で空にする。

### 4. CloudflareドメインはRoute53に移管できない

Cloudflareで取得したドメインはネームサーバーの変更が制限されているため、Route53への完全移管ができない。Route53のホストゾーンを作っても、CloudflareのNSが向いていない限り世界のDNSには影響しない。

---

## .gitignore

```
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
```

`terraform.tfstate` にはAWSのリソースIDや設定の詳細が含まれるためGitHubには上げない。

---

## 本来の設計

CloudflareのNS変更ができる場合、または別のレジストラでドメインを取得している場合は以下の構成が完全版：

```
Route53（DNS）
    ↓ Aliasレコード
CloudFront + ACM（独自ドメイン + HTTPS）
    ↓ OAC
S3
```

追加で必要なリソース：

- `aws_acm_certificate_validation`（証明書の検証完了を待つ）
- `aws_route53_record`（apex と www の Alias レコード）
- CloudFrontの `aliases` に独自ドメインを設定
- `viewer_certificate` に ACM証明書のARNを設定

---

## まとめ

コンソールのポチポチと違って、Terraformを使うと：

- **構成がコードとして残る** → GitHubで管理できる
- **再現性がある** → `terraform destroy` して `terraform apply` すれば同じ環境が戻る
- **差分が見える** → `terraform plan` で何が変わるかを事前に確認できる

「再現できる気がしない」という感覚の正体は「構成が頭に入っていない」ことだった。Terraformで書くことで、何が存在していて、どう繋がっているかを自分で宣言する必要があり、それが理解に繋がった。
