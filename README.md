# Kubernetes 프로젝트 요소기술 분석

> 분석 대상: `/home/c1_master1/kubernetes`
> 프로젝트 규모: 약 22,800개 파일, 1.6GB (vendor 포함)

---

## 1. 프로젝트 개요

| 항목 | 내용 | 근거 파일 |
|------|------|-----------|
| 프로젝트명 | Kubernetes (K8s) | `README.md` |
| 설명 | 컨테이너화된 애플리케이션의 배포, 관리, 스케일링을 자동화하는 오픈소스 컨테이너 오케스트레이션 시스템임 | `README.md` |
| 주요 언어 | Go (약 16,700개 `.go` 파일) | `go.mod` |
| Go 버전 | **1.25.7** (`.go-version` 파일에 명시됨) | `.go-version` |
| 모듈 경로 | `k8s.io/kubernetes` | `go.mod` 1행 |
| 라이선스 | Apache License 2.0 | `LICENSE` |
| 빌드 시스템 | Make → Bash → Docker 컨테이너 내 Go 컴파일 | `Makefile`, `build/common.sh` |
| 워크스페이스 | Go Workspaces (`go.work`)로 33개 staging 모듈 관리 | `go.work` |

---

## 2. 핵심 요소기술 목록

### 2.1 프로그래밍 언어 및 런타임

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **Go** | 1.25.7 | `.go-version` | 전체 Kubernetes 컴포넌트(apiserver, kubelet, scheduler 등)의 핵심 개발 언어임. `cmd/` 디렉토리의 각 바이너리 진입점이 `main()` 함수를 가진 Go 파일임 |
| **Bash/Shell** | - | `hack/`, `build/` | 빌드 자동화(`build/common.sh`의 19개 함수), 코드 검증(`hack/verify-*.sh` 50개 이상), 코드 생성(`hack/update-*.sh` 30개 이상) 스크립트임. 총 308개 쉘 스크립트가 존재함 |
| **Protocol Buffers** | v1.36.11 | `go.mod` → `google.golang.org/protobuf` | API 객체의 고성능 직렬화/역직렬화에 사용됨. `hack/update-codegen.sh`의 `codegen::protobuf()` 함수(115행)에서 protobuf 코드를 자동 생성함 |

### 2.2 컨테이너 및 런타임 기술

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **Docker** | - | `build/common.sh` | `build/run.sh`가 Docker 빌드 컨테이너(`kube-cross` 이미지)를 띄워서 그 안에서 Go 컴파일을 수행함. `kube::build::run_build_command()` 함수(357행)가 이 역할을 담당함. `build/release-images.sh`가 각 컴포넌트의 Docker 이미지를 생성함 |
| **Docker Buildx** | - | `build/lib/release.sh` | `kube::release::create_docker_images_for_server()` 함수에서 `docker buildx build --platform linux/${arch}` 명령으로 멀티 아키텍처 이미지를 빌드함 |
| **containerd** | v1.10.0 | `go.mod` → `github.com/containerd/containerd` | kubelet이 CRI(Container Runtime Interface)를 통해 containerd와 gRPC로 통신하여 실제 컨테이너를 생성/삭제함. `pkg/kubelet/` 내 46개 서브패키지에서 사용됨 |
| **cgroups** | v3 | `go.mod` → `github.com/containerd/cgroups/v3` | kubelet의 컨테이너 매니저(`pkg/kubelet/cm/`)가 Linux cgroups를 통해 CPU, 메모리 등의 리소스를 격리하고 제한하는 기능을 함 |
| **SELinux** | - | `go.mod` → `github.com/opencontainers/selinux` | 파드의 보안 컨텍스트(`pkg/securitycontext/`)에서 SELinux 라벨을 적용하여 컨테이너 간 접근을 제어하는 기능을 함 |

### 2.3 데이터 저장 및 통신

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **etcd** | v3.6.7 | `go.mod` → `go.etcd.io/etcd` | **유일한 클러스터 상태 저장소**임. kube-apiserver만 etcd에 직접 접근하며, `/registry/{리소스타입}/{네임스페이스}/{이름}` 형식의 키로 모든 리소스를 저장함. `staging/src/k8s.io/apiserver/pkg/storage/` 패키지가 etcd 접근 추상화 계층을 제공함 |
| **bbolt** | - | `go.mod` → `go.etcd.io/bbolt` | etcd의 내부 저장 엔진으로 사용되는 임베디드 키-값 데이터베이스임 |
| **gRPC** | v1.78.0 | `go.mod` → `google.golang.org/grpc` | kubelet↔containerd 간 CRI 통신, apiserver↔etcd 간 통신에 사용되는 고성능 RPC 프레임워크임. `staging/src/k8s.io/cri-api/`에서 CRI의 gRPC 서비스 정의를 관리함 |
| **gorilla/websocket** | - | `go.mod` → `github.com/gorilla/websocket` | `kubectl exec`, `kubectl logs -f` 등의 실시간 스트리밍 명령에서 apiserver와 WebSocket으로 양방향 통신하는 기능을 함 |

