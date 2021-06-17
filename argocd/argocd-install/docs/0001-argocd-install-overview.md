# ArgoCD Install 살펴보기

ArgoCD Install 을 위한 directory 입니다.

Directory Structure 는 아래와 같습니다.

```
- dev
  - base
  - values-dev.yaml
- prd
  - base
  - values-prd.yaml
```

`base`에는 template 가 있게 되고, 그 template manifest에는 환경별로 서로 다른 parameter
를 주입하여 최종적으로 원하는 pure manifest 을 얻게 됩니다.

다만, 이 구조에서 kustomize의 overlay 방식 사용하지는 않고 helm의 template engine을 사용하여 pure manifest 를 얻도록
하겠습니다.

helm을 사용하는 이유는 많은 프로젝트들이 helm package 를 통해서 복잡한 구조의 어플리케이션도
초기에 빠른 설치를 할 수 있도록 helm chart를 제공해주기 때문에 온보딩에 대한 이점이 있습니다.

다만, helm chart를 잘 활용하기 위해서는 각 chart에서 제공되는 values 변수들에 대한 설정 값 이해가 필요합니다.

아래에서 각각의 디렉토리 내부 content를 확인해보도록 하겠습니다.

# 1. 디렉토리 구조 살펴보기

## Base Directory

해당 Directory 에는 chart 방식 중 `umbrella helm` 방식으로 외부 helm chart dependency 로 부터
helm chart를 가져와서 사용하도록 `Chart.yaml` 에 대한 정의가 되어있습니다.

#### Chart.yaml
```
apiVersion: v2   # helm version (helm 3를 위해서는 apiversion이 v2)
name: argo-cd    # chart name
version: 1.0.0   # chart version
dependencies:
  - name: argo-cd
    version: 3.6.8
    repository: https://argoproj.github.io/argo-helm
```

이 chart.yaml 파일에서 주목해볼 부분은 `dependencies` 부분입니다.

```
dependencies:
  - name: argo-cd    # chart의 이름
    version: 3.6.8   # chart의 버전
    repository: https://argoproj.github.io/argo-helm  # chart가 저장되어있는 helm repository
```

`umbrella helm` 방식에서는 이와 같이 외부 디펜던시를 지정해줄 수 있고, 이는 해당 helm 차트의
디펜던시 차트가 어떤 것이 있는지 알려주는 것입니다.

`Chart.yaml` 이 있는 경로에서 `helm dependency update` 를 하면 아래와 같이 종속성들이 install
되는 것을 확인할 수 있습니다.

```
$ heml dependency update (또는 helm dep up)

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "argo" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading argo-cd from repo https://argoproj.github.io/argo-helm
Deleting outdated charts
```

위와 같이 dependency 를 가져오면 아래와 같이 `base/` 하위에 파일들이 추가되는 것을 알 수 있습니다.

```
- base
  - charts/argo-cd-3.6.8.tgz (압축을 풀면 charts, templates 등이 들어있는 차트 디렉토리)
  - Chart.lock
  - Chart.yaml
```

* charts/argo-cd-3.6.8.tgz
    * chart 디펜던시의 압축을 로컬로 가져온 것입니다. 이 차트를 이용해 로컬에서 `helm install` 을 통해
      target kubernetes에 배포 할 수 있습니다.
    * argocd 에서는 자체적으로 dependecy를 install할 수 있기 때문에, git에서 디펜던시 압축본을   
      관리할 필요가 없습니다. 그래서 `.gitignore`에서 `*.tgz` 는 제외합니다.
    * (참고) 로컬에서 `helm install` 을 하면 helm의 template engine을 통해서 (helm template)
      pure manifest yaml 을 얻게 되고, 해당 manifest yaml을 `kubectl apply` 을 통해서 배포하게 됩니다.

* Chart.lock
    * `helm dependency update` 에 성공하면 생성되는 lock 파일입니다.
    * 이 lock 파일을 통해서 `charts/` 를 다시 re-build 하는데에 사용할 수 있습니다.
    * `helm dependency build` 명령어를 사용하면 `Chart.lock` 파일을 통해 `charts/` 종속성을 리빌드합니다.
    * argocd 에서는 `Chart.lock`의 버전과 `Chart.yaml` 에 대한 버전이 일치하지 않으면 Sync에 실패하므로, 
    `Chart.yaml`의 버전을 변경하면 `helm dependency update` 를 통해서 생성된 `Chart.lock` 버전을 함께 커밋해주어야 합니다.
      
