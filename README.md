# Claims Based Routing Istio Example
_You must use Istio 1.12 for this demo._
- [Setup MiniKube](#setup-minikube)
- [Install Istio](#install-istio)
- [Deploy Demo](#deploy-demo)
- [Setup Gateway, Virtual Service, and RequestAuthentication](#setup-gateway-virtual-service-and-requestauthentication)
- [Test](#test)
- [Teardown](#teardown)

## Setup Minikube
```
minikube start --memory=10000 --cpus=4 --kubernetes-version=v1.20.2
```

## Install Istio 
If you do not have `istioctl`, read this [doc](https://preliminary.istio.io/latest/docs/setup/getting-started/)
```
istioctl install --set profile=demo -y   
```

## Deploy Demo Application
We will have a Red service and a Blue service. The Red Service serves an html page that says "RED" and the Blue Service services and html page that says "BLUE". We will mount the `index.html` page in a ConfigMap on the pod.

```
# Label namespace for sidecar injection
kubectl label namespace default istio-injection=enabled

# Create blue.html
echo "<html><body>BLUE</body></html>" > blue.html

# Create red.html
echo "<html><body>RED</body></html>" > red.html

# Create a ConfigMap with index.html for Blue Pod
kubectl create cm blue --from-file=index.html=blue.html

# Create a ConfigMap with index.html for Red Pod
kubectl create cm red --from-file=index.html=red.html

# Create the  Blue Pod
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: blue
  name: blue
spec:
  containers:
  - image: nginx
    name: blue
    volumeMounts:
      - name: index
        mountPath: /usr/share/nginx/html
    ports:
    - containerPort: 80
    resources: {}
  volumes:
   - name: index
     configMap:
       name: blue
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF

# Create the  Red Pod
kubectl apply -f -<<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: red
  name: red
spec:
  containers:
  - image: nginx
    name: red
    volumeMounts:
      - name: index
        mountPath: /usr/share/nginx/html
    ports:
    - containerPort: 80
    resources: {}
  volumes:
   - name: index
     configMap:
       name: red
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF

# Expose the Blue Pod with a service
kubectl expose pod/blue --port=80 

# Expose the Red Pod with a service
kubectl expose pod/red --port=80
```

## Setup Gateway, Virtual Service, and RequestAuthentication
Create a Request Authentication to enable JWT Validation on the Gateway
```
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: ingress-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/master/security/tools/jwt/samples/jwks.json"
EOF
```

### Set Environmental Variables for Tokens
We will be using the Token on the VirtualService for routing.   
   
`TOKEN_GROUP` has group1 and group2 claims.    
   
`TOKEN_NO_GROUP` has subject as `testing@secure.istio.io` and no group claims.

```
TOKEN_GROUP=eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjM1MzczOTExMDQsImdyb3VwcyI6WyJncm91cDEiLCJncm91cDIiXSwiaWF0IjoxNTM3MzkxMTA0LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInNjb3BlIjpbInNjb3BlMSIsInNjb3BlMiJdLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.EdJnEZSH6X8hcyEii7c8H5lnhgjB5dwo07M5oheC8Xz8mOllyg--AHCFWHybM48reunF--oGaG6IXVngCEpVF0_P5DwsUoBgpPmK1JOaKN6_pe9sh0ZwTtdgK_RP01PuI7kUdbOTlkuUi2AO-qUyOm7Art2POzo36DLQlUXv8Ad7NBOqfQaKjE9ndaPWT7aexUsBHxmgiGbz1SyLH879f7uHYPbPKlpHU6P9S-DaKnGLaEchnoKnov7ajhrEhGXAQRukhDPKUHO9L30oPIr5IJllEQfHYtt6IZvlNUGeLUcif3wpry1R5tBXRicx2sXMQ7LyuDremDbcNy_iE76Upg

TOKEN_NO_GROUP=eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ2ODU5ODk3MDAsImZvbyI6ImJhciIsImlhdCI6MTUzMjM4OTcwMCwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.CfNnxWP2tcnR9q0vxyxweaF3ovQYHYZl82hAUsn21bwQd9zP7c-LS9qd_vpdLG4Tn1A15NxfCjp5f7QNBUo-KC9PJqYpgGbaXhaGx7bEdFWjcwv3nZzvc7M__ZpaCERdwU7igUmJqYGBYQ51vr2njU9ZimyKkfDe3axcyiBZde7G6dabliUosJvvKOPcKIWPccCgefSj_GNfwIip3-SsFdlR7BtbVUcqR-yv-XOxJ3Uc1MI0tz3uMiiZcyPV7sNCU4KRnemRIMHVOfuvHsU60_GhGbiSFzgPTAa9WTltbnarTbxudb_YEOx12JiwYToeX0DCPb43W1tzIBxgm8NxUg
```

### Create the Gateway and VirtualService
The VirtualService matches on the headers based on the JWT Claim.   
   
The `Blue` service receives requests from valid tokens with group1 in the groups claim.   

The `Red` service receives requests from valid tokens without group1 and with sub claim equal to `testing@secure.istio.io`

```
  http:
  - match:
     - headers:
        "@request.auth.claims.groups":
          exact: group1
    rewrite:
      uri: "/"
    route:
    - destination:
        host: blue
        port:
          number: 80
  - match:
     - headers:
        "@request.auth.claims.sub":
          exact: testing@secure.istio.io
    rewrite:
      uri: "/"
    route:
    - destination:
        host: red
        port:
          number: 80          
```

Apply the yaml for the gateway and the virtual service be issuing the command below 
```
kubectl apply -f -<<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: default-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: default
spec:
  hosts:
  - "*"
  gateways:
  - default-gateway
  http:
  - match:
     - headers:
        "@request.auth.claims.groups":
          exact: group1
    route:
    - destination:
        host: blue
        port:
          number: 80
  - match:
     - headers:
        "@request.auth.claims.sub":
          exact: testing@secure.istio.io
    route:
    - destination:
        host: red
        port:
          number: 80          
EOF
```

## Test
First, we have to set minikube to tunnel, this will set a loadbalancer for our `istio-ingressgateway` service, do this in a different tab.

```
minikube tunnel
```

Make sure your `istio-ingressgateway` is exposed as type `LoadBalancer`
```
kubectl get svc -n istio-system istio-ingressgateway
```

output:
```
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.109.81.43   127.0.0.1     15021:31419/TCP,80:30207/TCP,443:32001/TCP,31400:30402/TCP,15443:30106/TCP   15m
```

Set Environmental Variables for curling
```
INGRESS_HOST=127.0.0.1
INGRESS_PORT=80
```

Now, lets test, we will send a request to the Istio at `/`. We will route based on the JWT claims. If you claims contain group1, then we will go to the blue service, if they do not contain group1 and the claim contains subject equal to `testing@secure.istio.io` then we will be routed to the Red service
   
Lets curl the Blue Application by using the token with groups
```
curl "http://$INGRESS_HOST:$INGRESS_PORT" -H "Authorization: Bearer $TOKEN_GROUP"
```
**output**
```
<html>
    <body>BLUE</body>
</html>
```
Lets send another request to the exact same address, but lets use a token with no groups, and the subject set to `testing@secure.istio.io`. This should route us to the Red service.
```
curl "http://$INGRESS_HOST:$INGRESS_PORT" -H "Authorization: Bearer $TOKEN_NO_GROUP"
```
**output**
```
<html>
    <body>RED</body>
</html>
```

# Teardown
```
minikube delete
```