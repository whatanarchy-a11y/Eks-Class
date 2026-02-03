# EKS 노드 External IP는 없는데 Subnet이 Private 아닌 것처럼 보일 때 (원인/점검/해결)

아래 내용은 **`eksctl create nodegroup --node-private-networking`** 로 노드그룹을 만들었는데  
`kubectl get nodes -o wide` 에서 **External IP는 none** 으로 보이지만, 콘솔에서 확인한 Subnet이 **Private subnet이 아닌 것처럼 보이는** 상황을 정리한 문서입니다.

---

## 1) 핵심 요약

- **External IP가 없다** = “노드 ENI에 공인 IP를 안 붙였다”는 의미일 수 있음  
- 하지만 **Subnet이 Private** 인지는 **Route Table에서 `0.0.0.0/0`가 어디로 가는지**로 판정해야 함
  - **Private subnet**: `0.0.0.0/0 -> NAT Gateway (nat-xxxx)`
  - **Public subnet**: `0.0.0.0/0 -> Internet Gateway (igw-xxxx)`

즉, **노드 공인IP가 없더라도 subnet 자체가 public(IGW route)** 일 수 있습니다.

---

## 2) “진짜 Private Subnet” 판정 기준 3가지

### A. Route Table의 기본 경로(가장 확실)
- **Private subnet**: `0.0.0.0/0 -> NAT Gateway`
- **Public subnet**: `0.0.0.0/0 -> Internet Gateway`

> 콘솔 Route Table 탭에서 IGW로 나가면 그건 Public subnet입니다.

### B. Subnet 설정: Auto-assign public IPv4
- **Private subnet**: 보통 `Disable`
- **Public subnet**: 보통 `Enable`

콘솔 경로: **VPC → Subnets → (해당 subnet) → Edit subnet settings**

### C. Subnet 태그(관례)
- Public subnet 관례: `kubernetes.io/role/elb=1`
- Private subnet 관례: `kubernetes.io/role/internal-elb=1`

> 태그는 “의도/역할” 확인용이고, 최종 판정은 Route Table이 가장 확실합니다.

---

## 3) 흔한 원인 4가지

### 원인 1) 노드는 공인 IP를 안 받지만, Subnet은 여전히 Public(IGW)
- `--node-private-networking` 등으로 노드 공인IP가 없어도,
- Subnet route가 IGW면 그 Subnet 자체는 public입니다.

✅ 해결: 노드그룹이 붙을 서브넷을 **private subnet** 으로 확정(명시)하거나, 클러스터 구성에서 private subnet을 올바르게 잡기

---

### 원인 2) 태그/분류가 꼬여서 eksctl이 “private”를 잘못 고름
- 기존 VPC를 사용할 때 태그가 엉키면
  - private로 의도했지만 public subnet을 선택하는 케이스가 발생

✅ 해결: subnet 태그 정리 (`internal-elb` / `elb`) 및 eksctl 구성 재확인

---

### 원인 3) 콘솔에서 다른 클러스터/노드그룹의 Subnet을 보고 있음(이름 혼재)
- `eksdemo`, `eksdemo1`, `eksdemo2` 등 이름이 섞여 있으면
- 실제 노드 인스턴스가 붙은 subnet과, 콘솔에서 클릭한 subnet이 다를 수 있음

✅ 해결: **EC2 인스턴스의 Subnet ID** 를 먼저 확인 후 그 Subnet ID로 VPC 콘솔에서 조회

---

### 원인 4) Route Table association이 잘못 연결됨
- private subnet으로 만들었으나
- 나중에 subnet이 public route table(IGW)을 물고 있을 수 있음

✅ 해결: Subnet → Route table association을 private RT(NAT)로 재연결

---

## 4) CLI로 “노드가 붙은 Subnet”과 “Subnet 라우트”를 확실히 확인하기

### 4-1) 노드가 실제로 어느 Subnet에 붙었는지(ProviderID → EC2 인스턴스)
```bash
kubectl get nodes -o wide

kubectl get node <NODE_NAME> -o jsonpath='{.spec.providerID}{"\n"}'
# 예) aws:///ap-northeast-2a/i-0123456789abcdef0
```

ProviderID에서 인스턴스 ID(`i-...`)를 추출한 뒤:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query 'Reservations[0].Instances[0].SubnetId' \
  --output text
```

---

### 4-2) 해당 Subnet의 Route Table에서 0.0.0.0/0 대상 확인(IGW vs NAT)
```bash
SUBNET_ID=subnet-xxxx

aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]' \
  --output table
```

- 결과에 `GatewayId: igw-...` → **Public subnet**
- 결과에 `NatGatewayId: nat-...` → **Private subnet**

---

### 4-3) Auto-assign public IP 설정(MapPublicIpOnLaunch) 확인
```bash
aws ec2 describe-subnets \
  --subnet-ids $SUBNET_ID \
  --query 'Subnets[0].MapPublicIpOnLaunch' \
  --output text
```

- `true` → 자동 공인IP 할당 켜짐(대개 public subnet에서 사용)
- `false` → 자동 공인IP 할당 꺼짐(대개 private subnet에서 사용)

---

## 5) 결론

- **External IP none** 은 “노드 공인IP가 없다”는 힌트일 뿐,
- **Subnet이 Private인지** 는 **Route Table의 `0.0.0.0/0`가 NAT인지 IGW인지** 로 결정됩니다.

따라서 지금 상황은 보통
1) 노드 공인IP는 안 주게 됐지만,
2) 노드가 실제로는 IGW route를 가진 subnet(=public)에 들어갔거나,
3) private subnet으로 의도했던 subnet의 라우트/태그/association이 잘못된 경우입니다.

---

## 6) 빠른 진단 체크리스트

1. `kubectl get node <NODE> -o jsonpath='{.spec.providerID}'` 로 인스턴스 ID 확인
2. `aws ec2 describe-instances --instance-ids ... --query '...SubnetId'` 로 Subnet ID 확인
3. `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<SUBNET>` 로 0.0.0.0/0 목적지 확인
4. `aws ec2 describe-subnets --subnet-ids <SUBNET> --query 'Subnets[0].MapPublicIpOnLaunch'` 확인
5. (선택) subnet 태그 `kubernetes.io/role/elb` / `internal-elb` 확인
