# Calico 네트워크 설치 및 확인 가이드

본 문서는 Kubernetes 클러스터에서 **Calico CNI** 를 적용한 뒤 정상적으로 동작하는지 확인하는 절차를 정리한 가이드입니다.  

---

## 1. Calico 설치 확인
```bash
kubectl -n kube-system get pods -o wide | grep calico

2. IPPool 확인
kubectl get ippool -n kube-system -o yaml


Pod 네트워크 CIDR (192.168.0.0/16 등) 이 원하는 값으로 설정됐는지 확인합니다.

3. Node CIDR 확인
kubectl get nodes -o yaml | grep podCIDR


각 노드에 /24 대역이 자동 할당됐는지 확인합니다.
(예: 192.168.0.0/24, 192.168.1.0/24, 192.168.2.0/24)

4. Pod 간 통신 확인
kubectl run tmp1 --rm -it --image=busybox:1.28 -- sh
/ # ping <다른 Pod IP>


✅ 다른 노드에 있는 Pod IP 도 ping 이 통하면 정상입니다.

5. CoreDNS 확인
kubectl run tmp-dns --rm -it --image=busybox:1.28 -- sh
/ # nslookup kubernetes.default


✅ 10.96.0.1 로 응답이 오면 정상입니다.

6. 서비스 접근 확인
kubectl run tmp-curl --rm -it --image=appropriate/curl -- sh
/ # curl http://<service-name>:<port>


✅ ClusterIP/NodePort Service 에 정상적으로 접근되는지 확인합니다.

7. (선택) BGP 상태 확인
kubectl -n kube-system exec -it <calico-node-pod> -- calicoctl node status


✅ BGP 세션이 Established 상태여야 합니다.

🔎 현재 설정된 Network Policy 정리


1️⃣ allow-backend-to-db

대상 Pod: app=pg 라벨이 있는 Postgres Pod

정책 의도: backend Pod에서 오는 트래픽만 DB(5432) 접근 허용 (추측)

실제 내용 확인:

kubectl get networkpolicy allow-backend-to-db -o yaml

2️⃣ allow-db-access

대상 Pod: 마찬가지로 app=pg

정책 의도: 이름상 "DB 접근 허용" → 아마 특정 Namespace 또는 특정 IPBlock에서의 접근 허용

실제 내용 확인:

kubectl get networkpolicy allow-db-access -o yaml

---
