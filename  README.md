cloudfront から ALB へのアクセスは HTTPS のみにしている
cloudFront への HTTP はリダイレクト

### S3 バケット作成

```
aws s3 mb s3://wordpress-stack-artifacts --region us-east-1
```

## ドメイン登録

### ContactInfo.json

```
{
"FirstName": "Hiroto",
"LastName": "Soda",
"ContactType": "PERSON", // 法人は COMPANY
"OrganizationName": "", // 個人の場合は空欄、法人の場合は会社名を記入
"AddressLine1": "1-2-3 Example St.",
"City": "Minato",
"State": "JP-13", // 東京都 (JP-13)
"CountryCode": "JP",
"ZipCode": "106-0032",
"PhoneNumber": "+81.9012345678",
"Email": "hiroto.soda@hiroto-soda.com"
}
```

### route53 に登録

```
aws route53domains register-domain \
 --region us-east-1 \
 --domain-name DOMAIN_NAME \
 --duration-in-years 1 \
 --admin-contact file://ContactInfo.json \
 --registrant-contact file://ContactInfo.json \
 --tech-contact file://ContactInfo.json
```

### ドメイン確認

```
aws route53domains list-domains --region us-east-1
```

### ドメインの承認

メールが来るはず

### ホストゾーン ID 確認

```
aws route53 list-hosted-zones \
  --query "HostedZones[?Name=='DOMAIN_NAME.'].Id" \
  --output text
```

### ACM を先に作る

```
aws cloudformation deploy \
  --template-file cloudformation/acm.yaml \
  --stack-name ACMStack \
  --region us-east-1 \
  --parameter-overrides \
   DomainName="DOMAIN_NAME" \
   HostedZoneId="HOST_ZONE_ID" \
  --capabilities CAPABILITY_NAMED_IAM
```

※ CloudFront に使う ACM 証明書と WAFv2 WebACL は、グローバルリージョン（us-east-1）でしか作成できない

### ACM Arn を取得

```
aws cloudformation describe-stacks --stack-name ACMStack --region us-east-1 --query "Stacks[0].Outputs"

```

aws acm describe-certificate \
 --region us-east-1 \
 --certificate-arn ACM_ARN \
 --query "Certificate.Status"

### キー作成

```
aws ec2 create-key-pair --key-name NewKey --query 'KeyMaterial' --output text > NewKey.pem --region us-east-1
```

### SSH 用のパーミッション設定

```
chmod 400 NewKey.pem
```

### デプロイ

```
aws cloudformation package --template-file main.yaml \
 --s3-bucket wordpress-stack-artifacts \
 --output-template-file main-artifact.yml \
 --region us-east-1

aws cloudformation deploy \
 --template-file main-artifact.yml \
 --stack-name wordpress-stack \
 --parameter-overrides \
   KeyName="NewKey" \
   WebACLArn="ACL_ARN" \
   ACMArn="ACM_ARN" \
 --capabilities CAPABILITY_NAMED_IAM \
 --region us-east-1
```

## RDS

```
mysql -h RDSInstanceEndpoint>-P 3306 -u DB_USER -p
```

※ パスワードはシークレットマネージャー

## EC2

```
ssh -i NewKey.pem ec2-user@EC2_public_IP
```
