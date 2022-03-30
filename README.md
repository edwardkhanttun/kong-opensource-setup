# Kong Gateway Opensource Version Setup

This repository is used to deploy Kong community version to SME with the plugin OIDC enabled.

## Create custom kong docker image with OIDC plugin enabled. This step can be skipped if kong base image upgrade is not needed.

1. Update kong-oidc/docker file to customize the docker image if necessary.

```
# Update base image accordingly if based kong image upgrade is needed.
FROM kong:2.5.0-alpine	

LABEL description="Alpine + Kong 2.5.0 + kong-oidc plugin"

USER root
RUN apk update && apk add git unzip luarocks
RUN luarocks install --pin lua-resty-jwt 0.2.2-0
RUN luarocks install kong-oidc

USER kong
```

2. Build the customized docker image

```
docker build -t 172.20.105.74/tr-dis/kong-oidc:2.5.0 .
```

3. Upload the built docker image to Mirantis image registry or docker hub.

```
docker login 172.20.105.74
```

```
docker push 172.20.105.74/tr-dis/kong-oidc:2.5.0
```

## Install Kong Gateway with the customized image using Helm

1. Create namespace

```
kubectl create namespace kong
```

2. Add the Kong charts repository:

```
helm repo add kong https://charts.konghq.com
```

3. Update Helm:

```
helm repo update
```

4. Install kong open source version with the values.yaml file.

```
helm install my-kong kong/kong -n kong --values ./values.yaml
```

Note: in the values.yaml file line 76, it enables oidc plugin in the Kong API gateway deployment.

```
71 env:
72   prefix: /kong_prefix/
73   database: postgres
74   pg_user: kong
75   pg_password: CloudEnabled
76   plugins: oidc
77   admin_api_uri: http://172.20.105.63:32424
```

In the values.yaml file line 103 and 104, it specifies the customized image in the Mirantis image registry.

```
101 # Specify Kong's Docker image and repository details here
102 image:
103   repository: 172.20.105.74/tr-dis/kong-opensource>
104   tag: "2.5.0"
105   # Kong Enterprise
106   # repository: kong/kong-gateway
107   # tag: "2.7.0.0-alpine"
```

5. Get Kong admin API NodePort number from the command below.

```
[root@st-mct-cplb kong-opensource]# kubectl -n kong get svc
NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                         AGE
my-kong-kong-admin            NodePort       10.108.83.170   <none>           8001:32424/TCP,8444:31667/TCP   4d17h
my-kong-kong-proxy            NodePort       10.101.53.87    <node>           80:31103/TCP,443:32326/TCP      4d17h
my-kong-postgresql            ClusterIP      10.102.228.20   <none>           5432/TCP                        4d17h
my-kong-postgresql-headless   ClusterIP      None            <none>           5432/TCP                        4d17h
```

6. Update values.yaml file line 77 with correct admin API endpoint from last step.

```
 71 env:
 72   prefix: /kong_prefix/
 73   database: postgres
 74   pg_user: kong
 75   pg_password: CloudEnabled
 76   plugins: oidc
 77   admin_api_uri: http://172.20.105.63:32424
 78   #admin_gui_url: https://172.20.105.71:35171
```

7. Run the helm update command below to update the installation.

```
helm upgrade my-kong kong/kong -n kong --values ./values.yaml
```

8. Change the kong proxy service from NodePort to ingress to get external IP.

```
[root@st-mct-cplb kong-opensource]# kubectl -n kong get svc
NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                         AGE
my-kong-kong-admin            NodePort       10.108.83.170   <none>           8001:32424/TCP,8444:31667/TCP   4d17h
my-kong-kong-proxy            LoadBalancer   10.101.53.87    172.20.105.225   80:31103/TCP,443:32326/TCP      4d17h
my-kong-postgresql            ClusterIP      10.102.228.20   <none>           5432/TCP                        4d17h
my-kong-postgresql-headless   ClusterIP      None            <none>           5432/TCP                        4d17h
```

9. Verify OIDC plugin is enabled in the installation.

```
> curl -s http://172.20.105.63:32424ss | jq .plugins.available_on_server.oidc
```

This should return true. Install the tool jq if not install in the VM yet.


## Create ingress using Kong ingress controller.

1. Create ingress using ingress.yaml file

```
kubectl create -f ingress.yaml
```

2. Enable https in the ingress (Configure an HTTPS Redirect).

Update ingress annotations to HTTPS and issue a 301 redirect with this command to patch the ingress entry to tell Kong to redirect all the HTTP requests. The self-signed certificate generated by kong will be used as https certificate.

```
kubectl patch ingress sample-angular -p '{"metadata":{"annotations":{"konghq.com/protocols":"https","konghq.com/https-redirect-status-code":"301"}}}'
```

## Integrate with Keycloak server using OIDC plugin

1. Setup a sample client in Keycloak. In this sample, a client kong is setup in kong realm.

2. Extract the info below from keycloak and update the sample oidc plugin manifest kong-oidc-plugin.yaml.

```
  client_id: kong # Realm
  client_secret: 47c84a80-cb38-467c-ad16-da4db3589b58  # Client Secret Copied
  realm: demo
  discovery: https://172.20.105.225/auth/realms/demo/.well-known/openid-configuration # OpenID Endpoint Configuration Copied
  scope: openid
  redirect_after_logout_uri : https://172.20.105.225/auth/realms/demo/protocol/openid-connect/logout?redirect_uri=https://172.20.105.225/sample-angular/
```

3. Create the kong oidc pluging by the command below and verify the kong oidc plugin object is created successfully.

```
kubectl apply -f kong-oidc-plugin.yaml

[root@st-mct-cplb kong-opensource]# kubectl get kp
NAME   PLUGIN-TYPE   AGE
oidc   oidc          12d
```

4. Update the ingress created previously to enable kong oidc plugin.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    konghq.com/strip-path: "true"
    kubernetes.io/ingress.class: kong
    konghq.com/plugins: request-transformer,oidc #indicate here
  name: angular-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: sample-angular
            port:
              number: 80
        path: /sample-angular/
        pathType: Prefix
```

5. Test the website with OIDC. 

![Alt text](./images/image-20211027142953063.png?raw=true "Title")

## Reference:
1. https://dev.to/robincher/securing-your-site-via-oidc-powered-by-kong-and-keycloak-2ccc
2. https://dzone.com/articles/kubernetes-full-stack-example-with-kong-ingress-co
3. https://www.jerney.io/secure-apis-kong-keycloak-1/

