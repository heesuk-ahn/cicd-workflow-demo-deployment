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
      관리할 필요가 없습니다. 그래서 `.gitignore`에서 `*.tgz` 는 제외합니
    * (참고) 로컬에서 `helm install` 을 하면 helm의 template engine을 통해서 (helm template)
      pure manifest yaml 을 얻게 되고, 해당 manifest yaml을 `kubectl apply` 을 통해서 배포하게 됩니다.

* Chart.lock
    * `helm dependency update` 에 성공하면 생성되는 lock 파일입니다.
    * 이 lock 파일을 통해서 `charts/` 를 다시 re-build 하는데에 사용할 수 있습니다.
    * `helm dependency build` 명령어를 사용하면 `Chart.lock` 파일을 통해 `charts/` 종속성을 리빌드합니다.
    * 이 또한 git에 저장할 필요가 없으므로 `.gitignore` 를 이용하여 git에 추가하지는 않습니다.

## 환경별 변수 파일

환경별로 디렉토리 구조는 분리되어 있습니다. (`dev`, `prd`)

각 환경별로 `values-{env}.yaml` 파일이 존재하고 이 파일에는 `base/` 템플릿에 merge 될 변수들을
정의하게 됩니다.

`base/` + `values` 가 합쳐져서 우리가 원하는 `pure manifest` 템플릿이 완성되고 이 파일을 이용해서  
최종적으로 kubernetes cluster에 배포를 하게 됩니다.

`argocd-install` 내의 value 파일을 살펴보겠습니다.

```argoCdVersion: 3.6.8

sever:
  configEnabled: true
  config:
    repositories: |
      - type: helm
        name: stable
        url: https://charts.helm.sh/stable
      - type: helm
        name: argo-cd
        url: https://argoproj.github.io/argo-helm
```