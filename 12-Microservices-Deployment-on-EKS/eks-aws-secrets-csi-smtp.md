# EKS에서 Kubernetes Secret 없이 AWS 암호화 리소스로 SMTP 자격증명 주입하기 (Secrets Manager/SSM + KMS)

요구사항: **Kubernetes Secret을 만들지 않고**, AWS 자체 암호화 리소스(**AWS Secrets Manager** 또는 **SSM Parameter Store SecureString + KMS**)에 저장된 비밀번호/계정을 EKS Pod에서 사용.

권장 패턴: **Secrets Store CSI Driver + AWS Provider(ASCP)**  
- **K8s Secret 생성 없이** Pod에 **파일로 마운트**해서 사용  
- IAM은 **IRSA(ServiceAccount ↔ IAM Role)** 로 최소권한 부여

---

## 1) 아키텍처 개요

- 개발자: SMTP 자격증명/From 주소를 AWS Secrets Manager(또는 SSM SecureString)에 저장
- EKS Pod: IRSA로 AWS API 권한을 받아
- Secrets Store CSI Driver가 런타임에 비밀을 조회 후 **/mnt/secrets-store/** 경로에 파일로 마운트
- 앱은 **파일을 읽어** SMTP 설정을 사용(또는 entrypoint에서 파일을 파싱해 env로 export)

> 주의: 이 구성은 보통 “SMTP 서버를 Pod로 띄워 메일을 수신”하는 방식이 아니라, **외부 SMTP(AWS SES SMTP 등)로 붙어 메일을 발신**하는 패턴입니다.

---

## 2) AWS에 비밀 저장 (둘 중 하나 선택)

### A) AWS Secrets Manager (권장)
JSON 한 덩어리로 저장하는 예시:

```bash
aws secretsmanager create-secret \
  --name notif/smtp \
  --secret-string '{"username":"SMTP_USER","password":"SMTP_PASS","from":"verified_from@example.com"}'
```
---
```
{
    "ARN": "arn:aws:secretsmanager:ap-northeast-2:086015456585:secret:notif/smtp-ehaQQu",
    "Name": "notif/smtp",
    "VersionId": "90d87fba-9622-4ffd-82f4-ce67704ff645"
}
kimdy@DESKTOP-CLQV18N:~/Eks-Class$ aws secretsmanager describe-secret \
  --secret-id arn:aws:secretsmanager:ap-northeast-2:086015456585:secret:notif/smtp-ehaQQu \
  --region ap-northeast-2
{
    "ARN": "arn:aws:secretsmanager:ap-northeast-2:086015456585:secret:notif/smtp-ehaQQu",
    "Name": "notif/smtp",
    "LastChangedDate": "2026-02-05T14:28:42.920000+09:00",
    "VersionIdsToStages": {
        "90d87fba-9622-4ffd-82f4-ce67704ff645": [
            "AWSCURRENT"
        ]
    },
    "CreatedDate": "2026-02-05T14:28:42.887000+09:00"
```


### B) SSM Parameter Store SecureString (+KMS)
키를 분리 저장하고 싶을 때 예시:

```bash
aws ssm put-parameter \
  --name "/notif/smtp/password" \
  --type "SecureString" \
  --value "SMTP_PASS" \
  --overwrite
```

---

## 3) EKS 구성: IRSA + SecretProviderClass + Deployment (K8s Secret 없음)

아래는 **Secrets Manager** 기준 예시입니다.

### 3.1 Namespace + ServiceAccount(IRSA) + SecretProviderClass + Deployment

> `eks.amazonaws.com/role-arn` 을 본인 IAM Role ARN으로 교체하세요.  
> `objectName` 은 Secrets Manager Secret 이름으로 맞추세요.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: notifications
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-sa
  namespace: notifications
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-notification-secrets-role
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: notif-smtp-spc
  namespace: notifications
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "notif/smtp"
        objectType: "secretsmanager"
        objectAlias: "smtp.json"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-microservice
  namespace: notifications
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notification-restapp
  template:
    metadata:
      labels:
        app: notification-restapp
    spec:
      serviceAccountName: notification-sa
      containers:
        - name: notification-service
          image: stacksimplify/kube-notifications-microservice:4.0.0-AWS-XRay
          ports:
            - containerPort: 8096
          volumeMounts:
            - name: aws-secrets
              mountPath: /mnt/secrets-store
              readOnly: true
          env:
            # SMTP 서버 주소(예: SES SMTP 엔드포인트)
            - name: AWS_MAIL_SERVER_HOST
              value: "email-smtp.ap-northeast-2.amazonaws.com"
            # 앱이 파일을 읽을 수 있도록 파일 경로만 env로 전달 (비밀값 자체를 env로 넣지 않음)
            - name: SMTP_SECRET_FILE
              value: "/mnt/secrets-store/smtp.json"
      volumes:
        - name: aws-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "notif-smtp-spc"
```

### 3.2 적용 & 확인

```bash
kubectl apply -f notifications-aws-secrets-csi.yaml
kubectl get all -n notifications

# 마운트된 파일 확인
kubectl -n notifications exec -it deploy/notification-microservice -- sh -lc \
  'ls -l /mnt/secrets-store && cat /mnt/secrets-store/smtp.json'
```

---

## 4) IAM 최소권한 정책 예시 (IRSA Role에 부여)

Secrets Manager에서 특정 Secret만 읽도록:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadNotifSmtpSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:notif/smtp-*"
    }
  ]
}
```

> IRSA Trust Policy는 **EKS OIDC Provider** 및 **ServiceAccount(subject)** 조건으로 구성해야 합니다.

---

## 5) 앱이 env만 지원하는 경우(파일 입력 미지원) 대안

만약 컨테이너가 `AWS_MAIL_SERVER_USERNAME` / `AWS_MAIL_SERVER_PASSWORD` 같은 **환경변수만** 받고,
파일(`/mnt/secrets-store/smtp.json`)을 직접 읽지 못한다면:

- **entrypoint 스크립트**에서 json 파싱 후 `export` 하고 앱 실행
- 또는 initContainer에서 파싱 결과를 shared emptyDir에 쓰고 main 컨테이너가 읽도록

예시(개념):

1) CSI로 `/mnt/secrets-store/smtp.json` 마운트  
2) `command: ["sh","-lc", "export AWS_MAIL_SERVER_USERNAME=$(jq -r .username ...); ...; exec /app/start"]`

> 이 경우 이미지 안에 `sh`, `jq` 존재 여부가 중요합니다.

---

## 6) “메일 수신(받기)”을 원한다면

Kubernetes Pod로 SMTP 수신 서버(MX)를 운영하는 개념은 별도 인프라(도메인 DNS/MX, 스팸/보안, 저장소 등)가 필요합니다.  
AWS에서는 보통 **SES Receiving Rule Set + S3/Lambda** 같은 서버리스 구성이 일반적입니다.

---

## 참고 링크

- AWS Secrets Store CSI Driver Provider(ASCP): https://github.com/aws/secrets-store-csi-driver-provider-aws
- AWS Security Blog (EKS add-on + Secrets Manager): https://aws.amazon.com/blogs/security/how-to-use-the-secrets-store-csi-driver-provider-amazon-eks-add-on-with-secrets-manager/
- SES SMTP 엔드포인트/접속: https://docs.aws.amazon.com/ses/latest/dg/smtp-connect.html
- SSM SecureString(KMS): https://docs.aws.amazon.com/systems-manager/latest/userguide/secure-string-parameter-kms-encryption.html
