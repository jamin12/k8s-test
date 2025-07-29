# Kubernetes 단계별 실습 가이드: 사전 준비사항

이 문서는 각 단계(v4, v5, v6)를 진행하기 위해 필요한 쿠버네티스 YAML 파일 외의 외부 도구 설치 및 환경 설정 방법을 안내합니다.

---

## v4: 인그레스(Ingress)를 이용한 L7 라우팅

v4는 쿠버네티스의 기본 `Ingress` 리소스를 사용하여 외부 트래픽을 서비스로 연결합니다. 이를 위해서는 `Ingress` 규칙을 실제로 처리해 줄 **인그레스 컨트롤러**가 클러스터에 반드시 필요합니다.

### 필수 도구: NGINX Ingress Controller

가장 널리 사용되는 NGINX 기반의 인그레스 컨트롤러입니다.

#### 설치 방법
아래 명령어를 사용하여 클러스터에 NGINX 인그레스 컨트롤러를 설치합니다.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```
이 명령어는 `ingress-nginx` 네임스페이스에 컨트롤러 파드와 외부 트래픽을 받기 위한 `LoadBalancer` 타입의 서비스를 생성합니다.

#### 테스트 방법
1.  **외부 IP 확인**: 아래 명령어로 컨트롤러 서비스의 외부 IP(`EXTERNAL-IP`)를 확인합니다.
    ```bash
    kubectl get svc -n ingress-nginx
    ```
2.  **`Host` 헤더와 함께 요청**: `Ingress` 리소스는 `Host` 헤더를 기준으로 라우팅하므로, `curl` 요청 시 `-H` 플래그로 호스트를 지정해야 합니다.
    ```bash
    # [INGRESS_EXTERNAL_IP]를 위에서 확인한 IP로 교체하세요.
    curl -H "Host: myapp.example.com" http://[INGRESS_EXTERNAL_IP]
    ```

---

## v5: GitOps 도입 (Argo CD)

v5는 Git 저장소를 '단일 진실 공급원(Single Source of Truth)'으로 사용하여 클러스터의 상태를 자동으로 동기화하는 GitOps를 도입합니다. 이를 위해 **Argo CD**를 사용합니다.

### 필수 도구: Argo CD

대표적인 쿠버네티스용 GitOps(Continuous Delivery) 도구입니다.

#### 설치 방법
1.  **네임스페이스 생성**: Argo CD를 위한 전용 네임스페이스를 생성합니다.
    ```bash
    kubectl create namespace argocd
    ```
2.  **Argo CD 설치**: 공식 매니페스트 파일을 사용하여 클러스터에 Argo CD를 설치합니다.
    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

#### 초기 설정 및 사용법
1.  **초기 비밀번호 확인**: 아래 명령어로 `admin` 계정의 초기 비밀번호를 확인합니다.
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
2.  **UI 접속**: `port-forward`를 사용하여 로컬 PC에서 Argo CD UI에 접속합니다.
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
    이후 웹 브라우저에서 `https://localhost:8080`으로 접속하고, `admin` 계정과 위에서 확인한 비밀번호로 로그인합니다.

3.  **애플리케이션 등록**: UI에서 `+ NEW APP`을 클릭하고, 이 Git 저장소의 주소와 배포할 YAML 파일들이 있는 경로(`v5`)를 지정하여 애플리케이션을 생성합니다.
    *   **Sync Policy**는 `Automatic`으로 설정해야 Git 변경사항이 자동으로 클러스터에 반영됩니다.

---

## v6: Istio 서비스 메시 도입

v6는 서비스 메시 도구인 **Istio**를 사용하여 카나리 배포와 같은 고급 트래픽 관리를 구현합니다.

### 필수 도구: Istio & istioctl

Istio 서비스 메시와 Istio를 쉽게 설치하고 관리하기 위한 전용 CLI 도구(`istioctl`)가 필요합니다.

#### 설치 방법
1.  **`istioctl` CLI 설치 (macOS)**: Homebrew를 사용하는 것이 가장 간편합니다.
    ```bash
    brew install istioctl
    ```
    (다른 OS는 Istio 공식 문서를 참고하세요.)

2.  **Istio 설치**: 실무 환경과 유사하게, 설정을 코드화하여 관리할 수 있는 `IstioOperator` 파일을 사용하여 설치합니다.
    *   먼저, 설치 설정을 담은 `istio.yml`과 같은 YAML 파일을 작성합니다. (v6 디렉터리 참고)
    *   `istioctl`을 사용하여 해당 설정 파일으로 클러스터에 Istio를 설치합니다.
    ```bash
    # istio.yml 파일이 있는 경로에서 실행합니다.
    istioctl install -f istio.yml -y
    ```

#### 환경 설정
1.  **사이드카 자동 주입 활성화**: Istio가 파드의 트래픽을 제어할 수 있도록, 애플리케이션을 배포할 네임스페이스에 사이드카 자동 주입 레이블을 추가해야 합니다.
    ```bash
    kubectl label namespace default istio-injection=enabled --overwrite
    ```
2.  **파드 재시작**: 레이블 추가 후, 이미 실행 중인 파드가 있다면 `kubectl rollout restart deployment <deployment-name>` 명령어로 재시작하여 사이드카가 주입되도록 해야 합니다.

#### 테스트 방법
1.  **게이트웨이 외부 IP 확인**: Istio는 자체 인그레스 게이트웨이를 사용합니다. 아래 명령어로 외부 IP를 확인합니다.
    ```bash
    kubectl get svc istio-ingressgateway -n istio-system
    ```
2.  **`Host` 헤더와 함께 요청**: `Gateway` 리소스도 `Host`를 기준으로 동작하므로, `curl` 요청 시 호스트를 지정해야 합니다.
    ```bash
    # [ISTIO_GATEWAY_IP]를 위에서 확인한 IP로 교체하세요.
    for i in $(seq 1 10); do curl -s -H "Host: myapp.example.com" http://[ISTIO_GATEWAY_IP]; done
    ```
