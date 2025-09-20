# EKS Petclinic (EKS + HPA + Karpenter)

**Scope**  
본 레포는 팀 프로젝트에서 제가 직접 수행한 **EKS 배포/운영 파트**를 아카이브한 것입니다.  
**ALB Ingress / HPA / Karpenter**를 적용한 상태를 중심으로 정리했습니다.  
> 참고: CI/CD(GitHub Actions/ArgoCD), Secrets Manager/CSI, Monitoring/DR은 **범위 밖**으로 제외했습니다.

---

## 1) Architecture

![arch](./arch-eks-petclinic.png)

- 트래픽: **User → ALB(HTTPS) → `web`(Apache RP) → `was`(Tomcat/Spring Petclinic) → RDS**
- 네임스페이스: `web`, `was`
- 스케일링: **Pod = HPA**, **Node = Karpenter(선택 적용)**
- 이미지는 **ECR** 사용

---

## 2) 구성 요소(What)

- **Cluster**: [`cluster/eksctl-cluster.yaml`](cluster/eksctl-cluster.yaml)  
  - Endpoint: **Public + Private 병행**  
  - **Managed Node Group** + **OIDC**  
  - 기본 애드온(vpc-cni, kube-proxy, coredns, ebs-csi 등) 구성
- **Ingress**: [`ingress/ingress.yaml`](ingress/ingress.yaml)  
  - ALB Ingress Controller 전제(Helm 별도 설치)  
  - ACM/도메인 값은 현재 비활성(아카이브 목적)
- **Workloads**: `apps/web/*`, `apps/was/*`  
  - `web`: Apache Reverse Proxy (Dockerfile + httpd.conf)  
  - `was`: Tomcat 이미지에 Petclinic 배포 (Dockerfile)
- **Autoscaling (Pod)**: [`hpa/hpa-web.yaml`](hpa/hpa-web.yaml), [`hpa/hpa-was.yaml`](hpa/hpa-was.yaml)  
  - CPU 기반, `resources.requests`와 정합 유지
- **Autoscaling (Node)**: [`karpenter/*`](karpenter)  
  - `EC2NodeClass` + `NodePool`로 스파이크 대응/비용 최적화

---

## 3) 설계 결정(Why)

- **Endpoint Public+Private**: 운영 편의(퍼블릭)와 내부 경로 보호(프라이빗)의 균형  
- **Managed Node Group**: 관리 단순화/재현성  
- **ALB Ingress + ACM**: 표준 HTTPS 종단  
- **HPA 기준(예: CPU 60%)**: `requests.cpu`와 일치하게 설계 → 예측 가능한 스케일아웃  
- **Karpenter**: 급격한 부하/비용 최적화 대응(선택 적용)

---

## 4) 레포 맵

- `cluster/eksctl-cluster.yaml` : 클러스터 생성 정의  
- `apps/web/*`, `apps/was/*` : 워크로드 매니페스트 + Dockerfile/httpd.conf  
- `ingress/ingress.yaml` : ALB Ingress (ACM/host 값은 비활성)  
- `hpa/*` : HPA 정의(web/was)  
- `karpenter/*` : EC2NodeClass, NodePool

---

## 5) 참고 커맨드(기록용 – 재현 목적 아님)

```bash
# (A) 클러스터 생성
eksctl create cluster -f cluster/eksctl-cluster.yaml

# (B) AWS Load Balancer Controller (별도 설치: Helm + IRSA)
eksctl create iamserviceaccount \
  --cluster <CLUSTER_NAME> \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn <ALBC_POLICY_ARN> \
  --approve
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set region=ap-northeast-2 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# (C) 워크로드/인그레스/HPA/카펜터 적용
kubectl apply -f apps/web/
kubectl apply -f apps/was/
kubectl apply -f ingress/ingress.yaml
kubectl apply -f hpa/
kubectl apply -f karpenter/

# (D) 확인
kubectl get ingress -n web
kubectl get hpa -A
kubectl get nodes -o wide
