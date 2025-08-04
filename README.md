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
        --cluster-version=latest
    ```    *   생성에는 몇 분 정도 소요될 수 있습니다.

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

### 3단계: Gateway API를 이용한 HTTPS 설정

이제 Gateway API를 사용하여 외부에서 Argo CD UI로 안전하게 접속할 수 있는 HTTPS 엔드포인트를 만듭니다.

1.  **전역 고정 IP 주소 예약:**
    로드밸런서에 사용할 고정 IP 주소를 예약합니다.

    ```bash
    gcloud compute addresses create argocd-gateway-ip \
        --global
    ```

2.  **예약된 IP 주소 확인:**
    예약된 IP 주소를 확인하고, 이 주소를 DNS에 등록해야 합니다.

    ```bash
    export GATEWAY_IP=$(gcloud compute addresses describe argocd-gateway-ip --global --format="value(address)")
    echo "예약된 IP 주소: ${GATEWAY_IP}"
    ```
    > **DNS 설정:** 출력된 IP 주소를 복사하여 사용 중인 DNS 서비스에서 `argocd.mydomain.com`과 같은 원하는 도메인에 **A 레코드**로 등록하세요. DNS 전파에는 시간이 걸릴 수 있습니다.

3.  **TLS 인증서 생성 및 Secret 저장:**
    테스트를 위해 자체 서명 인증서를 생성합니다. (프로덕션 환경에서는 Let's Encrypt 등 신뢰할 수 있는 인증서를 사용하세요.)

    ```bash
    # 자체 서명 인증서와 키 생성
    openssl req -x509 -newkey rsa:4096 -nodes \
      -keyout tls.key -out tls.crt -sha256 \
      -days 365 -subj "/CN=argocd.mydomain.com"

    # Kubernetes Secret으로 저장
    kubectl create secret tls argocd-tls-cert -n argocd \
      --cert=tls.crt \
      --key=tls.key
    ```
    *   `argocd.mydomain.com`을 실제 사용하는 도메인으로 변경하세요.

4.  **Gateway 리소스 배포:**
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

5.  **HTTPRoute 리소스 배포:**
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

6.  **HealthCheckPolicy 리소스 배포 (⭐ 핵심 설정):**
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

7.  **모든 리소스 적용:**
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

이 섹션은 우리가 함께 겪었던 문제들을 중심으로, 유사한 상황 발생 시 해결할 수 있는 방법을 안내합니다.

#### 증상 1: `curl -kv https://argocd.mydomain.com`이 `503 Service Unavailable` 또는 `no healthy upstream` 오류를 반환합니다.

이는 **로드밸런서의 Health Check가 실패**했음을 의미합니다.

*   **원인 A: Argo CD 서버의 HTTPS 리디렉션**
    *   **진단:** `kubectl run`으로 테스트 Pod를 띄워 `curl http://<ARGO_CD_POD_IP>:8080/healthz` 실행 시 `307 Temporary Redirect`가 반환됩니다.
    *   **해결:** **2단계**에서 설명한 대로, `argocd-cmd-params-cm` ConfigMap에 `server.insecure: 'true'`를 설정하고 Pod를 재시작하세요.
        ```bash
        kubectl patch configmap argocd-cmd-params-cm -n argocd -p '{"data":{"server.insecure":"true"}}'
        kubectl rollout restart deployment argo-cd-argocd-server -n argocd
        ```

*   **원인 B: 잘못된 HealthCheckPolicy 설정**
    *   **진단:** Health Check 정책의 타입이나 포트가 잘못되었습니다.
    *   **해결:** **3단계**의 `healthcheck.yaml` 파일 내용이 올바른지 (`type: HTTP`, `port: 8080`) 다시 한번 확인하고 적용하세요.

*   **원인 C: NetworkPolicy에 의한 차단**
    *   **진단:** 클러스터에 기본 차단(deny-all) `NetworkPolicy`가 적용된 경우, Health Checker의 IP 대역이 차단될 수 있습니다. `curl`이 응답 없이 멈추는 현상으로 나타날 수 있습니다.
    *   **해결:** Google Cloud Health Checker IP 대역(`130.211.0.0/22`, `35.191.0.0/16`)을 허용하는 `NetworkPolicy`를 `argocd` 네임스페이스에 추가하세요.
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: allow-gclb-health-checks
          namespace: argocd
        spec:
          podSelector:
            matchLabels:
              app.kubernetes.io/name: argocd-server
          ingress:
          - from:
            - ipBlock: { cidr: "130.211.0.0/22" }
            - ipBlock: { cidr: "35.191.0.0/16" }
            ports:
            - protocol: TCP
              port: 8080
        ```

#### 증상 2: Google Cloud Console의 `부하 분산 > 백엔드 서비스`에서 백엔드가 `UNHEALTHY`로 표시됩니다.

이것은 **증상 1**의 근본 원인입니다. 위에서 설명한 Health Check 실패 원인들을 순서대로 점검하면 해결됩니다. 콘솔의 상태가 `HEALTHY`로 바뀔 때까지 1~2분 정도 기다려야 합니다.