### 2.4 API 및 인터페이스

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **OpenAPI v3** | - | `api/openapi-spec/` | kube-apiserver가 노출하는 REST API의 명세를 정의함. `hack/update-openapi-spec.sh`가 소스 코드에서 OpenAPI 스펙을 자동 생성하고, `hack/verify-openapi-spec.sh`가 유효성을 검증함 |
| **gnostic** | - | `go.mod` → `github.com/google/gnostic` | OpenAPI 명세를 파싱하고 처리하는 라이브러리임. kubectl이 API 서버의 OpenAPI 스펙을 다운로드하여 클라이언트 사이드 유효성 검증에 사용함 |
| **cobra** | v1.10.0 | `go.mod` → `github.com/spf13/cobra` | kubectl, kubeadm, kube-apiserver 등 모든 CLI 바이너리의 명령줄 인터페이스 프레임워크임. `cmd/` 디렉토리의 각 바이너리가 cobra를 사용하여 서브커맨드, 플래그, 도움말을 구현함 |
| **pflag** | - | `go.mod` → `github.com/spf13/pflag` | cobra와 함께 사용되는 POSIX 호환 커맨드라인 플래그 파서임 |
| **CEL** | - | `go.mod` → `github.com/google/cel-go` | Kubernetes 1.25+에서 API 유효성 검증 규칙을 CEL 표현식으로 정의할 수 있게 하는 언어임. `ValidatingAdmissionPolicy`에서 사용됨 |

### 2.5 모니터링 및 관측성 (Observability)

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **Prometheus client** | v1.23.2 | `go.mod` → `github.com/prometheus/client_golang` | 각 컴포넌트가 `/metrics` HTTP 엔드포인트를 노출하여 Prometheus 서버가 메트릭을 수집(scrape)할 수 있게 함. 예: apiserver의 요청 지연시간, kubelet의 파드 수 등 |
| **OpenTelemetry** | v1.40.0 | `go.mod` → `go.opentelemetry.io/otel` | 분산 추적(Distributed Tracing) 기능을 제공함. API 요청이 apiserver → scheduler → kubelet으로 전달되는 과정을 스팬(span)으로 추적할 수 있게 함 |
| **zap** | v1.27.1 | `go.mod` → `go.uber.org/zap` | 고성능 구조화 로깅 백엔드임. JSON 형식의 구조화된 로그를 생성하여 로그 수집기(fluentd 등)에서 파싱하기 쉽게 함 |
| **logr** | v1.4.3 | `go.mod` → `github.com/go-logr/logr` | Go의 로깅 추상화 인터페이스임. 실제 로깅 구현(zap, klog)을 교체할 수 있게 하는 어댑터 역할을 함 |

### 2.6 테스팅 프레임워크

| 기술 | 버전 | 근거 | 구체적 역할 |
|------|------|------|-------------|
| **Ginkgo** | v2.27.4 | `go.mod` → `github.com/onsi/ginkgo/v2` | E2E 테스트(`test/e2e/`)에서 사용하는 BDD(행위 주도 개발) 프레임워크임. `Describe`, `It`, `BeforeEach` 같은 구조로 테스트 시나리오를 기술함. `hack/make-rules/build.sh`에서 ginkgo CLI를 빌드함 |
| **Gomega** | v1.39.0 | `go.mod` → `github.com/onsi/gomega` | Ginkgo와 함께 사용되는 매처/어설션 라이브러리임. `Expect(pod.Status).To(Equal("Running"))` 같은 가독성 높은 어설션을 제공함 |
| **testify** | v1.11.1 | `go.mod` → `github.com/stretchr/testify` | 유닛 테스트에서 사용하는 어설션 헬퍼임. `assert.Equal(t, expected, actual)` 같은 간결한 테스트 문법을 제공함 |
| **gotestsum** | - | `hack/make-rules/test.sh` 191행 | `go test`를 감싸서 JUnit XML 리포트를 생성하고, 테스트 결과를 컬러풀하게 출력하는 도구임. `hack/make-rules/test.sh`의 `installTools()` 함수에서 자동 설치됨 |

### 2.7 빌드 시스템 상세

