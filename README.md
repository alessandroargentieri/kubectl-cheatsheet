## Get and change the kubernetes context from kubectl

Kubectl can have more kubernetes clusters linked into the `~/.kube/config` file.
You can change the cluster with the following:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context_name>
```

## Imperative deploy

You can proceed to use an imperative deploy and later on get the respective declarative yaml manifests.

```bash
# create deployment from docker image 'alpine' by passing a custom command (or else it exits immediately)
kubectl create deployment test --image alpine -- sh -c 'echo "hello!"; tail -f /dev/null'
# scaling replicas (you can do it only later with the imperative way!)
kubectl scale deployment test --replicas 3

# create deployment from docker image exporting a port
kubectl create deployment webserver --image nginx --port=80
# exposing deployment through a service
kubectl expose deployment webserver --name=webserver-lb --port=80 --target-port=80 --protocol=TCP --type=NodePort

# port forwarding the service and curl it from localhost
kubectl port-forward service/webserver-lb 8081:80
curl http://localhost:8081

# or, alternatively, run a temporary pod to curl the service from inside the cluster
kubectl run -it shell --image=curlimages/curl --restart=Never --rm -- sh
# keep in mind that, inside the cluster, the services can be accessed by their internal DNS names, so you can run:
curl http://webserver-lb.default.svc.cluster.local:80 # http://<svc_name>.<namespace>.svc.cluster.local:<svc_port>
```
You can get the yaml manifest to be applied in another context with these:

```bash
kubectl get deployments webserver -o yaml > webserver_deployment.yaml
kubectl get services webserver-lb -o yaml > webserver_service.yaml
```

## Get the exported port for a pod

You can know what port is exposed by a pod (and then port forward on it) this way:

```bash
kubectl get pods [YOUR_PODE_NAME] --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
kubectl port-forward [YOUR_PODE_NAME] [LOCALHOST_PORT]:[POD_PORT]
```

## CURL instead of kubectl

Kubectl hides https API calls. The required credentials are stored in `~/.kube/config` file.

### Certificate data embedded in `~/.kube/config`

If you have, for a cluster:

In `~/.kube/config`:

```yaml
  certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTlRFd05UZ3hNekF3SGhjTk1qSXdOREkzTVRFeE5UTXdXaGNOTXpJd05ESTBNVEV4TlRNdwpXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTlRFd05UZ3hNekF3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFSRFJTeldIVjk0Q2I2Ri9tcUZYb3VGTzFoazM0aEhONWVSZlg1ekpFcWUKcVVRcWNzMTVOVnFzc0N0UVpmU0k4RHpvemk2NFlTRjNEQ00rbWJlQlFYUDZvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVXgrZ3lqVXFSRzFQcERybFo4cmIxClY2eTFzb2d3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUlnS043UXdHcElKcnJldDRHSmNKbG55ditXcTJjZXRQb0UKWmVQZ1g2Z3ZiWFFDSVFDckg2SG03NlA2dXRuTnBwaERtcDFrNHIvc2xOWG1tK2ZVbGZQVzVlME5QQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  server: https://0.0.0.0:62575
  client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJrVENDQVRlZ0F3SUJBZ0lJQUl4NUI1VWF0cE13Q2dZSUtvWkl6ajBFQXdJd0l6RWhNQjhHQTFVRUF3d1kKYXpOekxXTnNhV1Z1ZEMxallVQXhOalV4TURVNE1UTXdNQjRYRFRJeU1EUXlOekV4TVRVek1Gb1hEVEl6TURReQpOekV4TVRVek1Gb3dNREVYTUJVR0ExVUVDaE1PYzNsemRHVnRPbTFoYzNSbGNuTXhGVEFUQmdOVkJBTVRESE41CmMzUmxiVHBoWkcxcGJqQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDlBd0VIQTBJQUJQQWU1NFU1TVpXNGNTYXYKOE1iOVZXN1IzYkFqZWFMT2ErU0RLaU5VNzhHQ2tNSVRxanQ3MHJ0Umo2bTdVOEFDS09JeU13ZFFzOHQ0ejI0VQplSWkxYnJhalNEQkdNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBakFmCkJnTlZIU01FR0RBV2dCUmQvOGZibFJDNGg3U2hXRjBxcGs2MFRSU3I2ekFLQmdncWhrak9QUVFEQWdOSUFEQkYKQWlFQXZxUHh1VUdiam5nYnJlMFZkTnFpbFRsUDF2WEx1elBEb0c2M0RlQlJRc1FDSUdZcU5qeDd6MjNvNTVwago3c3p0aFZ3OXNCVDh4T3RjVGNLYjMyWnlKdG9oCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0KLS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdFkyeHAKWlc1MExXTmhRREUyTlRFd05UZ3hNekF3SGhjTk1qSXdOREkzTVRFeE5UTXdXaGNOTXpJd05ESTBNVEV4TlRNdwpXakFqTVNFd0h3WURWUVFEREJock0zTXRZMnhwWlc1MExXTmhRREUyTlRFd05UZ3hNekF3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFTVW5jWHladGRqQlNVcjFyWjQ5SkJrU09rTVpmQnVtVFVoUndONnVLTUoKS2F2YU42bUFaaGtVQkh2VjZJMGVVMktVVldjekc2c3o0VDEvT1I0M3p2ZnNvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVVhmL0gyNVVRdUllMG9WaGRLcVpPCnRFMFVxK3N3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUlnUmw2NDBGYXVmRGdBdWJXc1BFLzNYMkxUZVdtM2paY2gKWEZMWUVHZ2xQM1VDSVFDTERUR1pIcE5UMmZJM1R1dS90cmQyWjNpVG9scFpnQ1pGeTE3RXRYN1NHZz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  client-key-data: LS0tLS1CRUdJTiBFQyBQUklWQVRFIEtFWS0tLS0tCk1IY0NBUUVFSUhyNnAwQ3pXU2VoOUVTRkNvTmVjcy9XbVJ3WHJhOWR2NWcrc2ZkOEN5dENvQW9HQ0NxR1NNNDkKQXdFSG9VUURRZ0FFOEI3bmhUa3hsYmh4SnEvd3h2MVZidEhkc0NONW9zNXI1SU1xSTFUdndZS1F3aE9xTzN2Uwp1MUdQcWJ0VHdBSW80akl6QjFDenkzalBiaFI0aUxWdXRnPT0KLS0tLS1FTkQgRUMgUFJJVkFURSBLRVktLS0tLQo=
