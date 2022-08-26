# kcc: kube-credential-cache

[![lint](https://github.com/ryodocx/kube-credential-cache/actions/workflows/golangci-lint.yaml/badge.svg)](https://github.com/ryodocx/kube-credential-cache/actions/workflows/golangci-lint.yaml)
[![CodeQL](https://github.com/ryodocx/kube-credential-cache/actions/workflows/codeql-analysis.yml/badge.svg)](https://github.com/ryodocx/kube-credential-cache/actions/workflows/codeql-analysis.yml)
[![asdf-test](https://github.com/ryodocx/kube-credential-cache/actions/workflows/asdf-test.yml/badge.svg)](https://github.com/ryodocx/kube-credential-cache/actions/workflows/asdf-test.yml)
[![GoReleaser](https://github.com/ryodocx/kube-credential-cache/actions/workflows/goreleaser.yaml/badge.svg)](https://github.com/ryodocx/kube-credential-cache/actions/workflows/goreleaser.yaml)
[![Go Report Card](https://goreportcard.com/badge/github.com/ryodocx/kube-credential-cache)](https://goreportcard.com/report/github.com/ryodocx/kube-credential-cache)

Fast access to Kubernetes!
Especially effective with kubectl + EKS

## Architecture
![](./docs/summary.drawio.svg)

## Features
Work as caching proxy of [ExecCredential](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/#client-authentication-k8s-io-v1-ExecCredential) object, when use [client-go credential plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins) of Kubernetes. (e.g. kubectl)

- Caching
  - [x] Cache [ExecCredential](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/#client-authentication-k8s-io-v1-ExecCredential) object
  - [ ] Async credential refresh
- Cache file
  - [x] Concern Command, Args, Env as cache-key
  - [ ] Encryption
- kubeconfig
  - [x] kubeconfig optimizer (inject kcc-cache command automatically)
  - [x] kubeconfig recovery  (remove injected commands)

## Effects
A one of notable effect is, when used [`aws eks update-kubeconfig`](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html) to access EKS. about 500ms faster!

![](./benchmark/graph_eks.svg)

benchmark is [here](./benchmark/)

## Installation

```sh
# go install
go install github.com/ryodocx/kube-credential-cache/cmd/kcc-cache@latest
go install github.com/ryodocx/kube-credential-cache/cmd/kcc-injector@latest

# asdf-vm
asdf plugin add kube-credential-cache
```

or download from [releases](https://github.com/ryodocx/kube-credential-cache/releases)

## Usage(edit kubeconfig)

:running: install & just run `kcc-injector -i ~/.kube/config`

:ambulance: restore kubeconfig: `kcc-injector -i -r <your kubeconfig>`


<details>
<summary>manual setup</summary>
<p>

if manually edit kubeconfig,
  * set `kcc-cache` to command
  * original command move to args
  * :warning: **Do not use the same pattern for command, args and env**
    * :warning:U sing the same pattern presents the risk of mixing up credentials
    * :warning: env is ignored if not in `KUBE_CREDENTIAL_CACHE_CACHEKEY_ENV_LIST`

EKS (same effect as `kcc-injector -i <your kubeconfig>`)

```diff
kind: Config
apiVersion: v1
clusters: [...]
contexts: [...]
current-context: <your-current-context>
preferences: {}
users:
  - name: user-name
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
-       command: aws
+       command: kcc-cache
        args:
+         - aws
          - --region
          - <your-region>
          - eks
          - get-token
          - --cluster-name
          - <your-cluster>
        env:
          - name: AWS_PROFILE
            value: <your-profile>
```

EKS with [aws-vault](https://github.com/99designs/aws-vault)

```diff
kind: Config
apiVersion: v1
clusters: [...]
contexts: [...]
current-context: <your-current-context>
preferences: {}
users:
  - name: user-name
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
-       command: aws
+       command: kcc-cache
        args:
+         - aws-vault
+         - exec
+         - <your-profile>
+         - --
+         - aws
          - --region
          - <your-region>
          - eks
          - get-token
          - --cluster-name
          - <your-cluster>
-       env:
-         - name: AWS_PROFILE
-           value: <your-profile>
```

kubeconfig specification
* https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
* https://pkg.go.dev/k8s.io/client-go/tools/clientcmd/api/v1#Config

</p>
</details>

## Configration

### kcc-cache

| Environment variable                    | default                                                                                                                                                                                                                                        | description                                        |
|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| KUBE_CREDENTIAL_CACHE_FILE              | macOS:</br>`~/Library/Caches/kube-credential-cache/cache.json`</br>Linux:</br>`$XDG_CACHE_HOME/kube-credential-cache/cache.json`</br>`~/.cache/kube-credential-cache/cache.json`</br>Windows:</br>`%AppData%\kube-credential-cache\cache.json` | path of Cache file                                 |
| KUBE_CREDENTIAL_CACHE_REFRESH_MARGIN    | `30s`                                                                                                                                                                                                                                          | margin of credential refresh                       |
| KUBE_CREDENTIAL_CACHE_CACHEKEY_ENV_LIST | `KUBE_CREDENTIAL_CACHE_USER,AWS_PROFILE,AWS_REGION,AWS_VAULT`                                                                                                                                                                                  | comma separated env names for additional cache-key |

### kcc-injector

```sh
$ kcc-injector -h
Usage: kcc-injector [flags] <kubeconfig filepath>
  -c string
        injection command (default "kcc-cache")
  -i    edit file in-place
  -r    restore kubeconfig to original
```
