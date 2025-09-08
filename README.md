# playson-k8s
A k8s technical assignemt from Playson

* Cluster Setup

The cluster runs K3s with 3 nodes, one master and two workers:

```
kubectl get nodes
NAME               STATUS   ROLES                  AGE    VERSION
boolayan           Ready    control-plane,master   103d   v1.32.5+k3s1
boolayan-worker1   Ready    <none>                 103d   v1.32.5+k3s1
boolayan-worker2   Ready    <none>                 100d   v1.32.5+k3s1
```

Note: I faced an issue deploying httpbin image provided in the technical assignemt (kennethreitz/httpbin:latest
) that's because it is built for amd64 architecture only and it was failing in my arm based raspberry pi nodes.

```
kubectl get pods -n playson
httpbin-b8666bbb5-7cwfr    0/1     CrashLoopBackOff    4 (58s ago)   4m59s
httpbin-b8666bbb5-8x8fj    0/1     CrashLoopBackOff    4 (46s ago)   4m59s
httpbin-b8666bbb5-ltd7g    0/1     Error               2 (40s ago)   4m59s
```

Upon inspecting the image architecture I can confirm the lack of arm support:

```
sudo ctr content get sha256:b138b9264903f46a43e1c750e07dc06f5d2a1bd5d51f37fb185bc608f61090dd | jq .architecture
"amd64"
```

I did a quick search and I found that there is a similar image which supports multi architecture: mccutchen/go-httpbin:v2.8.0

I have used *kustomization.yaml* as it is straightforward to wrap up everything with *-k* as an argument for kubectl command.
Also, if we scale it would be easier as *Kustomize* acts as an overlay where *kubectl apply -f k8s/* will deploy all the resources
inside the folder after rendering them and apply *Kustomize* settings.

```
kubectl get pods -l app=httpbin
NAME                      READY   STATUS    RESTARTS   AGE
httpbin-68c76789c6-p57tg  1/1     Running   0          10m
httpbin-68c76789c6-wxk7s  1/1     Running   0          10m
httpbin-68c76789c6-4k3mf  1/1     Running   0          10m
```
Port forwarding the httpbin service localy so we can test with curl:

```
kubectl get svc -n playson
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
httpbin   ClusterIP   10.43.225.10   <none>        8080/TCP   21m
boolayan@boolayan:~/playson $ kubectl -n playson port-forward svc/httpbin 8084:8080 &
[1] 26536
boolayan@boolayan:~/playson $ curl http://localhost:8084/get
Handling connection for 8084
{
  "args": {},
  "headers": {
    "Accept": [
      "*/*"
    ],
    "Host": [
      "localhost:8084"
    ],
    "User-Agent": [
      "curl/7.88.1"
    ]
  },
  "origin": "127.0.0.1:60860",
  "url": "http://localhost:8084/get"
}
boolayan@boolayan:~/playson $ 
```

Also a describe for one of the httpbin pods:

```
kubectl -n playson  describe pod  httpbin-68c76789c6-p57tg 
Name:             httpbin-68c76789c6-p57tg
Namespace:        playson
Priority:         0
Service Account:  default
Node:             boolayan/192.168.0.196
Start Time:       Mon, 08 Sep 2025 23:23:38 +0100
Labels:           app=httpbin
                  pod-template-hash=68c76789c6
Annotations:      <none>
Status:           Running
IP:               10.42.0.96
IPs:
  IP:           10.42.0.96
Controlled By:  ReplicaSet/httpbin-68c76789c6
Containers:
  httpbin:
    Container ID:   containerd://bff917158fe9f169b2a3c38913cf0251277c8bd07a0ef99da29c02d054b357a9
    Image:          mccutchen/go-httpbin:v2.8.0
    Image ID:       docker.io/mccutchen/go-httpbin@sha256:9380c104186fb3bcdcc0b158c26d231fef73ead15905f4f99031beefb09b3d28
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 08 Sep 2025 23:24:24 +0100
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     200m
      memory:  256Mi
    Requests:
      cpu:        100m
      memory:     128Mi
	Liveness:  http-get http://:8080/status/200 delay=10s timeout=1s period=10s #success=1 #failure=3
    Readiness: http-get http://:8080/status/200 delay=5s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wdpfb (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-wdpfb:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m31s  default-scheduler  Successfully assigned playson/httpbin-68c76789c6-p57tg to boolayan
  Normal  Pulling    2m28s  kubelet            Pulling image "mccutchen/go-httpbin:v2.8.0"
  Normal  Pulled     105s   kubelet            Successfully pulled image "mccutchen/go-httpbin:v2.8.0" in 42.941s (42.941s including waiting). Image size: 13579723 bytes.
  Normal  Created    105s   kubelet            Created container: httpbin
  Normal  Started    105s   kubelet            Started container httpbin
  ```
* Potential enhancement if the service going to be in production:

- Default deny NetworkPolicy
- PSP standards and opa gatekeeper
- mTLS communication between workloads.
- Resource quota per namespace (i.e limitRange)
- HPA and VPA.
- More robust gitops worklflow (for example argocd or flux2)
- Pipeline with automated deployment based on promotion