| 기술 | 근거 파일 | 구체적 역할 |
|------|-----------|-------------|
| **Make** | `Makefile` (517행) | 빌드 진입점임. `all`, `test`, `verify`, `release`, `cross`, `clean` 등의 타겟을 정의하고, 실제 구현은 `hack/make-rules/*.sh`에 위임함 |
| **Docker 빌드 컨테이너** | `build/common.sh` | `kube-cross` 이미지(`registry.k8s.io/build-image/kube-cross`)를 사용하여 재현 가능한 빌드 환경을 제공함. `kube::build::run_build_command()` 함수가 컨테이너를 띄우고 그 안에서 명령을 실행함 |
| **Go Workspaces** | `go.work` | `go work` 명령으로 루트 모듈과 33개 staging 모듈을 하나의 개발 환경으로 통합함. `hack/update-vendor.sh`의 232행 부근에서 `go work sync`로 버전을 동기화함 |
| **Vendor** | `vendor/`, `hack/update-vendor.sh` (347행) | 모든 외부 의존성 소스를 `vendor/`에 복사하여 네트워크 없이도 빌드 가능하게 함. `go work vendor` 명령으로 생성됨 |

---

## 3. 핵심 바이너리 (실행 파일) 구성

> `hack/lib/golang.sh`의 `kube::golang::server_targets()` 함수(69-83행)에 빌드 대상이 정의되어 있음.

### 3.1 컨트롤 플레인 컴포넌트

| 바이너리 | 진입점 파일 | Docker 이미지 | 기본 포트 | 구체적 역할 |
|----------|------------|--------------|-----------|-------------|
| **kube-apiserver** | `cmd/kube-apiserver/apiserver.go` | `registry.k8s.io/kube-apiserver` | 6443 (HTTPS) | 모든 API 요청의 **유일한 진입점**임. 인증→인가→어드미션 제어→etcd 저장 파이프라인을 처리함. `build/server-image/kube-apiserver/Dockerfile`에서 **2단계 빌드**로 `cap_net_bind_service` 커널 캐퍼빌리티를 설정하여 비루트 사용자로도 443 포트에 바인딩 가능하게 함 |
| **kube-controller-manager** | `cmd/kube-controller-manager/controller-manager.go` | `registry.k8s.io/kube-controller-manager` | 10257 | `pkg/controller/` 아래 38개 서브 컨트롤러(Deployment, ReplicaSet, Job, DaemonSet, Node, PV, ServiceAccount 등)를 **하나의 프로세스**에서 실행하는 기능을 함. 각 컨트롤러가 독립적인 Reconciliation Loop를 돌림 |
| **kube-scheduler** | `cmd/kube-scheduler/scheduler.go` | `registry.k8s.io/kube-scheduler` | 10259 | `pkg/scheduler/framework/` 플러그인 프레임워크를 사용하여 Pending 상태의 파드를 적절한 노드에 배치함. Filter→Score→Bind 단계를 거침 |
| **cloud-controller-manager** | `cmd/cloud-controller-manager/main.go` | - | - | AWS, GCP, Azure 등 클라우드 제공자별 노드/라우트/로드밸런서 관리 로직을 분리하여 실행하는 기능을 함 |

### 3.2 노드 컴포넌트

| 바이너리 | 진입점 파일 | 기본 포트 | 구체적 역할 |
|----------|------------|-----------|-------------|
| **kubelet** | `cmd/kubelet/kubelet.go` | 10250 | 각 워커 노드에서 **파드 생명주기를 관리하는 에이전트**임. `pkg/kubelet/`의 46개 서브패키지가 컨테이너 실행(`container/`), 볼륨 마운트(`volume/`), 헬스체크 프로브(`prober/`), cgroups 관리(`cm/`), 이미지 관리(`images/`) 등을 담당함. CRI gRPC로 containerd와 통신함 |
| **kube-proxy** | `cmd/kube-proxy/proxy.go` | - | 각 노드에서 Service→Pod 네트워킹 규칙을 관리함. `pkg/proxy/iptables/`, `pkg/proxy/ipvs/`, `pkg/proxy/nftables/` 3가지 구현체를 선택 가능함 |

### 3.3 CLI 도구

| 바이너리 | 진입점 파일 | Dockerfile | 구체적 역할 |
|----------|------------|------------|-------------|
| **kubectl** | `cmd/kubectl/kubectl.go` | `build/server-image/kubectl/Dockerfile` → `/bin/kubectl`을 ENTRYPOINT로 설정 | 사용자가 클러스터와 상호작용하는 **주요 CLI**임. `staging/src/k8s.io/kubectl/` 및 `staging/src/k8s.io/cli-runtime/`에 핵심 로직이 구현됨. cobra 프레임워크로 서브커맨드를 구성함 |
| **kubeadm** | `cmd/kubeadm/kubeadm.go` | - | 클러스터 초기화(`kubeadm init`)와 노드 조인(`kubeadm join`)을 수행하는 부트스트랩 도구임. `cmd/kubeadm/app/phases/`에 초기화 단계별 로직이 구현됨 |
| **kubectl-convert** | `cmd/kubectl-convert/kubectl_convert.go` | - | 매니페스트의 API 버전을 변환하는 유틸리티임 (예: `apps/v1beta1` → `apps/v1`) |
| **kubemark** | `cmd/kubemark/hollow-node.go` | - | 대규모 클러스터를 시뮬레이션하여 스케일 테스트를 수행하는 도구임. "hollow" 노드를 생성하여 실제 리소스 없이 수천 노드 환경을 테스트함 |

