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
kubectl create deployment test --replicas 4 --image alpine -- sh -c 'echo "hello!"; tail -f /dev/null'
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

## Get all the pods in a specific node
You can get all the pods deployed in a node, giving the node name:

```bash
kubectl get pods --all-namespaces --field-selector spec.nodeName=<NODE_NAME>
```

## Get the pod IP (visible only inside the cluster)

Even if you normally reach pods through their service objects, you could connect (or let another pod) connect to a pod using its own IP address.
To know the IP address of a pod you could:

```bash
$ kubectl get pod centos-pod -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
centos-pod   1/1     Running   0          45h   172.17.0.3   minikube   <none>           <none>
```

or using the template:

```bash
$ kubectl get pod centos-pod --template '{{.status.podIP}}'
172.17.0.3
```

## Node subnet

To guarantee each pod has its own unique private IP address inside the entire cluster, to any kubernetes node it's given a unique subnet from which pods are assigned IP addresses on that node.
To know the range of private IPs the subnet for a node is given you can use the following command:

```bash
$ kubectl get node <node-name> -o json | jq '.spec.podCIDR'
"10.244.0.0/24"
```

## Pod to pod communication (kubernetes networking)

To summarize the Kubernetes networking in a few works:
- Every node has its own IP address (public or private if the cluster is in a LAN), for example 172.23.1.1.
- Every pod has its own private IP adress, unique in the cluster taken from a subnet specific for the node, for example: 10.4.4.4
- Every node has a subnet of private IPs from which its pods get their private IPs, for example: 10.4.4.0/24.
- Flannel or Calico are CNI plugins that work in every node of the cluster and create container networks, namespaces and interfaces.

If a pod on a node must communicate with a pod of another node (knowing its unique IP, for example 10.4.4.4), the linux kernel of the node checks the route table of the node to check what node IP address corresponds to the specific private subnet to which that private pod IP address belongs to, for example:

```
10.4.4.0/24 routed to 172.9.9.9 
```

the CNI plugin wraps the request into an UDP package (vxlan for Flannel, ip-in-ip for Calico).
So instead of sending:

```
IP: 10.4.4.4
TCP stuff
HTTP stuff
```

we will send:

```
IP: 172.9.9.9
(extra wrapper stuff)
IP: 10.4.4.4
TCP stuff
HTTP stuff
```

The request is sent to that node (172.9.9.9) and Flannel or Calico unwrap the request, get the private pod IP (10.4.4.4) belonging to the private subnet for that node (10.4.4.0/24) and forwards the request to that namespace, network and IP.

The nodes are connected through HTTP, except if you manually configure all the certificates to expone them over HTTPS. A common best practice is to add every node to a VPN, so they can see each other like they were in the same LAN.

Useful links:

https://jvns.ca/blog/2016/12/22/container-networking

https://ronaknathani.com/blog/2020/08/how-a-kubernetes-pod-gets-an-ip-address/

https://stackunderflow.dev/p/network-namespaces-and-docker/

## Logging

Here the ways to inspect pod logs using `kubectl`:

```bash
# logs recorded since 15 minutes ago
kubectl logs --since=15m <podname>
# last 100 lines of logs
kubectl logs --tail=100 -f <podname>
# logs recorded before a crash-restart
kubectl logs --previous <podname>
```

Cumulate logs from all the pods of the same deployment:

```bash
kubectl -n <namespace> get pods | grep <deployment-name> | cut -d ' ' -f 1 | xargs -I {} kubectl -n <namespace> logs {} --since=5h | grep 'something to search in the last 5 hours in all the pods' | less
```

## Apply a YAML on the fly without saving a file

You can paste your YAML manifests in the terminal without having to save them locally by using this syntax:

```bash
$ kubectl apply -f - << EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: log-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
EOF  
```

## Copy files from local machine to pod and vice versa

Yoi can use kubectl to copy files from and to a pod:

