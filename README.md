# openshift-mtls-ingress-controller in AWS

This is an example of how to create an Openshift Ingress Controller with mTLS enabled in AWS.

- <https://docs.openshift.com/container-platform/4.9/networking/ingress-operator.html#nw-ingress-controller-configuration-parameters_configuring-ingress>
- <https://rcarrata.com/openshift/mtls-ingress-controller/>

## setup

```sh
export INGRESS_DOMAIN=mtls.$(oc get ingress.config.openshift.io cluster -o jsonpath={.spec.domain})
export TEST_APP_NAMESPACE=mytestproject
```

## install ingress controller

We use cert-manager to create a workload (client) certificate to trust.

```sh
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.7.1 --set installCRDs=true
helm upgrade -i certs helm/certs -n openshift-config

oc get secret workload-cert -n openshift-config -o jsonpath={.data.ca\\.crt} | base64 -d > /tmp/client-ca.crt

oc create configmap router-ca-certs-mtls --from-file=ca-bundle.pem=/tmp/client-ca.crt -n openshift-config

helm upgrade -i mtls-ingress-controller helm/mtls-ingress-controller \
  --set domain=${INGRESS_DOMAIN} \
   -n openshift-ingress-operator
```

## configure the default ingress controller

Adjust the default router to skip `type: mtls` routes with that label.

```sh
oc patch \
  -n openshift-ingress-operator \
  IngressController/default \
  --type='merge' \
  -p '{"spec":{"routeSelector":{"matchExpressions":[{"key":"type","operator":"NotIn","values":["mtls"]}]}}}'
```

## validate the ingress controller

From the docs...

```text
endpointPublishingStrategy is used to publish the Ingress Controller endpoints to other networks, enable load balancer integrations, and provide access to other systems.

If not set, the default value is based on infrastructure.config.openshift.io/cluster .status.platform:

AWS: LoadBalancerService (with external scope)

Azure: LoadBalancerService (with external scope)

GCP: LoadBalancerService (with external scope)

Bare metal: NodePortService

Other: HostNetwork

For most platforms, the endpointPublishingStrategy value cannot be updated. However, on GCP, you can configure the loadbalancer.providerParameters.gcp.clientAccess subfield.
```

Since this cluster is in AWS, you should see the following status in the mtls IngressController's status...

```sh
oc get ingresscontroller mtls -n openshift-ingress-operator -o jsonpath={.status.endpointPublishingStrategy} | jq
```

```json
{
  "loadBalancer": {
    "scope": "External"
  },
  "type": "LoadBalancerService"
}
```

Additionally, in the ingress operator pod, there should be logs showing that the external AWS loadbalancer service was configured...

```log
2022-04-29T18:22:51.636Z	INFO	operator.dns	aws/dns.go:505	updated DNS record	{"zone id": "Z01402822UWTV44PDZE3E", "domain": "*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.", "target": "a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com", "response": "{\n  ChangeInfo: {\n    Id: \"/change/C0676777CPZMOIQIN3K2\",\n    Status: \"PENDING\",\n    SubmittedAt: 2022-04-29 18:22:51.615 +0000 UTC\n  }\n}"}
2022-04-29T18:22:51.636Z	INFO	operator.dns	aws/dns.go:467	upserted DNS record	{"record": {"dnsName":"*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.","targets":["a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com"],"recordType":"CNAME","recordTTL":30}, "zone": {"tags":{"Name":"cluster-4lcpr-m8mzc-int","kubernetes.io/cluster/cluster-4lcpr-m8mzc":"owned"}}}
2022-04-29T18:22:51.636Z	INFO	operator.dns_controller	dns/controller.go:190	published DNS record to zone	{"record": {"dnsName":"*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.","targets":["a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com"],"recordType":"CNAME","recordTTL":30}, "dnszone": {"tags":{"Name":"cluster-4lcpr-m8mzc-int","kubernetes.io/cluster/cluster-4lcpr-m8mzc":"owned"}}}
2022-04-29T18:22:51.838Z	INFO	operator.dns	aws/dns.go:505	updated DNS record	{"zone id": "Z04791052DFJMW6BFGVQ5", "domain": "*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.", "target": "a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com", "response": "{\n  ChangeInfo: {\n    Id: \"/change/C0341369SLQCPPNL60W8\",\n    Status: \"PENDING\",\n    SubmittedAt: 2022-04-29 18:22:51.816 +0000 UTC\n  }\n}"}
2022-04-29T18:22:51.838Z	INFO	operator.dns	aws/dns.go:467	upserted DNS record	{"record": {"dnsName":"*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.","targets":["a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com"],"recordType":"CNAME","recordTTL":30}, "zone": {"id":"Z04791052DFJMW6BFGVQ5"}}
2022-04-29T18:22:51.838Z	INFO	operator.dns_controller	dns/controller.go:190	published DNS record to zone	{"record": {"dnsName":"*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.","targets":["a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com"],"recordType":"CNAME","recordTTL":30}, "dnszone": {"id":"Z04791052DFJMW6BFGVQ5"}}
2022-04-29T18:22:51.851Z	INFO	operator.dns_controller	controller/controller.go:298	updated dnsrecord	{"dnsrecord": {"metadata":{"name":"mtls-wildcard","namespace":"openshift-ingress-operator","uid":"51f194e4-808a-480d-864c-1a6374ab3381","resourceVersion":"39330","generation":1,"creationTimestamp":"2022-04-29T18:22:51Z","labels":{"ingresscontroller.operator.openshift.io/owning-ingresscontroller":"mtls"},"ownerReferences":[{"apiVersion":"operator.openshift.io/v1","kind":"IngressController","name":"mtls","uid":"7a57612d-bbe3-41e6-958f-b0d77ceaee83","controller":true,"blockOwnerDeletion":true}],"finalizers":["operator.openshift.io/ingress-dns"],"managedFields":[{"manager":"ingress-operator","operation":"Update","apiVersion":"ingress.operator.openshift.io/v1","time":"2022-04-29T18:22:51Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:finalizers":{".":{},"v:\"operator.openshift.io/ingress-dns\"":{}},"f:labels":{".":{},"f:ingresscontroller.operator.openshift.io/owning-ingresscontroller":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"7a57612d-bbe3-41e6-958f-b0d77ceaee83\"}":{}}},"f:spec":{".":{},"f:dnsName":{},"f:recordTTL":{},"f:recordType":{},"f:targets":{}}}},{"manager":"ingress-operator","operation":"Update","apiVersion":"ingress.operator.openshift.io/v1","time":"2022-04-29T18:22:51Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{".":{},"f:observedGeneration":{},"f:zones":{}}}}]},"spec":{"dnsName":"*.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com.","targets":["a67febb59bf8941fc81a06c8ee8537c0-77987723.us-east-2.elb.amazonaws.com"],"recordType":"CNAME","recordTTL":30},"status":{"zones":[{"dnsZone":{"tags":{"Name":"cluster-4lcpr-m8mzc-int","kubernetes.io/cluster/cluster-4lcpr-m8mzc":"owned"}},"conditions":[{"type":"Failed","status":"False","lastTransitionTime":"2022-04-29T18:22:51Z","reason":"ProviderSuccess","message":"The DNS provider succeeded in ensuring the record"}]},{"dnsZone":{"id":"Z04791052DFJMW6BFGVQ5"},"conditions":[{"type":"Failed","status":"False","lastTransitionTime":"2022-04-29T18:22:51Z","reason":"ProviderSuccess","message":"The DNS provider succeeded in ensuring the record"}]}],"observedGeneration":1}}}
2022-04-29T18:22:51.851Z	INFO	operator.dns_controller	controller/controller.go:298	reconciling	{"request": "openshift-ingress-operator/mtls-wildcard"}
```