---

## 4. 프로젝트 디렉토리 구조 (상세)

```
kubernetes/                          ← 프로젝트 루트 (KUBE_ROOT 변수가 이 경로를 가리킴)
│
├── Makefile                         ← 빌드 진입점 (517행). build/root/Makefile과 동일한 내용임.
│                                       SHELL=/usr/bin/env bash -e -o pipefail 로 설정됨
├── go.mod                           ← 모듈 정의. module k8s.io/kubernetes, go 1.25.7
├── go.sum                           ← 의존성 체크섬 (무결성 보장)
├── go.work                          ← Go 워크스페이스: use . 와 33개 staging 모듈을 나열함
├── .go-version                      ← "1.25.7" 한 줄 — Go 버전 고정
│
├── api/                             ← API 명세 정의
│   ├── api-rules/                   ← API 유효성 검증 규칙 파일들
│   ├── discovery/                   ← API 디스커버리 관련 파일들
│   └── openapi-spec/                ← OpenAPI v3 명세 (자동 생성됨)
│
├── build/                           ← 빌드 시스템 (8개 핵심 스크립트 + Dockerfile)
│   ├── common.sh                    ← 473행. 19개 함수 정의. 모든 빌드 스크립트의 공통 라이브러리임
│   ├── run.sh                       ← 37행. Docker 컨테이너 안에서 명령을 실행하는 래퍼임
│   ├── shell.sh                     ← 29행. Docker 빌드 컨테이너에 대화형 쉘로 접속하는 기능을 함
│   ├── release.sh                   ← 43행. 전체 릴리즈 프로세스 (빌드→테스트→패키징)를 실행함
│   ├── release-in-a-container.sh    ← 50행. Docker 컨테이너 안에서 릴리즈를 빌드함
│   ├── release-images.sh            ← 41행. 각 컴포넌트의 Docker 이미지를 생성함
│   ├── package-tarballs.sh          ← 28행. tar.gz 패키지를 생성함
│   ├── make-clean.sh                ← 27행. _output/ 디렉토리와 Docker 컨테이너를 정리함
│   ├── root/
│   │   └── Makefile                 ← 루트 Makefile과 동일한 내용 (517행)
│   ├── lib/
│   │   └── release.sh              ← 릴리즈 패키징 함수들 (package_tarballs, build_server_images 등)
│   ├── pause/
│   │   ├── Dockerfile              ← 4행. 최소 pause 컨테이너 (USER 65535:65535로 비루트 실행)
│   │   └── Dockerfile_windows      ← Windows용 pause 컨테이너
│   ├── server-image/
│   │   ├── Dockerfile              ← 2행. 범용 서버 이미지 (COPY --chmod=755 로 바이너리 복사)
│   │   ├── kube-apiserver/
│   │   │   └── Dockerfile          ← 2단계 빌드. setcap으로 cap_net_bind_service 커널 캐퍼빌리티 설정
│   │   └── kubectl/
│   │       └── Dockerfile          ← ENTRYPOINT ["/bin/kubectl"] 설정
│   └── build-image/
│       └── cross/                   ← kube-cross 크로스 컴파일 이미지 정의
│
├── cmd/                             ← 실행 바이너리 진입점 (각각 main() 함수 포함)
│   ├── kube-apiserver/              ← apiserver.go → main()
│   ├── kube-controller-manager/     ← controller-manager.go → main()
│   ├── kube-scheduler/              ← scheduler.go → main()
│   ├── kubelet/                     ← kubelet.go → main()
│   ├── kube-proxy/                  ← proxy.go → main()
│   ├── kubectl/                     ← kubectl.go → main()
│   ├── kubeadm/                     ← kubeadm.go → main()
│   ├── kubectl-convert/             ← kubectl_convert.go → main()
│   ├── kubemark/                    ← hollow-node.go → main()
│   ├── cloud-controller-manager/    ← main.go → main()
│   └── [코드 생성 도구들]/           ← gendocs, genfeaturegates, genyaml 등
│
├── hack/                            ← 개발자 유틸리티 스크립트 (가장 많은 스크립트가 위치)
│   ├── lib/                         ← Bash 유틸리티 라이브러리
│   │   ├── init.sh                  ← 219행. 모든 스크립트의 초기화 진입점.
│   │   │                               KUBE_ROOT, KUBE_OUTPUT 설정 후 5개 라이브러리 로드:
│   │   │                               util.sh, logging.sh, version.sh, golang.sh, etcd.sh
│   │   ├── logging.sh               ← 182행. 12개 로깅 함수 정의
│   │   │                               (kube::log::status, error, info, stack 등)
│   │   ├── golang.sh                ← 300+행. Go 빌드 환경 설정, 플랫폼 목록 관리
│   │   │                               서버 플랫폼: linux/{amd64,arm64,s390x,ppc64le}
│   │   │                               클라이언트 플랫폼: + darwin, windows 추가
│   │   ├── version.sh               ← 버전 정보(git tag 기반) 추출 함수
│   │   ├── etcd.sh                  ← etcd 설치/관리 유틸리티
│   │   └── util.sh                  ← 범용 유틸리티 함수
│   ├── make-rules/                  ← Makefile 타겟의 실제 구현체
│   │   ├── build.sh                 ← 30행. Go 바이너리 컴파일 실행
│   │   │                               → kube::golang::build_binaries "$@"
│   │   ├── cross.sh                 ← 39행. 5단계 크로스 컴파일
│   │   │                               서버→노드→클라이언트→테스트→서버테스트 순서
│   │   ├── test.sh                  ← 287행. 유닛 테스트 실행
│   │   │                               gotestsum + go test, JUnit XML 리포트 생성
│   │   │                               주요 변수: KUBE_TIMEOUT=180s, KUBE_RACE="-race"
│   │   ├── test-integration.sh      ← 통합 테스트 실행
│   │   ├── test-e2e-node.sh         ← 노드 E2E 테스트 실행
│   │   ├── verify.sh                ← 254행. 50개 이상의 verify-*.sh 스크립트를 일괄 실행
│   │   │                               QUICK 모드, EXCLUDED 패턴, JUnit 리포팅 지원
│   │   └── update.sh                ← 68행. 7개 update-*.sh 스크립트를 순차 실행
│   │                                    codegen, featuregates, docs, openapi, gofmt 등
│   ├── verify-*.sh (50+개)          ← 코드 품질 검증 스크립트들
│   ├── update-*.sh (30+개)          ← 코드 자동 생성 스크립트들
│   ├── local-up-cluster.sh          ← 로컬 개발용 전체 클러스터 구동 (50+개 설정 변수)
│   └── install-etcd.sh              ← 31행. etcd 다운로드 및 설치
│
├── pkg/                             ← 핵심 비즈니스 로직 (32개 디렉토리)
│   ├── controller/                  ← 38개 서브 컨트롤러
│   │   ├── deployment/              ← Deployment 롤아웃/롤백 관리
│   │   ├── replicaset/              ← 파드 복제본 수 유지
│   │   ├── job/                     ← 일회성 작업 완료 보장
│   │   ├── daemonset/               ← 모든 노드에 파드 배포
│   │   ├── statefulset/             ← 상태 유지 앱의 순서 배포
│   │   ├── nodelifecycle/           ← 노드 건강 상태 모니터링
│   │   ├── service/                 ← 서비스 리소스 관리
│   │   ├── endpoint/                ← 서비스↔파드 엔드포인트 매핑
│   │   ├── endpointslice/           ← 대규모 엔드포인트 분할 관리
│   │   ├── namespace/               ← 네임스페이스 생명주기
│   │   ├── garbagecollector/        ← 고아(orphan) 리소스 정리
│   │   ├── resourcequota/           ← 네임스페이스별 리소스 제한
│   │   ├── serviceaccount/          ← 서비스 계정/토큰 관리
│   │   ├── cronjob/                 ← 주기적 반복 작업 스케줄링
│   │   ├── ttlafterfinished/        ← 완료된 Job 자동 삭제
│   │   ├── certificates/            ← TLS 인증서 서명 요청 처리
│   │   └── ... (22개 더)
│   ├── kubelet/                     ← kubelet 구현체 (46개 서브패키지)
│   ├── scheduler/                   ← 스케줄러 구현체
│   │   ├── framework/               ← 플러그인 프레임워크
│   │   ├── queue/                   ← 스케줄링 큐
│   │   └── cache/                   ← 노드/파드 캐시
│   ├── proxy/                       ← kube-proxy 구현체 (15개 서브패키지)
│   │   ├── iptables/                ← iptables 기반
│   │   ├── ipvs/                    ← IPVS 기반
│   │   └── nftables/                ← nftables 기반
│   ├── kubeapiserver/               ← API 서버 구현 로직
│   ├── controlplane/                ← 컨트롤 플레인 초기화
│   ├── volume/                      ← 볼륨 관리 (20개 서브패키지)
│   ├── auth/                        ← 인증 메커니즘
│   ├── registry/                    ← 리소스 레지스트리 (팩토리 패턴)
│   └── ... (기타 유틸리티)
│
├── staging/src/k8s.io/              ← 33개 외부 공개 모듈
│   ├── api/                         ← API 타입 정의 (Pod, Service, Deployment 등)
│   ├── apimachinery/                ← API 직렬화, 변환, 리플렉션
│   ├── apiserver/                   ← 범용 API 서버 프레임워크
│   ├── client-go/                   ← 공식 Go 클라이언트 (Informer, Watch 등)
│   ├── kubectl/                     ← kubectl 핵심 로직
│   ├── cli-runtime/                 ← CLI 런타임 유틸리티
│   ├── code-generator/              ← 코드 자동 생성 도구
│   ├── cri-api/                     ← CRI gRPC 서비스 정의
│   └── ... (25개 더)
│
├── test/                            ← 테스트 인프라
│   ├── e2e/                         ← E2E 테스트 (27개 서브디렉토리)
│   ├── e2e_node/                    ← 노드 E2E (17개 서브디렉토리)
│   ├── integration/                 ← 통합 테스트 (64개 서브디렉토리)
│   ├── conformance/                 ← 적합성 테스트
│   └── images/                      ← 테스트용 Docker 이미지 (24개)
│
├── cluster/                         ← 클러스터 배포 (유지보수 모드, kubeadm 권장)
│   ├── kube-up.sh                   ← 클러스터 시작 (verify-prereqs → kube-up → validate-cluster)
│   ├── kube-down.sh                 ← 클러스터 종료 (verify-prereqs → kube-down)
│   ├── common.sh                    ← 공통 유틸리티 (create-kubeconfig 등)
│   └── gce/                         ← GCE 배포 설정
│
├── vendor/                          ← Go 벤더 디렉토리 (외부 의존성 소스 복사본)
├── third_party/                     ← 서드파티 코드
├── plugin/                          ← 어드미션/인가 플러그인
│   └── pkg/
│       ├── admission/               ← 어드미션 컨트롤러 플러그인
│       └── auth/                    ← 인가(RBAC 등) 플러그인
├── CHANGELOG/                       ← v1.2 ~ v1.36 변경 이력
└── LICENSES/                        ← 의존성 라이선스 파일들
```

