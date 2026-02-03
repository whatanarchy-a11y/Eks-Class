# EKS - 프라이빗 서브넷에서 EKS 노드 그룹 생성

## 단계-01: 소개
- VPC 프라이빗 서브넷에 노드 그룹을 생성합니다.
- 프라이빗 노드 그룹에 워크로드를 배포하며, 워크로드는 프라이빗 서브넷에서 실행되고 로드 밸런서는 퍼블릭 서브넷에 생성되어 인터넷에서 접근 가능합니다.

## 단계-02: EKS 클러스터의 기존 퍼블릭 노드 그룹 삭제
```
# EKS 클러스터의 노드 그룹 조회
eksctl get nodegroup --cluster=<Cluster-Name>
eksctl get nodegroup --cluster=eksdemo1

# 노드 그룹 삭제 - 노드 그룹 이름과 클러스터 이름을 교체
eksctl delete nodegroup <NodeGroup-Name> --cluster <Cluster-Name>
eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1
```

## 단계-03: 프라이빗 서브넷에 EKS 노드 그룹 생성
- 클러스터에 프라이빗 노드 그룹을 생성합니다.
- 핵심 옵션은 `--node-private-networking` 입니다.

```
eksctl create nodegroup --cluster=eksdemo2 \
                        --region=ap-northeast-2 \
                        --name=eksdemo2-ng-private2 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking                       
```

## 단계-04: 노드 그룹이 프라이빗 서브넷에 생성되었는지 확인

### 워커 노드의 External IP 주소 확인
- 워커 노드가 프라이빗 서브넷에 생성되었다면 External IP는 none이어야 합니다.
```
kubectl get nodes -o wide
```

### 서브넷 라우트 테이블 확인 - 아웃바운드 트래픽이 NAT 게이트웨이를 통과
- 노드 그룹 서브넷 라우트를 확인하여 프라이빗 서브넷에 생성되었는지 검증합니다.
  - Services -> EKS -> eksdemo -> eksdemo1-ng1-private 로 이동
  - **Details** 탭에서 Associated subnet 클릭
  - **Route Table** 탭 클릭
  - NAT 게이트웨이를 통한 인터넷 경로가 보여야 합니다(0.0.0.0/0 -> nat-xxxxxxxx)
