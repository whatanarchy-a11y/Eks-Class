# EKS 노드그룹 삭제 중 `unevictable pods` 발생 시 정리 (eksctl / kubectl)

아래 메시지는 `eksctl delete nodegroup ...`가 내부적으로 **드레인(drain)** 을 수행하는 과정에서, 특정 노드에 있는 일부 Pod가 **eviction(퇴거)** 으로는 빠질 수 없어(= unevictable) 노드 삭제가 막히고 있다는 뜻입니다.

- 로그 예시  
  `2026-02-03 17:27:44 [!]  9 pods are unevictable from node ip-192-168-0-101.ap-northeast-2.compute.internal`

---

## 1) 용어 정리: cordon vs drain vs eviction

### 1) cordon
- 노드를 **SchedulingDisabled(스케줄 불가)** 상태로 만들어 **새 Pod가 더 이상 배치되지 않게** 합니다.
- 기존 Pod는 그대로 남아있습니다.

### 2) drain
- 보통 다음 두 단계를 포함합니다.
  1) **cordon**
  2) 해당 노드의 Pod를 **evict(퇴거)** 해서 다른 노드로 재스케줄되게 함  
     (ReplicaSet/Deployment면 새 노드에 새 Pod가 뜨고, 기존 Pod는 종료)

### 3) eviction(퇴거)
- 쿠버네티스가 “안전하게 Pod를 내보내기” 위해 사용하는 메커니즘입니다.
- **PDB(PodDisruptionBudget)** 같은 정책을 존중하며, 조건이 맞지 않으면 거부됩니다.

---

## 2) “nodegroup 1개 삭제”인데 노드가 여러 개 cordon 되는 이유
`eksctl delete nodegroup`는 “노드그룹”을 대상으로 하며, 노드그룹에 속한 **모든 노드**를 cordon/drain 합니다.

따라서 로그에 노드가 2개 이상 나온다면 보통 아래 중 하나입니다.
- 해당 노드그룹의 EC2 노드가 원래 2대 이상(ASG desired/min/max 영향)
- 스케일/교체(rolling/refresh) 이벤트로 잠시 노드가 늘어난 상태
- 라벨/태그 기준으로 해당 노드그룹 소속으로 인식된 노드가 여러 대

---

## 3) 9 pods가 `unevictable`로 뜨는 대표 원인 5가지

### A. PDB(PodDisruptionBudget) 때문에 eviction 거부 (가장 흔함)
- “지금 이 Pod를 내보내면 가용성(minAvailable/maxUnavailable) 조건이 깨짐” → eviction 실패

**확인**
```bash
kubectl get pdb -A
kubectl describe pdb -n <ns> <pdb-name>
```

**해결(권장 순서)**
- 워크로드 replicas를 늘려서 `DisruptionsAllowed`가 1 이상 되게 만들기
- PDB를 일시적으로 완화(예: `minAvailable` 낮추거나 `maxUnavailable` 올림)
- 정말 임시로 필요하면 PDB를 잠시 삭제 후 drain (주의 필요)

---

### B. 컨트롤러 없는 Pod(unmanaged pod)이라 기본 drain이 못 지움
예: `kubectl run ... --restart=Never`로 만든 단독 Pod, ownerReferences 없는 Pod

**확인(ownerReferences)**
```bash
kubectl get pod -n <ns> <pod> -o jsonpath='{.metadata.ownerReferences[*].kind}{"\n"}'
```

**해결**
- 해당 Pod를 수동 삭제
- 또는 강제 drain 옵션(`--force`) 사용 (아래 6번 참고)

---

### C. 로컬 데이터(emptyDir 등) 때문에 삭제 경고로 차단
드레인은 노드 로컬 데이터 유실 가능성이 있으면 옵션에 따라 차단됩니다.

**확인(볼륨)**
```bash
kubectl get pod -n <ns> <pod> -o jsonpath='{.spec.volumes[*].emptyDir}{"\n"}'
```

**해결**
- `--delete-emptydir-data` 옵션을 포함하여 drain 수행

---

### D. 옮길 곳이 없음 (리소스 부족/affinity/taint/selector 제약)
eviction 자체는 가능해도, 새 노드에 스케줄이 안 되면 결과적으로 drain이 끝나지 않습니다.

**확인**
```bash
kubectl get pods -A | egrep "Pending|ContainerCreating"
kubectl describe pod -n <ns> <pod>   # Events에 스케줄 실패 이유 표시
```

**해결**
- 노드그룹 desired 용량을 잠시 늘리기
- 임시 노드그룹 추가
- affinity/selector/taint/toleration 조건 조정

---

### E. 종료가 오래 걸리는 Pod (terminationGracePeriod/finalizer 등)
evict는 됐는데 Pod가 Terminating에서 오래 버티면 drain이 지연됩니다.

**확인**
```bash
kubectl get pod -n <ns> <pod> -w
kubectl describe pod -n <ns> <pod>
```

---

## 4) 먼저 해야 할 일: “그 9개 Pod가 누구인지” 확인

```bash
NODE=ip-192-168-0-101.ap-northeast-2.compute.internal
kubectl get pods -A -o wide --field-selector spec.nodeName=$NODE
```

- `kube-system` Pod가 다수라면(특히 단일 노드에만 존재하는 형태) 원인 후보가 빠르게 좁혀집니다.

---

## 5) unevictable Pod를 빠르게 분류(Owner Kind 확인)

```bash
NODE=ip-192-168-0-101.ap-northeast-2.compute.internal

kubectl get pods -A -o jsonpath='{range .items[?(@.spec.nodeName=="'"$NODE"'")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.ownerReferences[0].kind}{"\n"}{end}'
```

- owner kind가 **비어있으면** → unmanaged Pod 가능성 큼  
- StatefulSet/Deployment인데도 안 빠지면 → PDB/스케줄링 제약 가능성 큼

---

## 6) (필요 시) 수동 drain로 “정확한 실패 이유” 출력 보기

`eksctl`은 요약 로그만 보여줄 때가 있어, 직접 drain을 걸면 어떤 Pod가 왜 거부되는지 더 자세히 볼 수 있습니다.

```bash
NODE=ip-192-168-0-101.ap-northeast-2.compute.internal
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=30 \
  --timeout=10m
```

> ⚠️ 주의  
> - `--force`, `--delete-emptydir-data`는 안전장치 해제 성격이 있어, **먼저 PDB/리소스 부족 같은 근본 원인을 확인**한 뒤 “삭제해도 되는 Pod”에만 적용하는 게 좋습니다.

---

## 7) 문제 원인을 바로 찍기 위한 최소 출력(붙여넣기용)

아래 두 결과만 있으면 “9개가 왜 unevictable인지”를 케이스별로 정확히 분류할 수 있습니다.

```bash
NODE=ip-192-168-0-101.ap-northeast-2.compute.internal
kubectl get pods -A -o wide --field-selector spec.nodeName=$NODE
kubectl get pdb -A
```

---
