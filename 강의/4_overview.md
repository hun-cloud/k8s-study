# Kubernetes 핵심 개념 간단 정리

## Master (Control Plane)
클러스터 전체를 관리하는 두뇌 역할. 주요 컴포넌트는 4가지.
- Kube-apiserver: 모든 요청의 진입점. kubectl 명령이든, 내부 컴포넌트 간 통신이든 전부 API Server를 거친다.
- etcd : 클러스터의 모든 상태 (어떤 pod가 어디 떠 있는지, 설정값 등)를 저장하는 key-value 저장소. 클러스터의 단일 진실 공급원(sorce of truth) 이다.
- kube-scheduler : 새로 생성된 pod를 어느 Node에 배치할지 결정한다. 리소스 여유, affinity 규칙 등을 고려한다.
- Kube-controller-manager: 여러 컨트롤러를 실행하며, "원하는 상태(desired state)"와 "현재 상태(current state)"를 계속 비교해서 맞춘다.

## Node (Worker Node)
실제 컨테이너가 실행되는 서버다. 각 Node에는 다음 컴포넌트가 동작한다.
- kubelet : Master의 지시를 받아 Pod를 생성/관리하고, 상태를 API Server에 보고.
- Kube-proxy : Service로 들어온 트래픽을 실제 Pod로 라우팅하는 네트워크 규칙 (iptables/IPVS)을 관리한다.
- Container Runtime : containerd 같은 런타임이 실제로 컨테이너를 실행한다.

## Kubernetes Cluster
Master + Node들의 집합이 하나의 Cluster다. 개별 서버를 신경 쓰지 않고 "이 앱을 3개 띄워죠"라고 선언만 하면, 클러스터가 알아서 배치하고 유지한다. 이 선언형(declarative) 모델이 쿠버네티스의 핵심 철학이다.

### Namespace
클러스터 안을 논리적으로 나누는 가상의 공간이다. 예를 들어 dev, staging, prod 네임스페이스를 나눠서 같은 클러스터 안에서도 환경을 분리할 수 있다. 리소스 이름은 네임스페이스 안에서만 유일하면 되고, ResourceQuota로 네임스페이스별 자원 제한도 걸 수 있다.

### Pod 
최소 배포 단위. 컨테이너를 직접 배포하는게 아니라 항상 pod로 감싸서 배포한다. 하나의 pod 안에는 보통 컨테이너 1개가 들어가지만, 로그 수집기 같은 보조 컨테이너를 함께 넣는 사이트카 패턴도 있다.

같은 pod 안의 컨테이너들은 IP와 볼륨을 공유하며 localhost로 서로 통신할 수 있다. Pod는 언제든 죽고 다시 생성될 수 있는 일회성 존재라서 IP가 계속 바뀌는데, 이 문제를 해결하는 것이 service다.

### Service
pod들에 접근하기 위한 고정된 진입점이다. Pod IP는 계속 바뀌지만 Service의 IP와 DNS 이름은 고정되어 있고, label selector로 대상 pod들을 찾아 트래픽을 분산한다.

- ClusterIP : 클러스터 내부에서만 접근 가능 (기본값)
- NodePort : 각 Node의 특정 포트를 열어 외부 접근 허용
- LoadBlancer : 클라우드의 로드밸런서를 붙여 외부 노출

### Container
애플리케이션과 그 실행환경을 하나로 패키징한 것. 쿠버네티스는 컨테이너를 직접 다루지 않고 Pod 단위로 관리한다는 점이 Docker 단독 사용과의 차이다.

### Volume
컨테이너의 파일시스템은 컨테이너가 죽으면 사라진다. Volume은 이 데이터를 유지하기 위한 저장소다.

- emptyDir : Pod가 살아있는 동안만 유지 (Pod 내 컨테이너 간 공유용)
- hostPath : Node의 디렉토리를 마운트
- Pv/Pvg : PersistentVolume은 실제 스토리지, PersistentVolumeClaim은 "이만큼 스토리지를 달라"는 요청이다. 개발자는 Pvc만 작성하면 되고 실제 스토리지가 NFS인지 클라우드 디스크인지 몰라도 되는, 스토리지 추상화 구조.

### ResourceQuota / LimitRange

- ResourceQuota: 네임스페이스 전체의 자원 총량을 제한한다. 예: "dev 네임스페이스는 CPU 총 10코어, Pod 최대 50개까지."
- LimitRange: 개별 Pod/컨테이너의 자원 최소/최대/기본값을 정한다. 개발자가 requests/limits를 지정하지 않았을 때 기본값을 넣어주는 역할도 한다.

둘을 함께 쓰면 특정 팀이 클러스터 자원을 독식하는 것을 막을 수 있다.

### Config Map / Secret
설정을 이미지에서 분리해 외부화하는 오브젝트다.

- ConfigMap: 일반 설정값 (DB 호스트명, 로그 레벨 등)
- Secret: 민감 정보 (비밀번호, 인증서, API 키). base64로 인코딩되어 저장되는데, 인코딩일 뿐 암호화가 아니라서 etcd 암호화나 외부 Vault 연동을 함께 고려해야 한다.

둘 다 환경변수 또는 파일 마운트 방식으로 Pod에 주입할 수 있다.

## Controller

"원하는 상태를 선언하면, 현재 상태를 감시하다가 차이가 생기면 스스로 조정하는" 컨트롤 루프다. 쿠버네티스 자동화의 핵심이다.

### Replication Controller, ReplicaSet

"Pod를 N개 유지하라"는 선언을 지키는 컨트롤러다. Pod가 죽으면 새로 만들고, 많으면 줄인다. Replication Controller는 구세대이고, label selector가 더 유연한 ReplicaSet이 이를 대체했다. 다만 ReplicaSet을 직접 쓰는 경우는 거의 없고, 보통 Deployment를 통해 간접적으로 사용한다.

### Deployment

ReplicaSet을 관리하는 상위 컨트롤러로, 실무에서 가장 많이 쓰인다. 핵심 가치는 무중단 배포다. 새 버전 배포 시 새 ReplicaSet을 만들어 Pod를 점진적으로 교체(Rolling Update)하고, 문제가 생기면 이전 ReplicaSet으로 즉시 롤백할 수 있다. Blue-Green이나 Canary 같은 배포 전략도 Deployment(+Service/Ingress 조작)를 기반으로 구현된다.

### DaemonSet

모든 Node에 Pod를 하나씩 배치하는 컨트롤러다. Node가 추가되면 자동으로 그 Node에도 Pod가 생긴다. 로그 수집기(Fluentd), 모니터링 에이전트(Node Exporter), 네트워크 플러그인처럼 노드마다 반드시 하나씩 있어야 하는 워크로드에 사용한다.

### CronJob

리눅스 crontab처럼 정해진 스케줄에 따라 Job을 실행하는 컨트롤러다. Job은 "완료되면 끝나는" 일회성 작업(배치, 백업 등)을 위한 오브젝트이고, CronJob이 이를 0 2 * * * 같은 cron 표현식으로 주기 실행한다. DB 백업, 정산 배치, 오래된 데이터 정리 같은 작업에 쓴다.


## 정리

Cluster는 Master(관제)와 Node(실행)로 구성되고, 애플리케이션은 Pod라는 최소 단위로 배포되며, Service가 고정 진입점을 제공하고, Controller가 선언된 상태를 자동으로 유지하며, ConfigMap/Secret/Volume이 설정과 데이터를 분리해 관리한다. 이 구조가 쿠버네티스의 뼈대다.