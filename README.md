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


