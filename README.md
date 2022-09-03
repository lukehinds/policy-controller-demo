# sigstore policy controller demo

This demo will show you how to use the sigstore policy controller to verify
keyless sigstore signatures.

Five tools are required to run this demo:
- [cosign](https://github.com/sigstore/cosign)
- [cosign-policy-controller](https://github.com/sigstore/policy-controller) (installed as part of the demo later on)
- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://github.com/kubernetes/kubernetes)
- [helm](https://helm.sh/)


Grab the nginx image:

```shell
docker pull nginxdemos/nginx-hello
```

Tag to images:

```shell
docker tag nginxdemos/nginx-hello ghcr.io/{$GITHUB_USERNAME}/nginx:signed

docker tag nginxdemos/nginx-hello ghcr.io/{$GITHUB_USERNAME}/nginx2:unsigned

docker push ghcr.io/{$GITHUB_USERNAME}/nginx:signed

docker push ghcr.io/{$GITHUB_USERNAME}/nginx2:unsigned
```

Create a kind cluster:

```shell
kind create cluster --name sigstore-demo --config manifests/kind-config.yaml
```
```shell
kubectl config use-context kind-sigstore-demo
```

Apply manifests:

```shell
kubectl apply -f manifests/base
```

```shell
kubectl apply -f manifests/ingress-controller
```

Setup the policy controller helm chart and install:

```shell
helm repo add sigstore https://sigstore.github.io/helm-charts
```

```shell
helm repo update
```

```shell
helm install policy-controller -n sigstore-system sigstore/policy-controller
```

Wait for ready STATUS
    
```shell
$ kubectl get pod -n sigstore-system
NAME                                                READY   STATUS    RESTARTS   AGE
policy-controller-policy-webhook-767d96744d-n2tf4   1/1     Running   0          59s
policy-controller-webhook-55587bc76d-v5j9w          1/1     Running   0   
```

Apply the image policy, but first set the email account you plan to sign with:

```shell
  - keyless:
        identities:
          - issuer: https://accounts.google.com
            subject: jdoe@gmail.com
```

```shell
$ kubectl apply -f manifests/imagePolicy.yaml
```

Ensure webhooks are set correctly:


```shell
$ kubectl get validatingwebhookconfiguration
NAME                                         WEBHOOKS   AGE
ingress-nginx-admission                      1          52m
policy.sigstore.dev                          1          79s
validating.clusterimagepolicy.sigstore.dev   1          79s

$ kubectl get mutatingwebhookconfiguration
NAME                                         WEBHOOKS   AGE
defaulting.clusterimagepolicy.sigstore.dev   1          83s
policy.sigstore.dev     

```

Ensure the policy is applied to the correct namespace
    
```shell
$ kubectl get ns nginx-ns1 -o jsonpath='{.metadata.labels}'
{"kubernetes.io/metadata.name":"nginx-ns1","policy.sigstore.dev/include":"true"}
```

# Deploy images

Sign the image with cosign

```shell
cosign sign ghcr.io/{$GITHUB_USERNAME}/nginx:signed
```

```shell
$ helm install nginx-signed --atomic -n nginx-ns1 ./manifests/nginx-demo -f manifests/nginx-signed.yaml

NAME: nginx-signed
LAST DEPLOYED: Fri Sep  2 18:55:28 2022
NAMESPACE: nginx-ns1
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

```shell
helm install nginx-unsigned --atomic -n nginx-ns2 ./manifests/nginx-demo -f manifests/nginx-unsigned.yaml
```