```

you can transform those data in files:

```bash
echo "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdGMyVnkKZG1WeUxXTmhRREUyTlRFd05UZ3hNekF3SGhjTk1qSXdOREkzTVRFeE5UTXdXaGNOTXpJd05ESTBNVEV4TlRNdwpXakFqTVNFd0h3WURWUVFEREJock0zTXRjMlZ5ZG1WeUxXTmhRREUyTlRFd05UZ3hNekF3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFSRFJTeldIVjk0Q2I2Ri9tcUZYb3VGTzFoazM0aEhONWVSZlg1ekpFcWUKcVVRcWNzMTVOVnFzc0N0UVpmU0k4RHpvemk2NFlTRjNEQ00rbWJlQlFYUDZvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVXgrZ3lqVXFSRzFQcERybFo4cmIxClY2eTFzb2d3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUlnS043UXdHcElKcnJldDRHSmNKbG55ditXcTJjZXRQb0UKWmVQZ1g2Z3ZiWFFDSVFDckg2SG03NlA2dXRuTnBwaERtcDFrNHIvc2xOWG1tK2ZVbGZQVzVlME5QQT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K" \
   | base64 -d > cacert

echo "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJrVENDQVRlZ0F3SUJBZ0lJQUl4NUI1VWF0cE13Q2dZSUtvWkl6ajBFQXdJd0l6RWhNQjhHQTFVRUF3d1kKYXpOekxXTnNhV1Z1ZEMxallVQXhOalV4TURVNE1UTXdNQjRYRFRJeU1EUXlOekV4TVRVek1Gb1hEVEl6TURReQpOekV4TVRVek1Gb3dNREVYTUJVR0ExVUVDaE1PYzNsemRHVnRPbTFoYzNSbGNuTXhGVEFUQmdOVkJBTVRESE41CmMzUmxiVHBoWkcxcGJqQlpNQk1HQnlxR1NNNDlBZ0VHQ0NxR1NNNDlBd0VIQTBJQUJQQWU1NFU1TVpXNGNTYXYKOE1iOVZXN1IzYkFqZWFMT2ErU0RLaU5VNzhHQ2tNSVRxanQ3MHJ0Umo2bTdVOEFDS09JeU13ZFFzOHQ0ejI0VQplSWkxYnJhalNEQkdNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBakFmCkJnTlZIU01FR0RBV2dCUmQvOGZibFJDNGg3U2hXRjBxcGs2MFRSU3I2ekFLQmdncWhrak9QUVFEQWdOSUFEQkYKQWlFQXZxUHh1VUdiam5nYnJlMFZkTnFpbFRsUDF2WEx1elBEb0c2M0RlQlJRc1FDSUdZcU5qeDd6MjNvNTVwago3c3p0aFZ3OXNCVDh4T3RjVGNLYjMyWnlKdG9oCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0KLS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJkekNDQVIyZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWpNU0V3SHdZRFZRUUREQmhyTTNNdFkyeHAKWlc1MExXTmhRREUyTlRFd05UZ3hNekF3SGhjTk1qSXdOREkzTVRFeE5UTXdXaGNOTXpJd05ESTBNVEV4TlRNdwpXakFqTVNFd0h3WURWUVFEREJock0zTXRZMnhwWlc1MExXTmhRREUyTlRFd05UZ3hNekF3V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFTVW5jWHladGRqQlNVcjFyWjQ5SkJrU09rTVpmQnVtVFVoUndONnVLTUoKS2F2YU42bUFaaGtVQkh2VjZJMGVVMktVVldjekc2c3o0VDEvT1I0M3p2ZnNvMEl3UURBT0JnTlZIUThCQWY4RQpCQU1DQXFRd0R3WURWUjBUQVFIL0JBVXdBd0VCL3pBZEJnTlZIUTRFRmdRVVhmL0gyNVVRdUllMG9WaGRLcVpPCnRFMFVxK3N3Q2dZSUtvWkl6ajBFQXdJRFNBQXdSUUlnUmw2NDBGYXVmRGdBdWJXc1BFLzNYMkxUZVdtM2paY2gKWEZMWUVHZ2xQM1VDSVFDTERUR1pIcE5UMmZJM1R1dS90cmQyWjNpVG9scFpnQ1pGeTE3RXRYN1NHZz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"\
   | base64 -d > cert

