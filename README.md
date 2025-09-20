# Kubernetes(EKS)로 배포한 Petclinic

☁️ Petclinic 애플리케이션을 AWS EKS에 배포하고, ALB Ingress / HPA / Karpenter를 적용한 구성의 아카이브입니다.  
학습 및 실습 과정에서 사용한 매니페스트와 디렉토리 구조를 정리했습니다.

---

## 📝 개요

- EKS(eksctl) 클러스터 구성 – 클러스터 엔드포인트 접근: Public + Private, Managed Node Group
- Pod 자격증명(권한 부여) – ALB Controller = IRSA(OIDC), Karpenter = EKS Pod Identity (podIdentityAssociations)
- ALB Ingress Controller(Helm 별도 설치) + ALB에서 TLS 종료(ACM 인증서), Route 53으로 도메인 라우팅
- HPA(CPU 기반) 적용, Karpenter로 노드 자동 확장
- 이미지: ECR, 네임스페이스: web, was

---

## 🏛️ Architecture
![Architecture](eks-archi.png)

구성 요소:
- **EKS(Managed Node Group)**: 관리형 노드 그룹으로 운영 단순화, 재현성 확보
- **ALB Ingress**: HTTPS 종단 및 L7 라우팅
- **HPA**: web/was 각각 CPU 기반 자동 확장
- **Karpenter**: 급격한 부하 시 Node 레벨 자동 확장
- **ECR/RDS**: 컨테이너 이미지 저장소 / 데이터베이스

---

## <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/kubernetes/kubernetes-original.svg" width="24"/> Kubernetes 구성
```bash
eks-petclinic/
├── cluster/
│   └── eksctl-cluster.yaml         # eksctl로 생성한 클러스터 정의
├── apps/
│   ├── web/
│   │   ├── web-deployment.yaml
│   │   ├── web-service.yaml
│   │   ├── Dockerfile
│   │   └── httpd.conf
│   └── was/
│       ├── was-deployment.yaml
│       ├── was-service.yaml
│       └── Dockerfile
├── ingress/
│   └── ingress.yaml                # ALB Ingress(ACM/도메인은 아카이브 값)
├── hpa/
│   ├── hpa-web.yaml
│   └── hpa-was.yaml
├── karpenter/
│   ├── ec2-nodeclass.yaml
│   └── nodepool.yaml
└── README.md
```
---

### 🗒️ 메모
- **엔드포인트 구성**: Public + Private 병행(운영 편의성과 접근 제어 균형)
- **ALB Controller 설치**: 클러스터 YAML이 아닌 Helm + IRSA로 별도 설치
- **metrics-server**: EKS Add-on(또는 Helm)으로 구성
- **HPA 기준**: resources.requests와 일치하도록 CPU 기반 타깃(예: 60%) 설정
- **이미지 태그**: :latest 지양, v1.0.0-<sha> 등 불변 태그 권장

---

## 📎 참고사항
- 종료된 학습/아카이브용 매니페스트이며, 일부 값(ACM ARN/도메인/ECR 경로 등)은 현재 비활성입니다.
- 구조 참고 및 기록 목적이며, 실제 운영 계정 정보는 포함하지 않습니다.
- 관련 레포
  - Terraform 3-Tier 인프라: (링크)
  - CI/CD: (추가 예정 혹은 범위 밖)
