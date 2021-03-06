# Purpose 

Demo the Apigee Adapter for Envoy on Kubernetes

# Prerequisites

Make sure you are pointed locally to a running Kubernetes cluster

## Google Kubernetes Engine (GKE)
```
gcloud container clusters get-credentials [cluster-name] --zone [zone-name]
```

# Provision Apigee Remote Service

1. Set Apigee credentials to env variables (*** USE YOUR OWN ***)

```console
export USER=ijustloveapis2020@gmail.com
export PASSWORD=iReallyLoveThem
export ORG=apinut-eval
export ENV=test
```

2. Provision the Apigee Remote Service in the cloud and gen local config 
```console
../apigee-remote-service-cli/apigee-remote-service-cli provision --legacy --username $USER --password $PASSWORD --organization $ORG --environment $ENV > config.yaml
```

# Create the API Product in Apigee that will control access

1. Click on API Products in the Apigee Management UI menu
   
2. Click the button to add a new API Product

3. Enter any intuitive name for the product (e.g. "HTTP Bin")

4. Select access as Private

5. Under 'API resources' add the proxy named `remote-service`

6. Under 'Apigee Remote Service Targets', click to add and enter the details as follows:
   1. Target name: `httpbin.org` (matches backend host)
   2. API proxy: `remote-service`
   3. Path: `/`

7. Add another path for access as `/` under 'API resources', so that both `/httpbin` and `/` then appear in the list.

8. Save the new API Product

# Register a client App in Apigee with access to the new API Product

1. Click on "Apps" in the Apigee Management UI menu and register a new app
2. Select the newly created API Product for access by the App
3. Save the new App
4. Grab the API Key for the newly registered App
5. Store the API Key to an environment variable at the command line
```console
export APIKEY=yourNewApiKeyHere
```

# Deploy the Apigee Remote Service on Kubernetes

1. Make sure you have a namespace set up on Kubernetes
```console
kubectl create namespace apigee
```

2. Create a K8s ConfigMap for the Apigee Remote Service using the config.yaml file that we generated with the CLI provisioning command.
```console
kubectl create configmap -n apigee apigee-remote-service-envoy --from-file=./config.yaml
```

3. Create the deployment and K8s service for the Apigee Remote Service 
```console
kubectl apply -f saas-apigee-remote-service-envoy.yaml
```

# Deploy the http service on Kubernetes

1. Transfer our local config file for the Envoy proxy to a K8s ConfigMap
```console
kubectl create configmap -n apigee standalone-envoy-config --from-file=./standalone-envoy-config.yaml
``` 

2. Create a standalone (non-sidecar) Deployment and Kubernetes Service for the Envoy proxy
```console
kubectl apply -f standalone-envoy-manifest.yaml
```

# Test the HTTP service

Before running the below, give time for the external IP to be acquired. You can check if it's available yet by running `kubectl get svc standalone-envoy -n apigee -o`.

1. Get the external IP address of the load balancer for the Kubernetes service
```console
export SVCIP=$(kubectl get svc standalone-envoy -n apigee -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
```

2. Confirm that your APIKEY and SVCIP environment variables are set
```console
echo $APIKEY
echo $SVCIP
```

3. Invoke the service via the Envoy proxy
```console
curl -i $SVCIP/httpbin/headers -H "x-api-key: $APIKEY"
```

# Sample Expected Output

```console
HTTP/1.1 200 OK
date: Tue, 21 Jul 2020 06:14:12 GMT
content-type: application/json
content-length: 797
server: envoy
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 213

{
  "headers": {
    "Accept": "*/*", 
    "Content-Length": "0", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.52.1", 
    "X-Amzn-Trace-Id": "Root=1-5f1687b5-2a97fdf121c56322af27e57d", 
    "X-Api-Key": "asd1f", 
    "X-Apigee-Accesstoken": "", 
    "X-Apigee-Api": "httpbin.org", 
    "X-Apigee-Apiproducts": "remote-service, HTTP Bin API", 
    "X-Apigee-Application": "HTTP Bin Client App", 
    "X-Apigee-Authorized": "true", 
    "X-Apigee-Clientid": "asd1f", 
    "X-Apigee-Developeremail": "ijustloveapis2020@gmail.com", 
    "X-Apigee-Environment": "prod", 
    "X-Apigee-Organization": "api-guru22", 
    "X-Apigee-Scope": "", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000", 
    "X-Envoy-Original-Path": "/httpbin/headers"
  }
}
```

# Deploy the gRPC service on Kubernetes

1. Transfer our local config file for the Envoy proxy to a K8s ConfigMap
```console
kubectl create configmap -n apigee grpc-envoy-config --from-file=./grpc-envoy-config.yaml
``` 

2. Create a standalone (non-sidecar) Deployment and Kubernetes Service for the Envoy proxy
```console
kubectl apply -f grpc-envoy-manifest.yaml
```

3. Create a Kubernetes Deployment and Service for the gRPC server
```console
kubectl apply -f grpc-server.yaml
```

# Test the gRPC service

1. Get the external IP address of the load balancer for the Kubernetes service
```console
export SVCIP=$(kubectl get svc grpc-envoy -n apigee -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
```

2. Confirm the IP address of the Kubernetes Service for gRPC and the API key are available
```console
echo $APIKEY
echo $SVCIP
```

3. Make sure the node.js dependencies are installed locally
```console
npm install
```

4. Invoke the service via the Envoy proxy
```console
node grpc-client.js $SVCIP $APIKEY
```

# Sample Expected Output
```
Greeting: Hello world
```
