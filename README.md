# sigstore policy controller demo

This demo will show you how to use the sigstore policy (admission) controller
to verify keyless sigstore signatures.

## Huh, what the hell is keyless?

Keyless is the slight of hand we use to perform crypto signing with no need for
long term key management.

The flow is in general terms:

An ecdsa key pair is generated (encoded to memory, never to touch any disk).

A user connects to sigstores fulcio CA to request a signing certificate. 

Fulcio directs them to an OpenID connect provider of their choosing (such as Github,
Google, Microsoft etc).

If the user auths correctly, an ID Token is provided.

The user then presents their OAuth2 ID Token to the sigstore fulcio CA and
requests a X509 certificate.

The certificate contains:

* The Public Key of the ephemeral key pair.
* The email address (stuck in a SAN)
* It has a chain to the sigstore root CA

The user then signs an artifact (container in this instance), and the digest / x509
cert are stored in sigstore rekor (transparency log)

We can then drop the private key, as that signing event is effectively frozen
in time. 

We can now look at the transparency log and know that:

At X time, a user with control of X OpenID account, was in possession of X key
pair, and X key pair signed X container digest.

Using the policy controller, we can then allow / deny an image, based on which
OAuth2 account signed the image.

As a follow up, I will add details of how we can use this for a sort of code
provenance using GitHubs OpenID ambient credentials. 

## Prerequisites 

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
ingress-sigstore-admission                      1          52m
policy.sigstore.dev                          1          79s
validating.clusterimagepolicy.sigstore.dev   1          79s

$ kubectl get mutatingwebhookconfiguration
NAME                                         WEBHOOKS   AGE
defaulting.clusterimagepolicy.sigstore.dev   1          83s
policy.sigstore.dev     

```

Ensure the policy is applied to the correct namespace
    
```shell
$ kubectl get ns sigstore-ns1 -o jsonpath='{.metadata.labels}'
{"kubernetes.io/metadata.name":"nginx-ns1","policy.sigstore.dev/include":"true"}
```

# Deploy images

Sign the image with cosign

```shell
cosign sign ghcr.io/{$GITHUB_USERNAME}/nginx:signed
```

Attempt to deploy the signed version (should succeed)

```shell
$ helm install cosign-action --atomic -n sigstore-ns1 ./manifests/sigstore-demo -f manifests/cosign-action-image.yaml

NAME: nginx-signed
LAST DEPLOYED: Fri Sep  2 18:55:28 2022
NAMESPACE: nginx-ns1
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Attempt to deploy the unsigned version (should fail)


```shell
$ helm install nginx-unsigned --atomic -n nginx-ns1 ./manifests/nginx-demo -f  manifests/cosign-action-image.yaml

Error: INSTALLATION FAILED: release nginx-unsigned failed, and has been uninstalled due to atomic being set: admission webhook "policy.sigstore.dev" denied the request: validation failed: failed policy: sigstore-demo: spec.template.spec.containers[0].image
ghcr.io/lukehinds/nginx2@sha256:32d4567494509b13a40899885dfbee46cec32ce918e39125161ac4de8337339c signature keyless validation failed for authority authority-0 for ghcr.io/lukehinds/cosign-keyless-action@sha256:32d4567494509b13a40899885dfbee46cec32ce918e39125161ac4de8337339c: no matching signatures:
```

Altenatively, you can use a different email address to sign the unsigned image,
which should also fail

```shell
& helm install nginx-unsigned --atomic -n nginx-ns1 ./manifests/nginx-demo -f manifests/nginx-unsigned.yaml

Error: INSTALLATION FAILED: release nginx-unsigned failed, and has been uninstalled due to atomic being set: admission webhook "policy.sigstore.dev" denied the request: validation failed: failed policy: sigstore-demo: spec.template.spec.containers[0].image
ghcr.io/lukehinds/nginx2@sha256:32d4567494509b13a40899885dfbee46cec32ce918e39125161ac4de8337339c signature keyless validation failed for authority authority-0 for ghcr.io/lukehinds/nginx2@sha256:32d4567494509b13a40899885dfbee46cec32ce918e39125161ac4de8337339c: no matching signatures:
none of the expected identities matched what was in the certificate
```

### credits

A lot of the k8s configs was lifted from [here](https://medium.com/@slimm609/image-signing-validation-on-k8s-4b3202dbcd6c), so thanks due to @slimm609 / Brian Davis.
