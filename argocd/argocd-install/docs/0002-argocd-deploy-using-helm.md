
## Helm Install 을 통한 배포

---

최초로 target kubernetes cluster에 배포할 때는 ArgoCD를 통해서 배포할 수 없기 때문에
`helm install` 커맨드를 통해서 배포를 진행해야 합니다.

```
$ cd argocd/argocd-install/dev/base
$ helm install argocd . \
--namespace=argocd \
--create-namespace \
-f ../argocd-install-values-dev.yaml
```

위의 커맨드를 사용하면 custom 하게 지정한 values 파일을 주입하여서 install을 진행할 수 있습니다.

```
$ kubectl get -n argocd pods

NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-77475b4f46-dhs86   1/1     Running   0          47h
argocd-dex-server-696cd79867-r9wqh               1/1     Running   0          47h
argocd-redis-6449cf85c9-wz48r                    1/1     Running   0          47h
argocd-repo-server-57f8c59c6b-phthh              1/1     Running   0          47h
argocd-server-88545d46b-9kmpl                    1/1     Running   1          47h
```

위와 같이 Pod이 모두 Running 상태이면 이제 ArgoCD Dashboard 에 접속할 수 있습니다.

현재 argocd 를 노출시키는 service mode가 ClsterIP 형태이므로 로컬에서 접속할 수 있도록 
`port-forwarding`을 하여야 합니다.

```
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
```

로컬에서 접속할 수 있도록 `8080`로 노출시킵니다.

이제 브라우저에서 `localhost:8080` 으로 접속합니다.

그러면 로그인 화면이 나오게 되는데, 이때 초기 password 는 아래와 같은 커맨드를 통해서 얻을 수 있습니다.

```
$ kubectl -n argocd get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

gTZKgpgUkpUEgXdk
```

초기 비밀번호를 지속적으로 사용하기 보다는 argocd cli 를 통해서 비밀번호를 변경하는 것이 권장됩니다.

초기 아이디는 `admin` 이며 password는 위에서 얻은 값을 넣어주면 됩니다.


