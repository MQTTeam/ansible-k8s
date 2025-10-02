# Calico 네트워크 설치 및 확인 가이드


---


```bash
🔹 1. Calico 설치 및 설정
1️⃣ Pod CIDR 일치 확인

Kubernetes 클러스터 init 시 지정한 Pod CIDR과 Calico IPPool CIDR이 반드시 같아야 합니다.

kubeadm init --pod-network-cidr=192.168.0.0/16
kubectl get ippool -n kube-system
kubectl describe ippool default-ipv4-ippool -n kube-system


출력 예시:

Spec:
  Cidr: 192.168.0.0/16
  Ipip Mode: Always
  Vxlan Mode: Never

2️⃣ Calico Pod 상태 확인
kubectl get pods -n kube-system -o wide | grep calico


calico-kube-controllers → Running

각 노드마다 calico-node-xxxxx → Running

3️⃣ Pod 통신/DNS 확인
# Pod IP 확인
kubectl get pods -o wide

# 워커1 Pod → 워커2 Pod ping
kubectl exec -it <pod1> -- ping -c 3 <pod2-IP>

# CoreDNS 조회
kubectl run tmp-dns --rm -it --image=busybox:1.28 -- nslookup kubernetes.default


Pod 간 통신 가능

kubernetes.default 가 10.96.0.1 로 조회되면 정상

🔹 2. iptables 기본 세팅 (모든 노드)

Calico는 내부적으로 iptables를 사용하기 때문에, 커널 및 방화벽 설정이 필요합니다.

1️⃣ 커널 모듈 및 sysctl
modprobe br_netfilter

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sysctl --system


✅ 확인:

sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward


둘 다 =1 이면 정상.

2️⃣ 필수 포트 오픈

📌 마스터 노드

6443 (API 서버)

2379-2380 (etcd)

10250 (kubelet)

10257/10259 (컨트롤러/스케줄러)

📌 워커 노드

10250 (kubelet)

30000-32767 (NodePort 범위)

✅ 확인:

iptables -L -n -v
iptables -t nat -L -n -v | grep KUBE


→ KUBE-SERVICES, KUBE-NODEPORTS 체인이 존재해야 함.

🔹 3. CoreDNS 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl run tmp-dns --rm -it --image=busybox:1.28 -- nslookup kubernetes.default


👉 ClusterIP (10.96.0.1) 정상 조회 시 OK.

🔹 4. NetworkPolicy 확인
적용된 정책
kubectl get networkpolicy -A


현재 정책:

allow-backend-to-db

대상: app=pg 라벨이 붙은 PostgreSQL Pod

의도: backend Pod → DB(5432) 접근 허용

allow-db-access

대상: 마찬가지로 app=pg

의도: 특정 namespace 또는 조건에서 DB 접근 허용

실제 확인
kubectl get networkpolicy allow-backend-to-db -o yaml
kubectl get networkpolicy allow-db-access -o yaml

✅ 최종 정상 상태

kubectl get nodes → 모든 노드 Ready

kubectl get pods -A → Calico, CoreDNS, backend, db, redis Running

kubectl get svc → backend-service, pg, redis, emqx 등 서비스 정상

kubectl get endpoints → 서비스에 Pod IP 바인딩 확인

kubectl exec tmp-dns -- nslookup pg → DB DNS 조회 성공

kubectl exec tmp-curl -- curl backend-service:3000/api/health → 헬스체크 성공

NetworkPolicy에 따라 허용된 트래픽만 통과됨

---
