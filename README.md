### GKE Autopilot에서 Gateway API와 Helm을 이용한 Argo CD HTTPS 설정 완벽 가이드

이 가이드는 GKE Autopilot 클러스터에 Argo CD와 Argo Rollouts를 설치하고, Kubernetes Gateway API를 사용하여 안전한 HTTPS 엔드포인트를 설정하는 전체 과정을 안내합니다.

#### 0단계: Google Cloud 환경 설정 및 초기화

가이드를 시작하기 전에, Google Cloud 프로젝트와 로컬 개발 환경을 설정하고 필요한 모든 서비스를 활성화합니다.

1.  **Google Cloud SDK 설치 및 초기화:**
    로컬 머신에 `gcloud` CLI가 설치되어 있지 않다면 [공식 문서](https://cloud.google.com/sdk/docs/install)를 따라 설치합니다. 이미 설치되어 있다면 최신 버전으로 업데이트합니다.

    ```bash
    gcloud components update
    ```

    `gcloud`를 처음 사용하는 경우, 계정 인증 및 프로젝트 설정을 위해 초기화를 진행합니다.

    ```bash
    gcloud init
    ```
    > 화면의 안내에 따라 로그인하고, 이 가이드에서 사용할 Google Cloud 프로젝트를 선택하세요.

2.  **환경 변수 설정:**
    반복적인 입력을 줄이고 실수를 방지하기 위해, 프로젝트 ID와 리전 정보를 환경 변수로 설정합니다.

    ```bash
    export PROJECT_ID=$(gcloud config get-value project)
    export REGION="asia-northeast3" # 서울 리전
    ```
    터미널 세션이 종료되면 이 변수들은 사라지므로, 새 터미널을 열 때마다 다시 실행해주세요.

3.  **필수 API 활성화:**
    GKE, Gateway API, Compute Engine(로드밸런서와 IP 주소용) API를 활성화합니다. API를 활성화하는 데 몇 분 정도 소요될 수 있습니다.

    ```bash
    gcloud services enable \
        container.googleapis.com \
        compute.googleapis.com \
        --project=${PROJECT_ID}
    ```
    이 명령은 필요한 모든 서비스를 한 번에 활성화하여, 이후 단계에서 권한 오류가 발생하는 것을 방지합니다.

#### 사전 준비 사항
*   Google Cloud 계정 및 프로젝트 (위 단계에서 설정 완료)
*   로컬 머신에 `gcloud` CLI, `kubectl`, `helm`이 설치 및 구성되어 있어야 합니다.
*   HTTPS 접속에 사용할 도메인 이름을 소유하고 있어야 합니다. (예: `argocd.mydomain.com`)

---

### 1단계: GKE Autopilot 클러스터 생성

Autopilot은 노드 관리를 Google에 위임하여 인프라 걱정 없이 애플리케이션에만 집중할 수 있게 해주는 GKE의 운영 모드입니다.

1.  **GKE 클러스터 생성:**
    `asia-northeast3`(서울) 리전에 `argocd-cluster`라는 이름의 Autopilot 클러스터를 생성합니다.

    ```bash
    gcloud container clusters create-auto argocd-cluster \
        --project=${PROJECT_ID} \
        --region=${REGION} \
        --release-channel=regular \
        --cluster-version=latest
    ```    
    > 생성에는 몇 분 정도 소요될 수 있습니다.

2.  **클러스터 인증 정보 가져오기:**
    `kubectl`이 클러스터와 통신할 수 있도록 인증 정보를 가져옵니다.

    ```bash
    gcloud container clusters get-credentials argocd-cluster --region=${REGION}
    ```

### 2단계: Argo CD 및 Argo Rollouts 설치

Helm을 사용하여 Argo CD와 Argo Rollouts를 설치합니다. 이때, 저희가 겪었던 Health Check 문제를 사전에 방지하기 위해 중요한 설정을 추가합니다.

1.  **Argo CD 네임스페이스 생성:**
    ```bash
    kubectl create namespace argocd
    ```

2.  **Argo Project Helm 리포지토리 추가:**
    ```bash
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
    ```

3.  **Argo CD 설치 (⭐ 핵심 설정 포함):**
    `helm install` 명령에 `--set-string` 플래그를 사용하여, Google Cloud Load Balancer의 Health Check가 정상적으로 작동하도록 `server.insecure` 옵션을 활성화합니다. 이렇게 하면 Argo CD 서버가 HTTP Health Check 요청에 대해 HTTPS로 리디렉션하지 않고 `200 OK`로 바로 응답하게 됩니다.

    ```bash
    helm install argo-cd argo/argo-cd \
        --namespace argocd \
        --set-string server.config."server.insecure"='true'
    ```

4.  **Argo Rollouts 컨트롤러 설치:**
    ```bash
    helm install argo-rollouts argo/argo-rollouts \
        --namespace argocd
    ```

5.  ⭐ **핵심 설정: Health Check를 위한 Insecure 설정 강제 적용**
    Helm 차트의 기본값으로 인해 `server.insecure` 설정이 `false`로 유지될 수 있습니다. 이를 해결하기 위해, 설치 후 직접 `ConfigMap`을 수정하고 Pod를 재시작하여 설정을 확실하게 적용합니다.

    ```bash
    # 1. ConfigMap에 server.insecure: 'true' 데이터를 패치(추가/수정)합니다.
    kubectl patch configmap argocd-cmd-params-cm -n argocd -p '{"data":{"server.insecure":"true"}}'

    # 2. 변경된 ConfigMap을 적용하기 위해 Deployment를 재시작합니다.
    kubectl rollout restart deployment argo-cd-argocd-server -n argocd
    ```

### 3단계: Gateway API를 이용한 HTTPS 설정

이제 Gateway API를 사용하여 외부에서 Argo CD UI로 안전하게 접속할 수 있는 HTTPS 엔드포인트를 만듭니다.

1.  **전역 고정 IP 주소 예약:**
    로드밸런서에 사용할 고정 IP 주소를 예약합니다.

    ```bash
    gcloud compute addresses create argocd-gateway-ip --global
    ```

2.  **예약된 IP 주소 확인:**
    예약된 IP 주소를 확인하고, 이 주소를 DNS에 등록해야 합니다.

    ```bash
    export GATEWAY_IP=$(gcloud compute addresses describe argocd-gateway-ip --global --format="value(address)")
    echo "예약된 IP 주소: ${GATEWAY_IP}"
    ```
    > **DNS 설정:** 출력된 IP 주소를 복사하여 사용 중인 DNS 서비스에서 `argocd.mydomain.com`과 같은 원하는 도메인에 **A 레코드**로 등록하세요. DNS 전파에는 시간이 걸릴 수 있습니다.

3.  ⭐ **핵심 설정: 로컬 머신 DNS 설정 (`/etc/hosts`)**
    `argocd.mydomain.com`은 실제 DNS에 등록된 도메인이 아니므로, 테스트를 위해 로컬 컴퓨터가 이 도메인을 GKE Gateway의 IP 주소로 인식하도록 `/etc/hosts` 파일을 수정해야 합니다. 관리자 권한이 필요합니다.

    *   **macOS / Linux:** `sudo nano /etc/hosts`
    *   **Windows:** 관리자 권한으로 메모장을 열고 `C:\Windows\System32\drivers\etc\hosts` 파일 열기

    파일의 맨 아래에 아래 형식의 줄을 추가하고 저장합니다.
    ```
    # 예시: 34.54.162.112 argocd.mydomain.com
    ${GATEWAY_IP} argocd.mydomain.com
    ```
    **설정 확인:** 터미널에서 `ping argocd.mydomain.com`을 실행했을 때, 위에서 받은 IP 주소로 응답이 오는지 확인합니다.

4.  **TLS 인증서 생성 및 Secret 저장 (⭐ 핵심 설정 3):**
    Google Cloud Load Balancer가 지원하는 **RSA 2048비트** 키로 인증서를 생성합니다.
    ```bash
    # RSA 2048비트 인증서와 키 생성
    openssl req -x509 -newkey rsa:2048 -nodes \
      -keyout tls.key -out tls.crt -sha256 \
      -days 365 -subj "/CN=argocd.mydomain.com"

    kubectl create secret tls argocd-tls-cert -n argocd --cert=tls.crt --key=tls.key
    ```
    > `argocd.mydomain.com`을 실제 사용하는 도메인으로 변경하세요.

5.  **Gateway 리소스 배포:**
    아래 내용을 `gateway.yaml` 파일로 저장합니다. 이 리소스는 Google Cloud Load Balancer를 생성하고, 443 포트로 들어오는 HTTPS 트래픽을 처리하며, 방금 생성한 TLS 인증서를 사용하도록 설정합니다.

    ```yaml
    # gateway.yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: argocd-gateway
      namespace: argocd
    spec:
      gatewayClassName: gke-l7-global-external-managed
      listeners:
      - name: https
        protocol: HTTPS
        port: 443
        allowedRoutes:
          namespaces:
            from: Same
        tls:
          mode: Terminate
          certificateRefs:
          - name: argocd-tls-cert # 3단계에서 생성한 Secret 이름
    ```

6.  **HTTPRoute 리소스 배포:**
    아래 내용을 `httproute.yaml` 파일로 저장합니다. `Gateway`로 들어온 특정 호스트 이름(`argocd.mydomain.com`)의 트래픽을 내부 `argo-cd-argocd-server` 서비스로 전달하는 라우팅 규칙입니다.

    ```yaml
    # httproute.yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: argocd-http-route
      namespace: argocd
    spec:
      parentRefs:
      - name: argocd-gateway # 바로 위에서 만든 Gateway 이름
      hostnames:
      - "argocd.mydomain.com" # 실제 사용하는 도메인
      rules:
      - backendRefs:
        - name: argo-cd-argocd-server
          port: 80 # Service의 포트
    ```

7.  **HealthCheckPolicy 리소스 배포 (⭐ 핵심 설정):**
    아래 내용을 `healthcheck.yaml` 파일로 저장합니다. 이것이 바로 로드밸런서가 Argo CD Pod의 상태를 **정확하게** 확인할 수 있도록 하는 핵심 설정입니다.

    *   `type: HTTP`: Pod가 HTTPS로 리디렉션하지 않으므로 일반 HTTP로 검사합니다.
    *   `port: 8080`: Pod 컨테이너가 실제로 리스닝하는 포트입니다.
    *   `requestPath: /healthz`: Argo CD의 공식 Health Check 경로입니다.

    ```yaml
    # healthcheck.yaml
    apiVersion: networking.gke.io/v1
    kind: HealthCheckPolicy
    metadata:
      name: argocd-health-check-policy
      namespace: argocd
    spec:
      default:
        config:
          type: HTTP
          httpHealthCheck:
            port: 8080
            requestPath: /healthz
      targetRef:
        group: ""
        kind: Service
        name: argo-cd-argocd-server
    ```

8.  **모든 리소스 적용:**
    ```bash
    kubectl apply -f gateway.yaml
    kubectl apply -f httproute.yaml
    kubectl apply -f healthcheck.yaml
    ```
    > 리소스가 프로비저닝되고 DNS가 전파되기까지 5~10분 정도 소요될 수 있습니다.

### 4단계: 샘플 애플리케이션 배포 및 테스트

모든 설정이 완료되었는지 확인하기 위해 간단한 샘플 앱을 Argo CD를 통해 배포합니다.

1.  **Argo CD 초기 비밀번호 확인:**
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```

2.  **Argo CD UI 접속:**
    웹 브라우저에서 `https://argocd.mydomain.com`으로 접속합니다. 사용자 이름은 `admin`이며, 비밀번호는 위에서 확인한 값을 사용합니다. (자체 서명 인증서이므로 브라우저의 보안 경고를 수락해야 합니다.)

3.  **샘플 앱 배포:**
    아래 내용을 `sample-app.yaml`로 저장합니다. 이 파일은 `rollouts-demo`라는 앱을 배포하는 Argo CD `Application` 리소스입니다.

    ```yaml
    # sample-app.yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: rollouts-demo
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/rollouts-demo.git
        targetRevision: HEAD
        path: .
      destination:
        server: https://kubernetes.default.svc
        namespace: default
    ```

4.  **애플리케이션 리소스 적용:**
    ```bash
    kubectl apply -f sample-app.yaml
    ```

5.  **배포 확인:**
    Argo CD UI에 다시 접속하면 `rollouts-demo` 앱이 나타나고, 동기화가 완료되면 `Healthy`와 `Synced` 상태로 표시됩니다. UI에서 배포된 리소스들(Rollout, Service, ReplicaSet 등)을 모두 확인할 수 있습니다.

---

### 5단계: 문제 해결 (Troubleshooting)

#### 증상 1: 브라우저에 `net::ERR_CERT_AUTHORITY_INVALID` 오류 발생

*   **원인:** 우리가 직접 만든 **자체 서명 인증서**를 사용했기 때문에 발생하는 정상적인 보안 경고입니다.
*   **해결:** 화면에서 **"Advanced"** 버튼을 누르고 **"Proceed to argocd.mydomain.com (unsafe)"** 링크를 클릭하여 접속합니다.

#### 증상 2: 일반 브라우저 탭에서는 "Proceed to..." 링크 없이 접속이 완전히 차단됩니다.

*   **원인:** 브라우저에 **HSTS 정책**이 캐시되어, 신뢰할 수 없는 인증서를 사용하는 사이트의 예외를 허용하지 않기 때문입니다.
*   **해결:** Chrome 주소창에 `chrome://net-internals/#hsts`를 입력 후, **"Delete domain security policies"** 섹션에서 해당 도메인을 삭제하고 브라우저를 재시작합니다.

#### 증상 3: `503 Service Unavailable` 오류가 발생하며, 확인 결과 `server.insecure` 설정이 `false`로 되어 있습니다.

*   **원인:** Helm 차트의 기본값이 설치 시 전달한 인수를 덮어썼거나, 다른 Helm 업그레이드 과정에서 설정이 초기화되었을 수 있습니다. 이로 인해 Health Check가 실패합니다.
*   **해결:** **2단계의 "핵심 설정 1"** 부분을 다시 실행하세요. `kubectl patch` 명령으로 ConfigMap을 직접 수정한 뒤, `kubectl rollout restart`로 Pod를 재시작하여 설정을 확실하게 적용하는 것이 가장 안정적인 방법입니다.

#### 증상 4: Gateway 이벤트에 `The SSL key size is unsupported` 오류가 표시됩니다.

*   **원인:** Google Cloud Load Balancer는 **RSA 2048비트** 키만 지원하는데, 다른 크기(예: RSA 4096)의 키로 인증서를 생성했기 때문입니다.
*   **해결:** **3단계의 "핵심 설정 3"**을 다시 확인하세요. `openssl` 명령어가 `rsa:2048`로 되어 있는지 확인하고, 잘못된 Secret을 삭제한 후 올바른 크기의 키로 Secret을 다시 생성합니다.

#### 증상 5: 로컬 PC에서 도메인으로 접속이 안 되거나, `ping`이 실패합니다.

*   **원인:** 가상의 도메인 이름을 사용하고 있기 때문에, 로컬 컴퓨터가 이 도메인에 해당하는 IP 주소를 알지 못합니다.
*   **해결:** **3단계의 "핵심 설정 2"**를 확인하세요. `/etc/hosts` 파일에 GKE Gateway의 IP 주소와 도메인 이름이 정확하게 입력되었는지 확인합니다. (관리자 권한으로 수정해야 합니다.)

#### 증상 6: `curl http://...` 또는 `curl argocd.mydomain.com` 실행 시 `Connection reset by peer` 오류가 발생합니다.

*   **원인:** Gateway가 HTTP(80) 포트를 리스닝하도록 설정되어 있지 않은 상태에서 HTTP로 접속을 시도했기 때문입니다.
*   **해결:**
    *   **빠른 확인:** `https://`를 명시하여 `curl -kv https://argocd.mydomain.com`으로 접속합니다.
    *   **영구적인 해결책:** 사용자의 편의를 위해 HTTP에서 HTTPS로 자동 리디렉션을 설정하는 것이 좋습니다. (이전 답변의 리디렉션용 Gateway 및 HTTPRoute 설정 참고)