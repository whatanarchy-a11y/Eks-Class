# AWS - 네트워크 로드 밸런서 - NLB

## 단계-01: AWS 네트워크 로드 밸런서 Kubernetes 매니페스트 생성 및 배포
- **04-NetworkLoadBalancer.yml**
```yml
apiVersion: v1
kind: Service
metadata:
  name: nlb-usermgmt-restapp
  labels:
    app: usermgmt-restapp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb    # 네트워크 로드 밸런서 생성
spec:
  type: LoadBalancer # 일반적인 k8s Service 매니페스트에서 type을 LoadBalancer로 설정
  selector:
    app: usermgmt-restapp     
  ports:
  - port: 80
    targetPort: 8095
```
- **모든 매니페스트 배포**
```
# 모든 매니페스트 배포
kubectl apply -f kube-manifests/

# 서비스 목록 조회 (새로 생성된 NLB 서비스 확인)
kubectl get svc

# 파드 확인
kubectl get pods
```

## 단계-02: 배포 확인
- 새로운 NLB가 생성되었는지 확인
  - Services -> EC2 -> Load Balancing -> Load Balancers 로 이동
    - NLB가 생성되어 있어야 함
    - DNS 이름 복사 (예: a85ae6e4030aa4513bd200f08f1eb9cc-7f13b3acc1bcaaa2.elb.us-east-1.amazonaws.com)
  - Services -> EC2 -> Load Balancing -> Target Groups 로 이동
    - 헬스 상태를 확인하고 active 상태인지 확인
- **애플리케이션 접속**
```
# 애플리케이션 접속
http://<NLB-DNS-NAME>/usermgmt/health-status
```    

## 단계-03: 정리
```
# 생성된 모든 오브젝트 삭제
kubectl delete -f kube-manifests/

# 현재 Kubernetes 오브젝트 확인
kubectl get all
```
