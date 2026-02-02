# EKS / eksctl 관련 정리 (서울 리전 + 403 권한 오류 + 사용자 정의 정책)

> 대화에서 다룬 내용을 한 문서로 정리했습니다.  
> 목표: `eksctl create cluster` 실행을 **서울 리전(ap-northeast-2)** 에서 진행하면서, 발생한 **EKS 403(AccessDenied)** 문제를 해결하고,  
> **관리자급(AdministratorAccess)** 까지는 아니지만 EKS 생성/운영에 필요한 “연관 리소스”까지 다룰 수 있는 **사용자 정의 IAM 정책(JSON)** 예시를 제공합니다.

---

## 1) 서울(Seoul) 리전으로 eksctl 명령 변경

아래처럼 `--region` 과 `--zones` 를 서울 리전에 맞춰 변경합니다.

```bash
eksctl create cluster --name=eksdemo1 \
  --region=ap-northeast-2 \
  --zones=ap-northeast-2a,ap-northeast-2b \
  --without-nodegroup
```

---

## 2) 오류 메시지(403 AccessDenied) 의미

오류 예시:

```
AccessDeniedException: User: arn:aws:iam::086015456585:user/devuser is not authorized to perform:
eks:DescribeClusterVersions ...
because no identity-based policy allows the eks:DescribeClusterVersions action
```

### 핵심 원인
- `eksctl` 이 실행 초기에 **EKS 지원 Kubernetes 버전 목록**을 조회하려고 `eks:DescribeClusterVersions` 를 호출합니다.
- 현재 실행 주체(`devuser` 또는 assumed role)에 **해당 권한을 Allow 하는 “Identity-based policy”** 가 없어서 403이 발생합니다.

---

## 3) 빠른 점검 체크리스트 (권한 붙였는데도 계속 403일 때)

### 3.1 지금 어떤 자격증명으로 실행 중인지 확인
```bash
aws sts get-caller-identity
aws configure list
echo $AWS_PROFILE
```

- ARN이 정말 `...:user/devuser` 인지 확인  
- 만약 `...:assumed-role/...` 로 나오면, **유저가 아니라 Role 권한**을 점검해야 합니다.

### 3.2 devuser에 실제로 정책이 붙어 있는지 확인
```bash
aws iam list-attached-user-policies --user-name devuser
aws iam list-user-policies --user-name devuser
```

그룹 정책을 통해 권한을 받는 구조라면:
```bash
aws iam list-groups-for-user --user-name devuser
aws iam list-attached-group-policies --group-name <GROUP_NAME>
aws iam list-group-policies --group-name <GROUP_NAME>
```

### 3.3 Permission Boundary 여부 확인
Boundary가 걸려 있으면 인라인 정책을 붙여도 여전히 막힐 수 있습니다.
```bash
aws iam get-user --user-name devuser --query "User.PermissionsBoundary"
```

### 3.4 딱 이 에러만 막 뚫는 최소 인라인 정책(DescribeClusterVersions)
```bash
cat > eks-describe-versions.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDescribeClusterVersions",
      "Effect": "Allow",
      "Action": "eks:DescribeClusterVersions",
      "Resource": "*"
    }
  ]
}
JSON

aws iam put-user-policy \
  --user-name devuser \
  --policy-name AllowDescribeClusterVersions \
  --policy-document file://eks-describe-versions.json
```

테스트:
```bash
aws eks describe-cluster-versions --region ap-northeast-2
```

---

## 4) “슈퍼권한”에 가까운 정책(참고)

- 가장 포괄적: `AdministratorAccess`
- 다만 조직(Organizations) **SCP** 나 **Permission Boundary** 로 막혀 있으면, 관리자급 정책을 붙여도 특정 액션이 거부될 수 있습니다.

---

## 5) EKS + 연관 리소스까지 가능한 사용자 정의 정책(JSON)

> 아래는 “관리자급은 아니지만”, **EKS 클러스터/노드그룹/네트워크/CloudFormation/IAM Role(OIDC 포함)** 등  
> eksctl 사용 시 자주 필요한 영역을 **폭넓게 허용**하는 예시입니다.  
> (환경/옵션에 따라 더 줄이거나 추가가 필요할 수 있습니다.)

### 5.1 A안: eksctl이 웬만하면 다 되는 “PoC/개발용” (IAM 범위가 넓음)