---

## 5. Makefile 타겟 상세

> 근거: `Makefile` (517행), `SHELL := /usr/bin/env bash -e -o pipefail`

| Make 명령 | 실제 실행 스크립트 | 구체적 동작 |
|-----------|-------------------|-------------|
| `make all` | `hack/make-rules/build.sh` | `kube::golang::setup_env` → `kube::golang::build_binaries` → `kube::golang::place_bins` 순서로 Go 바이너리를 컴파일함. `WHAT=cmd/kubectl` 같이 특정 타겟만 지정 가능 |
| `make cross` | `hack/make-rules/cross.sh` | 5단계 크로스 컴파일: ①서버(linux/amd64,arm64,s390x,ppc64le) ②노드(+windows) ③클라이언트(+darwin) ④테스트 ⑤서버테스트 |
| `make test` | `hack/make-rules/test.sh` | gotestsum으로 `go test -race` 실행. `KUBE_TIMEOUT=180s`, `PARALLEL` 워커 수 지정 가능. JUnit XML 리포트 생성 |
| `make test-integration` | `hack/make-rules/test-integration.sh` | 통합 테스트 실행. `test/integration/` 하위 64개 디렉토리의 테스트를 수행함 |
| `make verify` | `hack/make-rules/verify.sh` | 50개 이상의 `hack/verify-*.sh`를 일괄 실행. `QUICK=true` 옵션으로 빠른 검증만 가능. 실패 시 JUnit 리포트에 기록 |
| `make update` | `hack/make-rules/update.sh` | 7개 update 스크립트 순차 실행: codegen, featuregates, api-compatibility, docs, openapi, gofmt, golangci-lint-config |
| `make release` | `build/release.sh` | ①`make cross` ②`make test`(KUBE_RELEASE_RUN_TESTS=y 시) ③`make test-integration` ④`kube::release::package_tarballs()` |
| `make release-images` | `build/release-images.sh` | 서버 바이너리 빌드 후 `kube::release::build_server_images()`로 Docker 이미지 생성. `docker buildx build --platform linux/${arch}` 사용 |
| `make quick-release` | `build/release.sh` | `KUBE_RELEASE_RUN_TESTS=n KUBE_FASTBUILD=true`로 테스트를 건너뛰고 linux/amd64만 빌드함 |
| `make clean` | `build/make-clean.sh` + `hack/make-rules/clean.sh` | `_output/` 디렉토리 삭제, Docker 빌드 컨테이너 정리 (`kube::build::clean()`) |
| `make lint` | `hack/verify-golangci-lint.sh` | golangci-lint 정적 분석. `-a` 옵션으로 변경된 코드만 검사, `-n`으로 hints 모드 |
| `make kubectl` | `hack/make-rules/build.sh cmd/kubectl` | kubectl만 빌드 (동적 타겟: cmd/ 아래 모든 디렉토리가 자동으로 make 타겟이 됨) |

