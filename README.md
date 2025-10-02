# Calico 네트워크 설치 및 확인 가이드

본 문서는 Kubernetes 클러스터에서 **Calico CNI** 를 적용한 뒤 정상적으로 동작하는지 확인하는 절차를 정리한 가이드입니다.  

---

## 1. Calico 설치 확인
```bash
kubectl -n kube-system get pods -o wide | grep calico

1️⃣ 커널 및 iptables 기본 세팅 (모든 노드: 마스터 + 워커)

Calico와 쿠버네티스 네트워크는 브리지 네트워크를 iptables로 통과시켜야 합니다.

# 커널 모듈
modprobe br_netfilter

# sysctl 적용
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sysctl --system


✅ 확인:

sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward


둘 다 =1 이 나와야 정상.

2️⃣ iptables 포트 확인 (마스터/워커 구분)

📌 마스터 노드:

6443 (API 서버), 2379-2380 (etcd), 10250 (kubelet), 10257/10259 (컨트롤러/스케줄러)

📌 워커 노드:

10250 (kubelet), 30000-32767 (NodePort 범위)

✅ 확인:

iptables -L -n -v
iptables -t nat -L -n -v | grep KUBE


👉 KUBE-SERVICES, KUBE-NODEPORTS 체인 규칙이 자동 생성되어 있어야 함.

3️⃣ Calico CNI 정상 동작 확인
kubectl get pods -n kube-system -o wide | grep calico


예상 결과:

calico-kube-controllers → Running

calico-node-xxxxx → 각 노드마다 1개씩 Running

kubectl get ippool -n kube-system


CIDR이 클러스터에 할당한 Pod 대역(192.168.0.0/16)과 일치해야 함.

✅ Pod IP 확인:

kubectl get pods -o wide


👉 Pod IP가 192.168.x.x 대역에서 고르게 분배되는지 확인.

4️⃣ CoreDNS 정상 동작 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl run tmp-dns --rm -it --image=busybox:1.28 -- sh
/ # nslookup kubernetes.default


👉 ClusterIP (10.96.0.1)가 정상적으로 조회되면 OK.

5️⃣ 네트워크폴리시(NetworkPolicy) 확인
kubectl get networkpolicy -A


예:

allow-backend-to-db: backend → Postgres 접근 허용

allow-db-access: 특정 namespace 또는 selector에 대해 db 접근 허용

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
