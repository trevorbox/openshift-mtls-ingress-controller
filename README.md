# openshift-mtls-ingress-controller

This is an example of how to create an Openshift Ingress Controller with mTLS enabled.


- <https://docs.openshift.com/container-platform/4.9/networking/ingress-operator.html#nw-ingress-controller-configuration-parameters_configuring-ingress>
- <https://rcarrata.com/openshift/mtls-ingress-controller/>

## setup

```sh
export INGRESS_DOMAIN=mtls.$(oc get ingress.config.openshift.io cluster -o jsonpath={.spec.domain})
```

## install ingress controller

We use cert-manager to create a workload (client) certificate to trust.

```sh
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.7.1 --set installCRDs=true
helm upgrade -i certs helm/certs -n openshift-config

oc get secret workload-cert -n openshift-config -o jsonpath={.data.ca\\.crt} | base64 -d > /tmp/client-ca.crt
oc create configmap router-ca-certs-default --from-file=ca-bundle.pem=/tmp/client-ca.crt -n openshift-config

helm upgrade -i mtls-ingress-controller helm/mtls-ingress-controller \
  --set domain=${INGRESS_DOMAIN} \
   -n openshift-ingress-operator
```

## install test app

```sh
helm upgrade -i nginx-echo-headers helm/nginx-echo-headers \
  --set domain=${INGRESS_DOMAIN} \
  -n nginx --create-namespace
```

## test mtls

this should work...

```sh
oc get secret workload-cert -n openshift-config -o jsonpath={.data.tls\\.crt} | base64 -d > /tmp/tls.crt
oc get secret workload-cert -n openshift-config -o jsonpath={.data.ca\\.crt} | base64 -d > /tmp/ca.crt
oc get secret workload-cert -n openshift-config -o jsonpath={.data.tls\\.key} | base64 -d > /tmp/tls.key


curl -k -LIv https://nginx-echo-headers-nginx.${INGRESS_DOMAIN} \
  --cacert /tmp/ca.crt \
  --cert /tmp/tls.crt \
  --key /tmp/tls.key
```

this should fail...

```sh
curl -k -LIv https://nginx-echo-headers-nginx.${INGRESS_DOMAIN}
```

error output...

```sh
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
* Closing connection 0
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

## cleanup

```sh
helm delete nginx-echo-headers -n nginx
oc delete project nginx
helm delete mtls-ingress-controller -n openshift-ingress-operator
```