echo "LS0tLS1CRUdJTiBFQyBQUklWQVRFIEtFWS0tLS0tCk1IY0NBUUVFSUhyNnAwQ3pXU2VoOUVTRkNvTmVjcy9XbVJ3WHJhOWR2NWcrc2ZkOEN5dENvQW9HQ0NxR1NNNDkKQXdFSG9VUURRZ0FFOEI3bmhUa3hsYmh4SnEvd3h2MVZidEhkc0NONW9zNXI1SU1xSTFUdndZS1F3aE9xTzN2Uwp1MUdQcWJ0VHdBSW80akl6QjFDenkzalBiaFI0aUxWdXRnPT0KLS0tLS1FTkQgRUMgUFJJVkFURSBLRVktLS0tLQo" \
   | base64 -d > key
```

Or this way:

```bash
grep 'certificate-authority-data' $HOME/.kube/config | awk '{print $2}' | base64 -d > cacert
grep 'client-certificate-data' $HOME/.kube/config | awk '{print $2}' | base64 -d > cert
grep 'client-key-data' $HOME/.kube/config | awk '{print $2}' | base64 -d > key

CERT_INFO=`grep 'client-certificate-data' $HOME/.kube/config | awk '{print $2}' | base64 -d | openssl x509 -text`
echo $CERT_INFO
```

Then, get the right curl hidden behind a kubectl command with:

```bash
kubectl get pods -v=9
... curl -v -X GET -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" -H "User-Agent: kubectl/v1.22.4 (darwin/arm64) kubernetes/b695d79" 'https://0.0.0.0:62575/api/v1/namespaces/default/pods?limit=500' ...
```

Then, set the certificate and repeat the call:

```bash
curl -v --cacert cacert --cert cert --key key -X GET -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" -H "User-Agent: kubectl/v1.22.4 (darwin/arm64) kubernetes/b695d79" 'https://0.0.0.0:62575/api/v1/namespaces/default/pods?limit=500'
```

### Certificate files in `~/.kube/config`

If you have certificate files you can do directly:

```yaml
certificate-authority: /Users/alessandro.argentieri/.minikube/ca.crt
client-certificate: /Users/alessandro.argentieri/.minikube/profiles/minikube/client.crt
client-key: /Users/alessandro.argentieri/.minikube/profiles/minikube/client.key
```
Copy them in bash variables:

```bash
CA_CERT='/Users/alessandro.argentieri/.minikube/ca.crt'
CERT='/Users/alessandro.argentieri/.minikube/profiles/minikube/client.crt'
KEY='/Users/alessandro.argentieri/.minikube/profiles/minikube/client.key'
```

get the right curl done under the hood by kubectl:

```bash
kubectl get pods -v=9
```

copy it and reuse by adding the certs:

```bash
curl --cert $CERT --key $KEY --cacert $CA_CERT -X GET  -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" 'https://127.0.0.1:61988/api/v1/namespaces/default/pods?limit=500' | jq
```