---

## 6. 테스트 전략 상세

> 근거: `hack/make-rules/test.sh` (287행), `hack/make-rules/verify.sh` (254행)

| 테스트 유형 | 실행 명령 | 실행 스크립트 | 핵심 설정 | 상세 설명 |
|-------------|-----------|--------------|-----------|-----------|
| **정적 분석** | `make verify` | `hack/make-rules/verify.sh` | QUICK=true 가능 | 50개 이상의 verify 스크립트를 실행함. gofmt, lint, import, boilerplate, vendor, typecheck, shellcheck, govulncheck, licenses 등을 검증함 |
| **유닛 테스트** | `make test` | `hack/make-rules/test.sh` | KUBE_TIMEOUT=180s, KUBE_RACE="-race" | `gotestsum --format=pkgname-and-test-fails`으로 Go 유닛 테스트를 실행함. `KUBE_COVER=y`로 커버리지 수집 가능, `KUBE_JUNIT_REPORT_DIR`로 JUnit XML 출력 |
| **통합 테스트** | `make test-integration` | `hack/make-rules/test-integration.sh` | - | `test/integration/` 하위 64개 디렉토리의 테스트를 수행함. API 서버, kubelet, 스케줄러 등 컴포넌트 간 상호작용을 검증함 |
| **E2E 테스트** | `make test-e2e-node` | `hack/make-rules/test-e2e-node.sh` | - | Ginkgo 프레임워크로 `test/e2e_node/` 테스트를 실행함. 실제 kubelet을 구동하여 파드 생성/삭제를 검증함 |
| **적합성 테스트** | 별도 실행 | `test/conformance/` | - | 공식 Kubernetes API 준수 여부를 검증하는 테스트임. CNCF 인증에 사용됨 |