## 환경별 변수 파일

---

환경별로 디렉토리 구조는 분리되어 있습니다. (`dev`, `prd`)

각 환경별로 `values-{env}.yaml` 파일이 존재하고 이 파일에는 `base/` 템플릿에 merge 될 변수들을
정의하게 됩니다.

`base/` + `values` 가 합쳐져서 우리가 원하는 `pure manifest` 템플릿이 완성되고 이 파일을 이용해서  
최종적으로 kubernetes cluster에 배포를 하게 됩니다.

`argocd-install` 내의 value 파일을 살펴보겠습니다.

### argocd-install-values-dev.yaml
```yaml
argo-cd: (1)
  server: 
    ## ArgoCD config
    ## reference https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-cm.yaml
    configEnabled: true (2)
    config:
      repositories: |  (3)
        - type: git
          url: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
        - name: argo-helm
          type: helm
          url: https://argoproj.github.io/argo-helm
```

* (1) argo-cd
  *  여기서 중요하게 알아야 할 점은 `argo-cd` 로 시작한다는 점입니다.
  *  현재 dependency chart를 사용하는 umbrella chart 구조이기 때문에, 이 경우에 values 를 주입하기 위해서는
     child-chart의 이름으로 시작하여야 values를 주입할 수 있습니다.
  * #### (참고) Chart.yaml
    ```yaml
    dependencies:
    - name: argo-cd
      version: ~3.6.8
      repository: https://argoproj.github.io/argo-helm
    ```
    chart yaml 파일을 살펴보면 name이 `argo-cd` 이기 때문에, 해당 이름을 사용하여
    value를 정의합니다.
    
* (2) server.configEnabled: true
  * argocd-cm manifest를 활성화 시키기 위한 플래그입니다.
  * argocd-cm manifest 는 `configMap` 으로 key, value 를 저장할 수 있습니다.
  * 이를 활성화 한 후, watch할 repository의 정보를 넣을 수 있습니다.

* (3) config.repositories
  * argocd가 watch할 repository에 대해서 추가합니다.
  * deployment repo로 `https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git` 를 추가합니다.
  * helm repo로 `https://argoproj.github.io/argo-helm` 를 추가합니다.
  * (참고) 만약 private repository 일 경우, 접근 가능한 user/password 정보를 넣거나, ssh 접근 또한 할 수 있습니다.
    private repository 에서 auth 관련 추가를 하지 않을 경우, repository 를 watch 할 수 없습니다.
    관련 내용은 공식 문서를 확인해보시면 됩니다. [Repository Credentials](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#repository-credentials)

```yaml
argo-cd:
  server :
  .
  .
  .
  additionalApplications:
  - name: argocd
    namespace: argocd
    project: argocd
    source:
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      path: argocd/argocd-install/dev/base
      helm:
        version: v3
        valueFiles:
          - ../argocd-install-values-dev.yaml
      targetRevision: HEAD
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      syncOptions:
        - CreateNamespace=true
  - name: argocd-apps
    namespace: argocd
    project: argocd
    source:
      path: argocd/argocd-apps
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      targetRevision: HEAD
      directory:
        recurse: true
        jsonnet: { }
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      automated:
        selfHeal: true
        prune: true
  - name: argocd-appprojects
    namespace: argocd
    project: argocd
    source:
      path: argocd/argocd-appprojects
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      targetRevision: HEAD
      directory:
        recurse: true
        jsonnet: {}
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      automated:
        selfHeal: true
        prune: true
```

 다음으로 `additionalApplications` 을 살펴보겠습니다. 이 부분은 초기에 argo CD가 구동되었을 때
선언적인 방식으로 SetUp 하기 위해서 Application을 만들어주는 부분입니다.

---
**NOTE) Application 이란?** 
* `Application` 은 ArgoCD 에서 배포를 관리하는 최소 단위로 `source`, `destination` 으로 이루어져있습니다.
* `source`의 repository path를 watch 하다가 매니페스트 변경이 일어나면 `destination` 으로 배포를 진행
  하게 됩니다.
  
---
  
 이 부분을 추가할 경우 ArgoCD가 시작되었을 때 `app of apps` 패턴이 셋팅되도록 초기 구성을 할 수 있습니다.
위에서 추가되는 ArgoCD Application 은 총 3개입니다.

