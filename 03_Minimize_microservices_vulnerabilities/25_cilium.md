# Pod to pod encryption

![](../images/25_pod2pod_enc_1.png)
![](../images/25_pod2pod_enc_2.png)

mTLS - service mesh like Istio  
Cilium - uses IPsec or wireguard  
Calico - uses IPSec  

## Cilium
### Installation
Installing Cilium and Enabling Encryption  
Enable encryption by installing Cilium with encryption enabled:

Add the Cilium Helm repository
```helm repo add cilium https://helm.cilium.io/```

Install Cilium with encryption enabled
```
helm install cilium cilium/cilium --version 1.16.3 \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```
You can verify that Cilium is running and that encryption is enabled by running the following commands:

Wait for the status of cilium to be OK  
```cilium status```
Check the encryption status of the Cilium installation  
```cilium encryption status```
![](../images/25_pod2pod_enc_3.png)
![](../images/25_pod2pod_enc_4.png)
![](../images/25_pod2pod_enc_5.png)
![](../images/25_pod2pod_enc_6.png)
![](../images/25_pod2pod_enc_7.png)
![](../images/25_pod2pod_enc_8.png)
![](../images/25_pod2pod_enc_9.png)

### Other way to verify encryption
Check connectivity between the pods, in a new terminal window run the following command:

```watch kubectl exec -it curlpod -- curl -s http://nginx```

The watch curl command should return the HTML content of the NGINX welcome page, indicating that the client pod can access the NGINX pod.

Run a bash shell in one of the Cilium pods with ```kubectl -n kube-system exec -ti ds/cilium -- bash``` and execute the following commands:

Check that WireGuard has been enabled (number of peers should correspond to a number of nodes subtracted by one): ```cilium-dbg status | grep Encryption```

Install tcpdump:  
```
apt-get update
apt-get -y install tcpdump
```
Check that traffic is sent via the cilium_wg0 tunnel device is encrypted:  

```tcpdump -n -i cilium_wg0 -X```

Here we are using `tcpdump`` to capture and display detailed network packets on the cilium_wg0 interface.

The -n option avoids DNS lookups, and the -X option shows packet content in both hexadecimal and ASCII format.

### Cilium Network Policies
#### Deny policies
There is an existing CiliumNetworkPolicy
default-allow in namespace team-azure which allows all traffic.
  
Create a CiliumNetworkPolicy in Namespace team-azure as follows:
  
Create a Layer 3 policy named p1 that denies outgoing traffic from Pods with the label role=messenger to Pods with the label role=database.
```
root@controlplane ~ ➜  k get cnp -n team-azure -o yaml
apiVersion: v1
items:
- apiVersion: cilium.io/v2
  kind: CiliumNetworkPolicy
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"cilium.io/v2","kind":"CiliumNetworkPolicy","metadata":{"annotations":{},"name":"default-allow","namespace":"team-azure"},"spec":{"egress":[{"toEndpoints":[{"matchLabels":{}}]}],"endpointSelector":{},"ingress":[{"fromEndpoints":[{"matchLabels":{}}]}]}}
    creationTimestamp: "2025-01-07T12:53:35Z"
    generation: 1
    name: default-allow
    namespace: team-azure
    resourceVersion: "2855"
    uid: 9aaf99b8-2525-4303-b1a1-ac7100bed3b3
  spec:
    egress:
    - toEndpoints:
      - matchLabels: {}
    endpointSelector: {}
    ingress:
    - fromEndpoints:
      - matchLabels: {}
kind: List
metadata:
  resourceVersion: ""

root@controlplane ~ ➜  
```
Deny policy:  
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: p1
  namespace: team-azure
spec:
  endpointSelector:
    matchLabels:
      role: messenger
  egressDeny:
  - toEndpoints:
    - matchLabels:
        role: database
```

Via tcpdump, you should see the traffic between the pods.

We see requests from curlpod to nginx and responses from nginx to curlpod in tcpdump output.