### 유닛 테스트(test.sh)의 상세 실행 흐름

```
1. kube::golang::setup_env()          ← Go 환경 설정
2. 테스트 패키지 탐색                  ← _test.go 파일이 있는 패키지를 자동 발견
   (vendor, test/e2e, test/integration 제외)
3. installTools()                      ← gotestsum, prune-junit-xml 설치 (191행)
4. checkFDs()                          ← 파일 디스크립터 한도 확인 (270행)
5. go test 실행 (gotestsum 래핑)
   - -race 플래그 (기본 활성화)
   - -timeout 180s (기본값)
   - -count 1 (반복 횟수)
   - -p ${PARALLEL} (병렬 워커)
6. JUnit XML 리포트 생성               ← KUBE_JUNIT_REPORT_DIR 지정 시
7. 커버리지 HTML 리포트 생성            ← KUBE_COVER=y 지정 시
```

---

## 7. 코드 자동 생성 시스템 상세

> 근거: `hack/update-codegen.sh` (1037행, 15개 생성 함수)

Kubernetes는 **Go 소스 코드의 `+k8s:` 태그**를 읽어서 반복적인 코드를 자동 생성하는 시스템을 사용함.

| 생성 함수 | 소스 위치 (행) | 입력 태그 | 출력 | 설명 |
|-----------|---------------|-----------|------|------|
| `codegen::protobuf` | 115행 | - | `*.pb.go` | Protocol Buffers 직렬화 코드를 생성함 |
| `codegen::deepcopy` | 165행 | `+k8s:deepcopy-gen=` | `zz_generated.deepcopy.go` | 모든 API 객체의 Deep Copy 메서드를 자동 생성함 |
| `codegen::swagger` | 266행 | - | swagger doc | Swagger API 문서를 자동 생성함 |
| `codegen::prerelease` | 289행 | `+k8s:prerelease-lifecycle-gen=true` | lifecycle 코드 | Alpha/Beta/GA 생명주기 코드를 생성함 |
| `codegen::defaults` | 353행 | `+k8s:defaulter-gen=` | defaulter 코드 | API 객체의 기본값 설정 코드를 생성함 |
| `codegen::validation` | 415행 | `+k8s:validation-gen=` | validation 코드 | API 유효성 검증 코드를 생성함 |
| `codegen::conversions` | 499행 | `+k8s:conversion-gen=` | conversion 코드 | API 버전 간 변환 코드를 생성함 (v1beta1 ↔ v1) |
| `codegen::register` | 560행 | `+k8s:register-gen=package` | register 코드 | API 그룹/버전 등록 코드를 생성함 |
| `codegen::openapi` | 631행 | `+k8s:openapi-gen=true` | OpenAPI 코드 | OpenAPI 명세 코드를 생성함 |
| `codegen::applyconfigs` | 714행 | - | apply config 코드 | Server-Side Apply 설정 코드를 생성함 |
| `codegen::clients` | 756행 | - | client 코드 | client-go 클라이언트 코드를 생성함 |
| `codegen::listers` | 811행 | - | lister 코드 | 리소스 리스터(lister) 코드를 생성함 |
| `codegen::informers` | 851행 | - | informer 코드 | Informer 코드를 생성함 |
| `codegen::subprojects` | 900행 | - | 서브프로젝트 코드 | staging 서브프로젝트의 코드를 생성함 |
| `codegen::protobindings` | 925행 | - | protobuf 바인딩 | Protocol Buffers 바인딩을 생성함 |