---
**Note) App of Apps 패턴이란?**

* ArgoCD 에 Application 을 추가하여 다른 Application의 변경을 watch 하여 설정이 배포되도록 
하는 패턴을 `App of Apps` 이라고 합니다.
* 이때 다른 애플리케이션을 `Child Application` 이라고 하고 이를 watch하는 어플리케이션을 `Root Application` 이라고 합니다.
  
---
아래에서 각각의 Application 을 살펴보도록 하겠습니다.

### Self Management 를 위한 ArgoCD Application

* #### ArgoCd Application
  * argoCd 는 `self-management` 가 가능합니다. 이 말이 무엇이냐면 만약 argoCD의 버전을 업그레이드 하거나
    다운그레이드 할 때도 GitOps 방식으로 바라보는 매니페스트가 변경되면 ArgoCD 는 자기 스스로 버전을 업/다운 
    할 수 있습니다.
  * 이를 위해서 ArgoCD 자신의 Install Manifest 를 바라보는 `ArgoCD Application` 을 생성해야 합니다.
  * ```yaml
    - name: argocd
    namespace: argocd
    project: argocd
    source:
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      path: argocd/argocd-install/dev/base
      helm:
        version: v3
        valueFiles:
          - ../argocd-install-values-dev.yaml
      targetRevision: HEAD
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      syncOptions:
        - CreateNamespace=true
    ```
  * 위 매니페스트를 살펴보면, `path: argocd/argocd-install/dev/base` 를 `helm` 타입으로 바라보고 있습니다.
  * 그래서 만약 `argocd/argocd-install/dev/base` 에 있는 `Chart.yaml` 파일에 버전 변경이 발생하면 argoCD는
    이를 감지하고 자체적으로 업/다운그레이드를 진행하게 됩니다.
  * (예시)
    
    #### 버전변경 전 (3.6.8) 
    ```yaml
    apiVersion: v2
    name: argo-cd
    version: 1.0.0
    dependencies:
    - name: argo-cd
      version: 3.6.0 # 버전 변경 전
      repository: https://argoproj.github.io/argo-helm
    ```
    
    #### 버전변경 후 (3.6.0)
    ```yaml
    apiVersion: v2
    name: argo-cd
    version: 1.0.0
    dependencies:
    - name: argo-cd
      version: 3.6.8 # 버전 변경 
      repository: https://argoproj.github.io/argo-helm
    ```
     
    #### ArgoCD는 이를 감지하고 차트버전 3.6.0을 참조하여 argoCD 앱을 셀프 업그레이 하게 됩니다 (3.6.0 -> 3.6.8)

### ArgoCD의 Projects 를 선언적으로 관리 - App of Apps 패턴턴

---

**Note) ArgoCD - Project 란?**

* ArgoCD의 Application 들은 논리적인 그룹인 Project로 묶일 수 있습니다.
* `AppProject` 는 `Application` 들에게 적용될 access 룰들을 관리할 수 있습니다.
    * 예) `source`에 대한 제한, `destination` 에 `namespace` 제한 등..
* 기본적으로 `default` app project에 속하게 되며 이는 별다른 제한이 없이 모든 access 를 개방하는 정책입니다.

---

* app of apps 패턴으로 argocd의 project를 선언적으로도 관리할 수 있습니다.
* 
  ```yaml
  - name: argocd-appprojects
    namespace: argocd
    project: argocd
    source:
      path: argocd/argocd-appprojects
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      targetRevision: HEAD
      directory:
        recurse: true
        jsonnet: {}
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      automated:
        selfHeal: true
        prune: true
  ```
* `argocd/argocd-appprojects` path에 추가되는 AppProject manifest 들에 대해서 ArgoCD Project를 생성합니다.

### ArgoCD의 Application 을 선언적으로 관리 - App of Apps 패턴

- ```yaml
  - name: argocd-apps
    namespace: argocd
    project: argocd
    source:
      path: argocd/argocd-apps
      repoURL: https://github.com/heesuk-ahn/cicd-workflow-demo-deployment.git
      targetRevision: HEAD
      directory:
        recurse: true
        jsonnet: { }
    destination:
      namespace: argocd
      server: https://kubernetes.default.svc
    syncPolicy:
      automated:
        selfHeal: true
        prune: true
  ```
- `argocd/argocd-apps` 경로에 추가되는 `Application Manifest yaml` 들을 `ArgoCD` 에 적용합니다.