```bash
# copy file from local machine to pod
kubectl cp $PWD/example_file.txt centos-pod:/home/example_file.txt

# copy file from pod to local machine (renaming the file)
kubectl cp centos-pod:/home/example_file.txt $PWD/hello.txt
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

You can avoid using `CA_CERT` but you must activate the flag --insecure (or -k) in the curl operation to allow the untrusted HTTPS self signed certificate of the control plane.

### I don't want to use the flag -v=9!

Let's do it again but we don't wanna use the facilitation of the -v=9 flag!

Get the API Server URLs:

```bash
$ kubectl config view | grep server
server: https://501F17E9B.we9.eu-west-1.eks.amazonaws.com
server: https://127.0.0.1:64775
```

Get the certificates:

```bash
$ export clientcert=$(grep client-cert ~/.kube/config | cut -d " " -f 6)
$ echo $clientcert
/Users/alessandro.argentieri/.minikube/profiles/minikube/client.crt

$ export clientkey=$(grep client-key ~/.kube/config | cut -d " " -f 6)
$ echo $clientkey
/Users/alessandro.argentieri/.minikube/profiles/minikube/client.key

$ export certauth=$(grep certificate-authority ~/.kube/config | cut -d " " -f 6)
$ echo $certauth
/Users/alessandro.argentieri/.minikube/profiles/minikube/ca.pem
```

perform the curl to the URL found before:

```bash
$ curl --cert $clientcert --key $clientkey --cacert $certauth https://127.0.0.1:64775/api/v1/pods | head -12
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "45284"
  },
  "items": [
    {
      "metadata": {
        "name": "geocall-79896c8757-4wwkm",
        "generateName": "geocall-79896c8757-",
        "namespace": "default",
```

as before, if the certificates in the ~/.kube/config are shown as strings transform those in files:

```bash
$ export clientcert=$(grep client-cert-data ~/.kube/config | cut -d " " -f 6)
$ echo $clientcert
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...

$ export clientkey=$(grep client-key-data ~/.kube/config | cut -d " " -f 6)
$ echo $clientkey
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktL...

$ export certauth=$(grep certificate-authority-data ~/.kube/config | cut -d " " -f 6)
$ echo $certauth
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk...

$ echo $clientcert | base64 -d > ./client.pem
$ echo $clientkey | base64 -d > ./client-key.pem
$ echo $certauth | base64 -d > ./ca.pem
```

and perform the curl by passing the files just created:

```bash
$ curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://127.0.0.1:64775/api/v1/pods | head -12
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "45284"
  },
  "items": [
    {
      "metadata": {
        "name": "geocall-79896c8757-4wwkm",
        "generateName": "geocall-79896c8757-",
        "namespace": "default",
```

### cURL kubernetes through a proxy (without certificates)

You can enable a proxy to curl your kube api server:

```bash
$ kubectl proxy --port=8081
Starting to serve on 127.0.0.1:8081
```

when the proxy is enabled you can curl with no authentication:

```bash
$ curl http://localhost:8081/api/v1/namespaces/default/pods | head -11
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "44567"
  },
  "items": [
    {
      "metadata": {
        "name": "geocall-79896c8757-4wwkm",
        "generateName": "geocall-79896c8757-",
        "namespace": "default",
```

## cURL API server from inside a pod

To let a pod communicate with the kube api server we need to extend its grants.
If you have RBAC enabled on your cluster, use the following snippet to create role binding which will grant the default service account view permissions:

```bash
$ kubectl create clusterrolebinding default-view --clusterrole=view --serviceaccount=default:default
clusterrolebinding.rbac.authorization.k8s.io/default-view created
```

run a pod inside your cluster and go inside it with the bash command:

```bash
$ kubectl run centos-pod --image=centos:7 -- bash -c "tail -f /dev/null" --restart=Never
$ kubectl exec -it centos-pod -- bash
```
the pods have a specific folder in which a bearer token and a cacert is contained for their specific serviceaccount.
From inside the pod let's get the token and call the kube API server:

```bash
$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
       -H "Authorization: Bearer $TOKEN" \
       https://kubernetes.default/api/v1/pods | head -11
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "951"
  },
  "items": [
    {
      "metadata": {
        "name": "centos-pod",
        "namespace": "default",
```
The token is necessary because the kube API server has an authentication mechanism, the cacert is also necessary because it communicates through HTTPS with its nodes (not with the rest of the internet) with a self-signed certificate. So there is no certificate authority to pass the cacert and we need to use the one provided by the kube API server when the pod has been initialised.

We can avoid using the cacert but we need to use `-k` flag or `--insecure` flag to our `curl` command:

```bash
# alternatives to the usage of the --cacert flag
$ curl -k -H "Authorization: Bearer $TOKEN"  https://kubernetes.default/api/v1/pods
$ curl -H "Authorization: Bearer $TOKEN"  https://kubernetes.default/api/v1/pods --insecure
```

In particular, it's equivalent curling the host `https://kubernetes.default` or the host `https://kubernetes.default/svc`, so it would be equivalent doing:

```bash
$ curl -k -H "Authorization: Bearer $TOKEN"  https://kubernetes.default/api/v1/pods
$ curl -k -H "Authorization: Bearer $TOKEN"  https://kubernetes.default.svc/api/v1/pods
```

but instead of using a DNS name we can use directly the kube API server IP visible ONLY from inside the pod:

```bash
$ echo $KUBERNETES_SERVICE_HOST
10.96.0.1

$ curl -k -H "Authorization: Bearer $TOKEN"  https://$KUBERNETES_SERVICE_HOST/api/v1/pods
```

# API-Resources and CRDs

You can get the whole list of kubernetes API resources (standard and custom by typing:

```bash
$ kubectl api-resources
```

If you want to get only the custom ones type:

```bash
$ kubectl get crds
```

# Saving a kubeconfig from a secret

Sometimes you can have sensitive data in a kubernetes secret that you want to save. For example you can have a kubeconfig file to access to another cluster (a subcluster for example). Let's suppose you have the field `.data.kubeconfig` of the json version of the secret containing a new kubeconfig to be saved locally. Then you can acquire the information (opaque in base64), decode it and save in a file. Then you can switch the context your kubectl is pointing to by exporting the $KUBECONFIG environment variable and pointing to the new kubeconfig file:

```bash
$ kubectl get secret mysecret -o jsonpath="{.data.kubeconfig}" | base64 --decode > ~/.kube/config_new-context
$ export KUBECONFIG=~/.kube/config_new-context
# now your kubectl points to the new context / cluster
```

# ClusterRole, ClusterRoleBinding, ServiceAccount

```bash
$ kubectl create clusterrolebinding NAME --clusterrole=NAME [--user=username] [--group=groupname] [--serviceaccount=namespace:serviceaccountname] [--dry-run=server|client|none]

$ kubectl create namespace test-namespace
$ kubectl create serviceaccount deleter-svc-account

$ kubectl create clusterrole pod-deleter-c-role --verb=get,list,watch,delete --resource=pods
$ kubectl create clusterrolebinding deleter --clusterrole=pod-deleter-c-role --serviceaccount=test-namespace:deleter-svc-account
$ kubectl run --rm -i chaos-deleter --image=alessandroargentieri/chaos --serviceaccount=deleter-svc-account

# pod -> svc-account
# cluster-role-binding -> cluster-role
#                      -> svc-account
```

# Some commands through temporary pods

Suppose we launched these commands:

```
CREATE DATABASE staff;
USE staff;
CREATE TABLE editorials (id INT, name VARCHAR(20), email VARCHAR(20), PRIMARY KEY(id));
SHOW TABLES;
INSERT INTO editorials (id, name, email) VALUES (01, "Olivia", "olivia@company.com");
SELECT * FROM editorial;
```

Launch a pod with a MySQL client and use it:

```
$ kubectl run -it mysql-client --image=logiqx/mysql-client --restart=Never --rm -- sh
~/work $ mysql --host=10.88.88.34 --port=3306 --database=staff --user=root --password=abcd1234 

MySQL [staff]> show tables;
+-----------------+
| Tables_in_staff |
+-----------------+
| editorials      |
+-----------------+
1 row in set (0.004 sec)

MySQL [staff]> select * from editorials;
+----+--------+--------------------+
| id | name   | email              |
+----+--------+--------------------+
|  1 | Olivia | olivia@company.com |
+----+--------+--------------------+
1 row in set (0.002 sec)
```
# Dump a Redis Database in a Kubernetes cluster and copy locally

```bash
# fetch the container env var REDIS_PASS:
# export REDIS_PASS=`k exec -it redis-0 -- sh -c 'env | grep REDIS_PASS | cut -d "=" -f 2'`
kubectl exec -it redis-0 -- sh
/ env | grep REDIS_PASS
REDIS_PASS=e0ub6jEsxHmYAdLe149Zaqh6ohvLTP
/ redis-cli 
> AUTH REDIS_PASS=e0ub6jEsxHmYAdLe149Zaqh6ohvLTP
> SELECT 13
> DBSIZE
> SAVE
> exit
kubectl cp redis-0:/data/dump.rdb ./redis-dump.rdb
```

# Expose cluster with the official dashboard:

Create the dashboard in your cluster:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
Expose it through a nodeport service:
```
kubectl expose deployment kubernetes-dashboard --type=NodePort --name=kubernetes-dashboard-nodeport --namespace=kubernetes-dashboard --port=80 --target-port=8443
```
Fetch the nodeport port, in this case is 32254:
```
kubectl get svc -n kubernetes-dashboard
NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard            ClusterIP   10.43.24.84    <none>        443/TCP        8m4s
dashboard-metrics-scraper       ClusterIP   10.43.32.18    <none>        8000/TCP       7m59s
kubernetes-dashboard-nodeport   NodePort    10.43.167.15   <none>        80:32254/TCP   21s
```
Fetch the public URL of one node (the controlplane is good in this case 212.2.241.66):
```
$ kubectl cluster-info
Kubernetes control plane is running at https://212.2.241.66:6443
CoreDNS is running at https://212.2.241.66:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://212.2.241.66:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```
Fetch the security token from the deployed secret:
```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep kubernetes-dashboard | awk '{print $1}')
```
Visit your browser: https://212.2.241.66:32254/
And paste the token when requested.

The dashboard has metrics and logs for the whole cluster and uses the metrics server to fetch all the metrics. The same you could achieve with:
```
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods?labelSelector=<label_selector>
kubectl logs <podname> -n namespace
```
# HTTPS with ingress

In this example:
- we have a domain in `Tophost.it` and we move it to `civo.com`
- we add two DNS records for `@ and www` in `civo.com`
- we have a kubernetes cluster in `civo.com`
- we issue a production certificate for `alessandroargentieri.com` and `www.alessandroargentieri.com`

Create a CIVO cluster:
```bash
$ civo kube create civo-tls
```
When the cluster is ready download its config and export the KUBECONFIG variable:
```bash
$ civo kube config civo-tls > ~/.kube/config_civo-tls
$ export KUBECONFIG=$HOME/.kube/config_civo-tls
```
Check if the cluster is reachable:
```bash
$ kubectl cluster-info
```

Install cert manager through helm:
```bash    
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
```
Install NGINX ingress controller through helm:
```bash    
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm update
$ helm install ingress-controller ingress-nginx/ingress-nginx
```
Install the `ClusterIssuer` for production by specifying your own domain and associated email:
```bash    
$ kubectl apply -f - << EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: hello@alessandroargentieri.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```
Install the Hello Deployment, the service and the Ingress (rules) to access via TLS (HTTPS):
```bash    
$ kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
  tls:
    - hosts:
      - alessandroargentieri.com
EOF
```
Check the External IP to reach the cluster:
```bash
$ kubectl get ingress
NAME    CLASS    HOSTS   ADDRESS         PORTS     AGE
hello   <none>   *       74.220.21.171   80, 443   3m48s
```
That means, via an external `LoadBalancer` which is created for the ingress (but the Loadbalancer is L4, and the ingress-controller reached through it is L7 so completes the logics that are missing in the Loadbalancer):
```bash
$ kubectl get svc
NAME                                                    TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
kubernetes                                              ClusterIP      10.43.0.1      <none>          443/TCP                      26m
ingress-controller-ingress-nginx-controller-admission   ClusterIP      10.43.49.111   <none>          443/TCP                      16m
ingress-controller-ingress-nginx-controller             LoadBalancer   10.43.181.71   74.220.21.171   80:30376/TCP,443:31485/TCP   16m
hello                                                   ClusterIP      10.43.111.5    <none>          80/TCP                       13m
```
Then from the Civo Dashboard navigate to `Networking>DNS>Add`
From there add `alessandroargentieri.com`.
You will be asked to set the Civo DNS servers in the originary website where you've bought that domain.
In my case it was Namecheap. By logging in Namecheap and going to the section `DNS>Custom Domains` I can indicate the new Civo Nameservers which will substitute the Namecheap ones. So I'll add `ns0.civo.com` and `ns1.civo.com` as the new nameservers for that domain.
The change, after saving, will be visible by doing (in the terminal):
```bash
$ dig NS alessandroargentieri.com
$ dig alessandroargentieri.com
```
I could have left Namecheap as DNS Nameserver and pointed to the Civo cluster IP address, but I did so to move also the Domain "Ownership" to Civo.
After that you can go back to civo.com and continue the DNS configuration by adding the DNS records.
So, click on the right side of the added DNS on `Actions>DNS Records`
Add two DNS records:
```
Type: A, Name: @, Value: 74.220.21.171
Type: A, Name: www, Value: 74.220.21.171
```

Now let's issue a Certificate.
We need the DNS entry for the Loadbalancer (L4) created for the Ingress (L7).
We can fetch via `civo` CLI with:
```bash
$ civo loadbalancer ls
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------+
| ID                                   | Name                                                         | Algorithm   | Public IP     | State     | Backends                 |
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------+
| fb88ba8c-7c4a-4ce4-a524-c0b82aee1698 | civo-tls-default-ingress-controller-ingress-nginx-controller | round_robin | 74.220.21.171 | available | 192.168.1.3, 192.168.1.3 |
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------+

$ civo loadbalancer show civo-tls-default-ingress-controller-ingress-nginx-controller
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------------------------------+--------------------------+
| ID                                   | Name                                                         | Algorithm   | Public IP     | State     | DNS Entry                                        | Backends                 |
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------------------------------+--------------------------+
| fb88ba8c-7c4a-4ce4-a524-c0b82aee1698 | civo-tls-default-ingress-controller-ingress-nginx-controller | round_robin | 74.220.21.171 | available | fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com | 192.168.1.3, 192.168.1.3 |
+--------------------------------------+--------------------------------------------------------------+-------------+---------------+-----------+--------------------------------------------------+--------------------------+
```
Or via `kubectl` over the `ingress` object with:
```bash
$ kubectl get ingress hello -o jsonpath={".status.loadBalancer.ingress[0].hostname"}
fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com
```
Or, alternatively, via `kubectl` over the `loadbalancer` service with:
```bash
$ kubectl get service/ingress-controller-ingress-nginx-controller-o jsonpath={".status.loadBalancer.ingress[0].hostname"}
fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com
```
We can compose and apply the `Certificate` manifest to be issued (reference: `https://cert-manager.io/docs/reference/api-docs/#cert-manager.io`)

```bash
$ kubectl apply -f - << EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-cert
  namespace: default
spec:
  secretName: tls-cert
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
  - fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com
  - alessandroargentieri.com 
EOF
```
Now that we,ve created the certificate, let's wait it to be ready:
```bash
$ kubectl get certs tls-cert
```
When it's ready if we search `https://alessandroargentieri.com` on the browser, we see it's still unsecure.
We need to update the ingress by setting the `tls-cert`:

```bash
$ kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  rules:
    - host: alessandroargentieri.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
    - host: fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80                  
  tls:
    - hosts:
      - alessandroargentieri.com
      - fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com
      secretName: tls-cert 
EOF
```
Here we decided to bring TLS to both the `alessandroargentieri.com` domain and the `fb88ba8c-7c4a-4ce4-a524-c0b82aee1698.lb.civo.com` (ingress) loadbalancer domain.
This is possible, because we mentioned both the URLs in the `tls-cert` and in the `ingress`.
Alternatively, you could choose only to encrypt the `alessandroargentieri.com` connection.
And now, by searching on the browser `https://alessandroargentieri.com` we find a valid certificate, verifiable through:
```bash
$ echo | openssl s_client -connect alessandroargentieri.com:443 -servername alessandroargentieri.com
```

# HTTPS with ingress (2)

In this example:
- we have a sub domain in `Tophost.it`
- we have a kubernetes cluster in `civo.com`
- we issue a production certificate for `hello.quicktutorialz.com` and `www.hello.quicktutorialz.com`

Create a CIVO cluster with Civo CLI:
```bash
$ civo kube create hello-tls
```

When the cluster is ready download its config and export the KUBECONFIG variable:
```bash
$ civo kube config hello-tls > ~/.kube/config_hello-tls
$ export KUBECONFIG=$HOME/.kube/config_hello-tls
```
Check if the cluster is reachable:
```bash
$ kubectl cluster-info
```

Install cert manager through helm:
```bash    
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
```

Install NGINX ingress controller through helm:
```bash    
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm update
$ helm install ingress-controller ingress-nginx/ingress-nginx
```

Install the Hello Deployment exposing port 8080 and its Service exposing port 80:
```bash    
$ kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: jmalloc/echo-server:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
```
You can verify the `hello` service is correctly connected to the `hello-deployment` via port-forwarding to the `hello` service:
```bash
    $ k get pods
NAME                    READY   STATUS    RESTARTS   AGE
hello-d5686fbfd-7mssx   1/1     Running   0          14s
hello-d5686fbfd-vxvhl   1/1     Running   0          14s
~  $ k port-forward services/hello 8003:80
Forwarding from 127.0.0.1:8003 -> 8080
Forwarding from [::1]:8003 -> 8080
Handling connection for 8003
```
You can verify from your browser by visiting `http://localhost:8003` or via `curl http://localhost:8003`.
After that, you can press `CTRL+C` to stop the port-forward process.

We can expose the Cluster to the outside world (no need anymore of `port-forward`) using:
- a NodePort which is a Service exposing a port (of type `NodePort`) from every node of the cluster. With the NodePort we're not able to insert the TLS logic easily because this service is a Network Layer service (L4) so it doesn't discriminate the TLS information from the packets it receives. The TLS information is in the HTTP protocol which is implemented in the Application Layer (L7), instead the Network Layer (L4) of the NodePort service allows this service to only send packets to the right destinations using TCP (or UDP). The HTTP protocol is built on top of TCP, but the NodePort service is too dumb to understand its logics. Besides that, the NodePort service in Kubernetes open a port on every node of the cluster and this can bring to security issues.
- a LoadBalancer, (often named "external" LoadBalancer) which is a Service type in Kubernetes that secretely creates private NodePorts (invisible to the administrators) and, through the "Cloud Controller Manager" pod installed in the Managed Cluster, triggers the creation of an external service acting as a Reverse Proxy and sending requests (in Round Robin) to the pointed services/deployments in the cluster via the secreted NodePorts. If the cluster is not managed, no Cloud Controller Manager is installed and by installing a `LoadBalancer` Service it only generate a NodePort and/or some proxy on it (like for Minikube). Normally, when you build the cluster from the ground-up, you need to install some sort of provider to allow the creation of an external LoadBalancer, like MetalLB which is a famous tool used for self-managed clusters. With `LoadBalancer` Service type, we have L4 level services, so even in this case is not possible to manage directly TLS.
- an Ingress which is a L7 (Application Layer) service type. By creating an Ingress we're triggering the creation of an External LoadBalancer (L4) which connects to invisible NodePorts in every kubernetes node, backed by a deployment which contains all the L7 logic (TLS, prefixes, headers, cookies, etc.). This way is much more easy and preferred to integrate TLS.

Let's install a basic NGINX ingress controller:
```bash
$ kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
EOF
```
Once the ingress is deployed, you can check the exported public IP through:
```bash
$ kubectl get ingress
NAME    CLASS    HOSTS  ADDRESS         PORTS     AGE
hello   <none>   *      74.220.18.132   80        2m
```
With the public IP Address you can go to your Domain provider and add two DNS records as follows:
If the domain is registered in `Civo`, add the domain: `hello.quicktutorialz.com` and from this, add the two records:
```
Type: A, Name: @, Value: 74.220.18.132
Type: A, Name: www, Value: 74.220.18.132
```
In my case, it is registered in `Tophost.it` so I was required to add the two DNS records for the registered `quicktutorialz.com` domain:
```
Type: A, Name: hello.quicktutorialz.com, Value: 74.220.18.132
Type: A, Name: www.hello.quicktutorialz.com, Value: 74.220.18.132
```
Once we have the two DNS registered and the information spread all over the internet (you can check with `dig NS hello.quicktutorialz.com`) we can start thinking about the TLS by adding a `CertificateIssuer` and issuing a `Certificate` (both Custom Resources have been already installed in the cluster through Helm).

Install the `ClusterIssuer` for production by specifying your own domain and associated email:
```bash    
$ kubectl apply -f - << EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: alexmawashi87@gmail.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

Once we have the `ClusterIssuer` we can proceed creating a `tls-cert` of type `Certificate` Custom Resource, already installed in our cluster.
We want to have the certificate to secure the domain `hello.quicktutorialz.com` which has been already registered and to which we will connect our cluster via the Ingress `hello`:
```bash
$ kubectl apply -f - << EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-cert
  namespace: default
spec:
  secretName: tls-cert
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
  - hello.quicktutorialz.com     # it must exist somewhere as added DNS record/domain
  - www.hello.quicktutorialz.com # it must exist somewhere as added DNS record/domain
EOF
```
Now that we,ve created the certificate, let's wait it to be ready:
```bash
$ kubectl get certs tls-cert
NAME       READY   SECRET     AGE
tls-cert   True    tls-cert   34s
```
Now that the `tls-cert` is ready for the existing `hello` ingress and the existing and registered `hello.quicktutorialz.com` we can update the ingress definition by adding the TLS:
```bash
$ kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  rules:
    - host: hello.quicktutorialz.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
    - host: www.hello.quicktutorialz.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80                  
  tls:
    - hosts:
      - hello.quicktutorialz.com
      - www.hello.quicktutorialz.com
      secretName: tls-cert 
EOF
```
Now, if we query the ingress, the 443 port for TLS and the given domains will appear:
```bash
$ kubectl get ingress
NAME    CLASS    HOSTS                                                   ADDRESS         PORTS     AGE
hello   <none>   hello.quicktutorialz.com,www.hello.quicktutorialz.com   74.220.18.132   80, 443   2m
```
You can check the secured connection via:
```bash
curl https://hello.quicktutorialz.com
curl https://www.hello.quicktutorialz.com
```
and, alternatively, you can check the validity of your certificates through:
```bash
$ echo | openssl s_client -connect hello.quicktutorialz.com:443 -servername hello.quicktutorialz.com
```

And. what if we have `https://mydomain.com` for the website and `https://api.mydomain.com` for the APIs?
Isn't it exactly the same?
😉

---

## Add Profiling to your code with `pprof` and `parca` (as client)

###  1. install the pprof in the code (https://www.parca.dev/docs/instrumenting-go)
```golang
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Your application code...
}
```
### 2. run the parca server inside your code and visit:
http://localhost:6060/debug/pprof/

### 3. if that code is deployed in three replicas in kubernetes pods, port-forward to 6061, 6062, 6063
Run the port forward from the three replicas on which the pprof is running on port 6060 (inside the pods) exposing them on ports 6061, 6062, 6063 (from three different terminals):
```bash
$ kubectl port-forward pods/mypod-7998bd4dcd-k5tn7 -n mynamespace 6061:6060
$ kubectl port-forward pods/mypod-7998bd4dcd-rnbj2 -n mynamespace 6062:6060
$ kubectl port-forward pods/mypod-7998bd4dcd-tzxmg -n mynamespace 6063:6060
```
You can visit the browser http://localhost:6061/debug/pprof/, http://localhost:6062/debug/pprof/, http://localhost:6063/debug/pprof/

### 4. install parca locally, a better client for the pprof (https://www.parca.dev/docs/binary)
Installation of parca locally:
```bash
curl -sL https://github.com/parca-dev/parca/releases/download/v0.18.0/parca_0.18.0_`uname -s`_`uname -m`.tar.gz | tar xvfz - parca
```
Download the base configs for parca, customise and use them:
```bash
curl -sL https://raw.githubusercontent.com/parca-dev/parca/v0.18.0/parca.yaml > parca.yaml
nano parca.yaml # substitute the port with 6061, 6062, 6063 (you can have an array of ports from different pod replicas, for example)
./parca --config-path="parca.yaml"
```
From the process logs you can see the local parca server is starting on port 7070 so you can visit your browser at http://localhost:7070 where the parca server (client for the pprof) is running (listening on the three replicas)

--

## how k8s manages its internal DNS names (like service.namespace.cluster.local)

Kubernetes usa dei nomi logici al posto degli IP e il DNS Server (nameserver) che viene interpellato dai pod per la risoluzione di tali nomi è il pod di CoreDNS attivo nel namespace `kube-system`.
I pod, come tutte le macchine linux, hanno l'informazione del nameserver nel file locale `/etc/resolv.conf` inizializzato quando il pod viene avviato.
```bash
# avvio un pod nel cluster
$ kubectl run centos-pod --image=centos:7 -- bash -c "tail -f /dev/null" --restart=Never
pod/centos-pod created

# vediamo il suo IP
$ k get pods/centos-pod -o wide
NAME         READY   STATUS    RESTARTS   AGE     IP           NODE                                             NOMINATED NODE   READINESS GATES
centos-pod   1/1     Running   0          3m19s   10.42.0.20   k3s-hello-tls-e92c-b6f3b2-node-pool-645d-8bfg9   <none>           <none>

# vediamo come viene risolto il DNS a livello del pod (il file etc/resolv.conf contiene il nameserver del cluster 10.43.0.10)
 $ k exec -it centos-pod -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.43.0.10
options ndots:5

# vediamo il file degli host per il pod
$ k exec -it centos-pod -- cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.42.0.20	centos-pod

# guardacaso il nameserver 10.43.0.10 è proprio il SVC che punta a coreDNS avviato come deployment nel ns kube-system
$ k get svc/kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.43.0.10   <none>        53/UDP,53/TCP,9153/TCP   82d

# coreDNS è il server DNS (nameserver) interno al cluster per risolvere tutti i DNS interni al cluster
$ k get deploy/coredns -n kube-system -o wide
NAME      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                   SELECTOR
coredns   1/1     1            1           82d   coredns      rancher/mirrored-coredns-coredns:1.9.1   k8s-app=kube-dns
```
