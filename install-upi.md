# OpenShift UPI 설치 요약 (2025)

> **UPI(User-Provisioned Infrastructure)**: 사용자가 인프라(DNS, DHCP, LB, 노드 등)를 직접 준비하고, 설치 프로그램은 `install-config.yaml` → manifests → ignition 파일 생성까지만 수행하는 방식.

## 0. 배경 – UPI vs IPI
- **UPI**: 인프라 구성 책임은 사용자. DNS(A/PTR), API/Ingress LB, 방화벽 포트 개방(예: 6443, 22623, 80/443) 등 직접 준비.  
- **IPI**: 설치기가 클라우드/하이퍼바이저 자원까지 프로비저닝.

## 1. 사전 준비 체크리스트
- **도구**: `openshift-install`, `oc` (해당 버전과 일치 필요).  
- **네트워크**:
  - DNS: `api.<cluster>.<baseDomain>`, `api-int.<...>`, `*.apps.<...>`, `bootstrap`, 각 노드의 A/PTR.
  - LB: API(6443), Machine Config Server(22623), Ingress(80/443).
  - 방화벽: 클러스터 컴포넌트 간 통신 포트 허용.
- **이미지/부팅**: RHCOS/FCOS 부트 이미지(ISO/PXE), ignition 파일을 제공할 웹/HTTP/TFTP 등.

## 2. 표준 설치 흐름(요지)
1) **설치 도구 획득**  
   - `openshift-install` & `oc` 바이너리 다운로드/설치.
2) **설정 파일 생성**  
   - `install-config.yaml` 작성 → `openshift-install create manifests`  
   - 필요 시 매니페스트 커스터마이즈 → `openshift-install create ignition-configs`
3) **인프라/노드 준비**  
   - vSphere/베어메탈에 **bootstrap / control-plane(3) / worker(≥2)** VM(또는 베어메탈) 준비.  
   - 각 노드가 대응 ignition을 부팅 시점에 참조하도록 설정(ISO/LU/Netboot + HTTP/TFTP/S3 등).
4) **DNS/LB 검증**  
   - `dig`로 `api`, `api-int`, `*.apps`, `bootstrap`, 각 노드 A/PTR이 올바르게 해석되는지 점검.  
  
<!-- minor formatting tweak -->
 - LB 뒤 IP가 기대값과 일치하는지 확인.
5) **부팅 및 완료 처리**  
   - bootstrap → masters → workers 순으로 네트부팅/ISO부팅.  
   - bootstrap 완료 후 bootstrap 노드 정리.  
   - 필요 시 CSR 승인으로 워커 조인 확인.  
   - `oc get nodes`, `oc get co`로 상태 점검.

## 3. vSphere 메모(예시 값)
- 네트워크: 고정 IP 또는 DHCP+예약.  
- 스토리지: 각 노드 최소 스펙(예: 4vCPU/16GB/120GB 디스크 for control-plane; 워커는 워크로드에 따라).  
- 부팅: ISO 또는 PXE, 커널 파라미터로 ignition endpoint 전달.

## 4. install-config.yaml 최소 예시 (vSphere)
> 실제 값으로 치환 필요. 클러스터/도메인/네트워크/CIDR/리소스풀/데이터스토어/폴더/템플릿명 등 맞추기.
```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp4
platform:
  vsphere:
    vcenter: vcenter.example.com
    username: "administrator@vsphere.local"
    password: "<REDACTED>"
    datacenter: "DC1"
    defaultDatastore: "vsanDatastore"
    cluster: "Compute"
    network: "VM Network"
    folder: "/DC1/vm/ocp4"
pullSecret: '<json>'
sshKey: 'ssh-ed25519 AAAA... user@host'
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
controlPlane:
  name: master
  replicas: 3
  platform: { vsphere: {} }
compute:
  - name: worker
    replicas: 2
    platform: { vsphere: {} }