> 계정 ID: `086015456585` (필요 시 변경)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StsCallerIdentity",
      "Effect": "Allow",
      "Action": ["sts:GetCallerIdentity"],
      "Resource": "*"
    },
    {
      "Sid": "EKSAll",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    },
    {
      "Sid": "EC2DescribeReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*",
        "ec2:SearchTransitGatewayRoutes"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2NetworkAndComputeForEKS",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:ModifySubnetAttribute",
        "ec2:CreateInternetGateway",
        "ec2:DeleteInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway",
        "ec2:AllocateAddress",
        "ec2:ReleaseAddress",
        "ec2:CreateNatGateway",
        "ec2:DeleteNatGateway",
        "ec2:CreateRouteTable",
        "ec2:DeleteRouteTable",
        "ec2:CreateRoute",
        "ec2:ReplaceRoute",
        "ec2:DeleteRoute",
        "ec2:AssociateRouteTable",
        "ec2:DisassociateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:CreateLaunchTemplate",
        "ec2:CreateLaunchTemplateVersion",
        "ec2:DeleteLaunchTemplate",
        "ec2:DeleteLaunchTemplateVersions",
        "ec2:ModifyLaunchTemplate",
        "ec2:RunInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudFormationForEksctlStacks",
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DeleteStack",
        "cloudformation:Describe*",
        "cloudformation:List*",
        "cloudformation:Get*",
        "cloudformation:ValidateTemplate",
        "cloudformation:CreateChangeSet",
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DeleteChangeSet"
      ],
      "Resource": [
        "arn:aws:cloudformation:*:086015456585:stack/eksctl-*/*"
      ]
    },
    {
      "Sid": "CloudFormationReadForAll",
      "Effect": "Allow",
      "Action": [
        "cloudformation:DescribeStacks",
        "cloudformation:ListStacks",
        "cloudformation:ListExports",
        "cloudformation:ListImports"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMForEksctlRolesAndProfiles",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:UpdateRole",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:TagRole",
        "iam:UntagRole",
        "iam:PassRole",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:CreateServiceLinkedRole"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMOIDCProviderForEKS",
      "Effect": "Allow",
      "Action": [
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider",
        "iam:TagOpenIDConnectProvider",
        "iam:UntagOpenIDConnectProvider",
        "iam:UpdateOpenIDConnectProviderThumbprint",
        "iam:GetOpenIDConnectProvider"
      ],
      "Resource": "arn:aws:iam::086015456585:oidc-provider/*"
    },
    {
      "Sid": "AutoscalingForNodegroups",
      "Effect": "Allow",
      "Action": [
        "autoscaling:*",
        "autoscaling-plans:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ELBForK8sIngressAndServices",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogsForControlPlaneAndWorkloads",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:DeleteLogGroup",
        "logs:PutRetentionPolicy",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:GetLogEvents",
        "logs:FilterLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SSMParametersForEKSOptimizedAMI",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRPullForClusterWorkloads",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KMSForEKSSecretsEncryptionIfUsed",
      "Effect": "Allow",
      "Action": [
        "kms:ListKeys",
        "kms:ListAliases",
        "kms:DescribeKey",
        "kms:CreateGrant"
      ],
      "Resource": "*"
    }
  ]
}
```

#### A안 주의사항
- `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole` 같은 IAM 권한이 넓으면 **권한 상승 리스크**가 생길 수 있습니다.
- 운영/보안 환경이라면 B안처럼 **Permissions Boundary 강제**가 더 안전합니다.

---

### 5.2 B안: Permissions Boundary 강제로 권한 상승을 막는 방식(권장)

핵심:
1) `EksctlBoundary` 라는 **permissions boundary 정책**을 계정에 만들어 둡니다.
2) 사용자 정책에서 `iam:CreateRole` 등을 허용하되, **반드시 그 Boundary를 붙일 때만** 허용합니다.

> 아래는 “CreateRole 시 Boundary 강제”가 들어간 사용자 정책 예시입니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSAndInfraSameAsOptionA",
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity",
        "eks:*",
        "ec2:Describe*",
        "ec2:Get*",
        "cloudformation:*",
        "autoscaling:*",
        "elasticloadbalancing:*",
        "logs:*",
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath",
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability",
        "kms:ListKeys",
        "kms:ListAliases",
        "kms:DescribeKey",
        "kms:CreateGrant"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMRead",
      "Effect": "Allow",
      "Action": ["iam:Get*", "iam:List*"],
      "Resource": "*"
    },
    {
      "Sid": "IAMCreateRoleOnlyWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:UpdateRole",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:TagRole",
        "iam:UntagRole"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::086015456585:policy/EksctlBoundary"
        }
      }
    },
    {
      "Sid": "IAMPassRoleLimited",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::086015456585:role/*"
    },
    {
      "Sid": "IAMInstanceProfileOps",
      "Effect": "Allow",
      "Action": [
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMOIDCProvider",
      "Effect": "Allow",
      "Action": [
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider",
        "iam:TagOpenIDConnectProvider",
        "iam:UntagOpenIDConnectProvider",
        "iam:UpdateOpenIDConnectProviderThumbprint",
        "iam:GetOpenIDConnectProvider"
      ],
      "Resource": "arn:aws:iam::086015456585:oidc-provider/*"
    }
  ]
}
```

---

## 6) EKS 관련 “AWS 관리형 정책(EKS 이름 포함)” 빠르게 목록 뽑기(참고)

콘솔에서 찾기 어렵다면 CLI로 검색:

```bash
aws iam list-policies --scope AWS \
  --query "Policies[?contains(PolicyName, 'EKS')].[PolicyName,Arn]" \
  --output table
```

---

## 7) 다음 단계(선택): 최소권한으로 더 줄이기

더 타이트한 최소권한 정책을 만들려면 아래 정보를 기반으로 권한을 줄입니다.

- 기존 VPC 사용 여부 (`--vpc-*` 옵션)
- managed nodegroup / fargate 여부
- IRSA(OIDC) 사용 여부
- Control plane 로그/암호화(KMS) 사용 여부

`eksctl create cluster ...` 전체 명령(옵션 포함)을 주면, 그 구성에 맞춰 **최소권한 JSON**으로 재구성 가능합니다.

---
