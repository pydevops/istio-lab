* [Table of Contents](#table-of-contents)
   * [Set Up](#set-up)
      * [Install istio](#install-istio)
      * [Verify istio installation](#verify-istio-installation)
      * [Install Grafana](#install-grafana)
      * [Install Prometheus](#install-prometheus)
      * [Install Kiali](#install-kiali)
   * [Authentication Policy Lab](#authentication-policy-lab)
      * [Permissive mTLS as default](#permissive-mtls-as-default)
      * [Enable STRICT mTLS mesh wide](#enable-strict-mtls-mesh-wide)
      * [Enable mTLS per namespace](#enable-mtls-per-namespace)
      * [Enable mTLS per workload](#enable-mtls-per-workload)
      * [Clean up](#clean-up)
   * [Mutual TLS Migration lab](#mutual-tls-migration-lab)
      * [Lock down mTLS by namespace](#lock-down-mtls-by-namespace)
      * [Lock down mTLS mesh wide](#lock-down-mtls-mesh-wide)
      * [Clean up](#clean-up-1)
## Set Up

### Install istio
```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.13.0/
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
kubectl -n istio-system get deploy
```

### Verify istio installation

```
$ kubectl -n istio-system get deploy
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
istio-egressgateway    1/1     1            1           22h
istio-ingressgateway   1/1     1            1           22h
istiod                 1/1     1            1           22h

$ istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote

$ istioctl profile dump demo
```

### Install Grafana
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/addons/grafana.yaml
# verify
kubectl -n istio-system get svc
```

### Install Prometheus
```
# verify
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/addons/prometheus.yaml
kubectl -n istio-system get svc
```

### Install Kiali
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/addons/kiali.yaml
# open the dashboard
istioctl dashboard kiali
```

## Authentication Policy Lab
* https://istio.io/latest/docs/tasks/security/authentication/authn-policy/

```
# all reachable by default
for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl -s "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done

# verify no pa or dr
kubectl get peerauthentication --all-namespaces
kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
```

### Permissive mTLS as default

All traffic between workloads with proxies uses mTLS
```
$ kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl -s http://httpbin.foo:8000/headers -s | grep X-Forwarded-Client-Cert | sed 's/Hash=[a-z0-9]*;/Hash=<redacted>;/'
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=<redacted>;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/sleep"
```

Plaintext workloads aren't using mTLS
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name})" -c sleep -n foo -- curl http://httpbin.legacy:8000/headers -s | grep X-Forwarded-Client-Cert
```

### Enable STRICT mTLS mesh wide
from root name space, default as *istio-system*

```
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT
EOF
```

Run the following test again, the plain text traffic are denied.
```
for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
```

Remove the peer authentication policy and test it agian.
```
kubectl delete peerauthentication -n istio-system default
```

### Enable mTLS per namespace

```
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
EOF
```


### Enable mTLS per workload 

```
cat <<EOF | kubectl apply -n bar -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "httpbin"
  namespace: "bar"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
EOF
```

### Clean up
```
kubectl delete peerauthentication -n istio-system default
```

## Mutual TLS Migration lab
* https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/


```
# all reachable by default
for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl -s "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done

# verify no pa or dr
kubectl get peerauthentication --all-namespaces
kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
```

### Lock down mTLS by namespace

```
kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```


### Lock down mTLS mesh wide

```
kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

### Clean up
```
kubectl delete peerauthentication -n istio-system default
```


## working with k8s ingress
* https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/

```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: istio
spec:
  controller: istio.io/ingress-controller
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: istio
  name: ingress
  namespace: foo
spec:
  rules:
  - host: httpbin.example.com
    http:
      paths:
      - path: /status/*
        backend:
          serviceName: httpbin
          servicePort: 8000
EOF
```


```
# verify the ingress using istio gateway
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')

curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```