### 태그 기반 코드 생성 예시

```go
// staging/src/k8s.io/api/core/v1/types.go

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:prerelease-lifecycle-gen:introduced=1.0
type Pod struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   PodSpec    `json:"spec,omitempty"`
    Status PodStatus  `json:"status,omitempty"`
}
```

위 `+k8s:deepcopy-gen` 태그를 읽고 → `zz_generated.deepcopy.go` 파일에 `func (in *Pod) DeepCopy() *Pod` 메서드를 자동 생성함.

---

## 8. Docker 이미지 빌드 상세

> 근거: `build/server-image/` Dockerfile들, `build/lib/release.sh`

### 이미지별 Dockerfile 구조

| 이미지 | Dockerfile 위치 | 빌드 방식 | 특이사항 |
|--------|-----------------|-----------|----------|
| **kube-apiserver** | `build/server-image/kube-apiserver/Dockerfile` | **2단계(multi-stage) 빌드** | Stage 1: `setcap cap_net_bind_service=+ep`로 비루트 사용자가 443 포트 바인딩 가능하게 함. Stage 2: 최종 이미지에 바이너리 복사 |
| **kube-controller-manager** | `build/server-image/Dockerfile` (공용) | 단일 단계 | `COPY --chmod=755 ${BINARY} /usr/local/bin/${BINARY}` |
| **kube-scheduler** | `build/server-image/Dockerfile` (공용) | 단일 단계 | 위와 동일 |
| **kube-proxy** | `build/server-image/Dockerfile` (공용) | 단일 단계 | 기본 이미지가 `distroless-iptables`임 (iptables 포함) |
| **kubectl** | `build/server-image/kubectl/Dockerfile` | 단일 단계 | `ENTRYPOINT ["/bin/kubectl"]` 설정 |
| **pause** | `build/pause/Dockerfile` | 단일 단계 | `USER 65535:65535`로 비루트 실행. 최소한의 pause 바이너리만 포함 |

### 기본 이미지 매핑 (build/common.sh에서 정의)

```bash
# build/common.sh 에서 정의된 기본 이미지들:
KUBE_APISERVER_BASE_IMAGE            → distroless-iptables (동적 링킹)
KUBE_CONTROLLER_MANAGER_BASE_IMAGE   → go-runner (정적 링킹)
KUBE_SCHEDULER_BASE_IMAGE            → go-runner (정적 링킹)
KUBE_PROXY_BASE_IMAGE                → distroless-iptables (iptables 필요)
KUBECTL_BASE_IMAGE                   → go-runner (정적 링킹)
KUBE_BUILD_SETCAP_IMAGE              → setcap 이미지 (apiserver 전용)

# 레지스트리:
KUBE_BASE_IMAGE_REGISTRY = registry.k8s.io/build-image
```

---

## 9. 지원 플랫폼

> 근거: `hack/lib/golang.sh` (23-66행, readonly 배열)

| 카테고리 | 지원 플랫폼 | 용도 |
|----------|------------|------|
| **서버** | `linux/amd64`, `linux/arm64`, `linux/s390x`, `linux/ppc64le` | 컨트롤 플레인 + 노드 컴포넌트 실행 |
| **노드** | 서버 플랫폼 + `windows/amd64` | 워커 노드 에이전트(kubelet, kube-proxy) 실행 |
| **클라이언트** | 서버 + 노드 + `darwin/amd64`, `darwin/arm64`, `windows/386`, `linux/386` | kubectl 등 CLI 도구 실행 |
| **테스트** | `linux/amd64`, `linux/arm64`, `linux/s390x`, `linux/ppc64le`, `darwin/amd64`, `darwin/arm64`, `windows/amd64` | 테스트 바이너리(ginkgo, e2e.test) 실행 |

`KUBE_FASTBUILD=true` 설정 시 현재 호스트 플랫폼만 빌드하여 빌드 시간을 대폭 단축함.