## install test app

```sh
helm upgrade -i nginx-echo-headers helm/nginx-echo-headers \
  --set domain=${INGRESS_DOMAIN} \
  -n ${TEST_APP_NAMESPACE} --create-namespace
```

## validate the route

The nginx-echo-headers route in the ${TEST_APP_NAMESPACE} should have a status indicating the route is served using the correct ingress controller. The default router/controller should NOT be listed...

```sh
oc get route nginx-echo-headers -n ${TEST_APP_NAMESPACE} -o jsonpath={.status.ingress} | jq
```

```json
[
  {
    "conditions": [
      {
        "lastTransitionTime": "2022-04-29T23:44:15Z",
        "status": "True",
        "type": "Admitted"
      }
    ],
    "host": "nginx-echo-headers-mytestproject.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com",
    "routerCanonicalHostname": "router-mtls.mtls.apps.cluster-4lcpr.4lcpr.sandbox893.opentlc.com",
    "routerName": "mtls",
    "wildcardPolicy": "None"
  }
]
```

## test mtls

this should work...

```sh
oc get secret workload-cert -n openshift-config -o jsonpath={.data.tls\\.crt} | base64 -d > /tmp/tls.crt
oc get secret workload-cert -n openshift-config -o jsonpath={.data.ca\\.crt} | base64 -d > /tmp/ca.crt
oc get secret workload-cert -n openshift-config -o jsonpath={.data.tls\\.key} | base64 -d > /tmp/tls.key


curl -k -LIv https://nginx-echo-headers-${TEST_APP_NAMESPACE}.${INGRESS_DOMAIN} \
  --cacert /tmp/ca.crt \
  --cert /tmp/tls.crt \
  --key /tmp/tls.key

curl -k -LIv https://nginx-echo-headers-${TEST_APP_NAMESPACE}.${INGRESS_DOMAIN} \
  --cert /tmp/tls.crt \
  --key /tmp/tls.key
```

requesting without a client certificate should fail...

```sh
curl -k -LIv https://nginx-echo-headers-${TEST_APP_NAMESPACE}.${INGRESS_DOMAIN}
```

error output...

```sh
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
* Closing connection 0
curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

a client certificate with a Common Name that is not listed in the IngressController's spec.clientTLS.allowedSubjectPatterns should fail...

```sh
oc get secret workload-cert-unallowed -n openshift-config -o jsonpath={.data.tls\\.crt} | base64 -d > /tmp/tls-unallowed.crt
oc get secret workload-cert-unallowed -n openshift-config -o jsonpath={.data.ca\\.crt} | base64 -d > /tmp/ca-unallowed.crt
oc get secret workload-cert-unallowed -n openshift-config -o jsonpath={.data.tls\\.key} | base64 -d > /tmp/tls-unallowed.key

curl -kv https://nginx-echo-headers-${TEST_APP_NAMESPACE}.${INGRESS_DOMAIN} \
  --cert /tmp/tls-unallowed.crt \
  --key /tmp/tls-unallowed.key
```

error output...

```sh
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< content-length: 93
< cache-control: no-cache
< content-type: text/html
< 
<html><body><h1>403 Forbidden</h1>
Request forbidden by administrative rules.
</body></html>
```

## cleanup

```sh
helm delete nginx-echo-headers -n ${TEST_APP_NAMESPACE}
oc delete project ${TEST_APP_NAMESPACE}
helm delete mtls-ingress-controller -n openshift-ingress-operator
oc delete configmap router-ca-certs-mtls -n openshift-config
helm delete certs -n openshift-config
oc delete secret workload-cert -n openshift-config
oc delete secret workload-cert-unallowed -n openshift-config
oc delete secret workload-rootca -n openshift-config
```
