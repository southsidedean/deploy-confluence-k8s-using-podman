## Deploying a Confluence Server in Kubernetes Using a Podman-Generated YAML Manifest

### Tom Dean: [LinkedIn](https://www.linkedin.com/in/tomdeanjr/) / [GitHub](https://github.com/southsidedean)

## Introduction

In a previous tutorial, [Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://github.com/southsidedean/deploy-kind-using-podman-ubuntu), we took a look at how to use Podman to deploy a KIND container.

While discussing good use cases for an article on deploying an application in a Podman Pod with my friend Jeff Kaleth, he suggested Atlassian Confluence, as he does quite a bit of work documenting things with it.  Confluence tends to be consumed as SaaS these days, but sometimes you might want to run a local instance.

Traditionally, this would mean installing software either on a bare metal server or virtual machine, on top of an operating system.  This would be messy, and take time and effort to get everything configured properly.  Yuck!

*There has to be a better way.  There is!*

In a previous tutorial, [Using Podman to Generate and Test a Kubernetes YAML Manifest](https://github.com/southsidedean/using-podman-generate-test-k8s-manifest), we took a look at how we can use Podman to deploy a Confluence server, with a Postgres database, in a pod, and then create a YAML manifest, using the `podman generate kube` command.  We then used Podman to test our YAML manifest.

In this tutorial, I'm going to show you how to deploy the YAML manifest for our Confluence server pod in Kubernetes.  We're going to deploy (*and debug*) our manifest to get it working on a KIND, or Kubernetes-In-Docker, cluster.

## References

[GitHub: Deploying a Confluence Server in Kubernetes Using a Podman-Generated YAML Manifest](https://github.com/southsidedean/deploy-confluence-k8s-using-podman)

[GitHub: Using Podman to Generate and Test a Kubernetes YAML Manifest](https://github.com/southsidedean/using-podman-generate-test-k8s-manifest)

[GitHub: Deploying a Confluence Server in a Podman Pod Using Containers](https://github.com/southsidedean/deploy-confluence-podman-pod)

[GitHub: Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://github.com/southsidedean/deploy-kind-using-podman-ubuntu)

[KIND: Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

[Getting Started with Podman ](https://podman.io/getting-started/)

[Kubernetes Documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)

## Prerequisites

You will need an x64 Linux instance (physical or virtual) to deploy the Confluence container we will be using.  You're also going to need Podman installed and configured on your Linux instance.  If you need to accomplish this, see the [Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://github.com/southsidedean/deploy-kind-using-podman-ubuntu) article.

You're also going to need the YAML manifest we created in the [Using Podman to Generate and Test a Kubernetes YAML Manifest](https://github.com/southsidedean/using-podman-generate-test-k8s-manifest) tutorial.  If you haven't generated the YAML manifest, head over to that article and do it!

Finally, you're going to need a Kubernetes-In-Docker, or KIND cluster.  If you have another Kubernetes cluster to deploy to, you can use that.  If you need to prepare your host to deploy a KIND cluster, this is also covered in the [Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://github.com/southsidedean/deploy-kind-using-podman-ubuntu) tutorial.

## Deploying a Kubernetes-In-Docker (KIND) Cluster

The first thing we're going to need is a Kubernetes cluster to deploy to.  A quick and easy option is a Kubernetes-In-Docker, or KIND, cluster.  A KIND cluster is a Kubernetes cluster that runs as a container.  I deploy KIND using Podman, rootlessly, but you can use Docker if you prefer.

Let's check for existing KIND clusters:
```bash
kind get clusters
```

We shouldn't see any:
```bash
No kind clusters found.
```

If you have an existing cluster, you can either deploy to it, or delete it and deploy a new KIND cluster.  The choice is yours.

Let's deploy a KIND cluster named `confluence-cluster`:
```bash
kind create cluster --name confluence-cluster
```

We should see something like the following:
```bash
Cgroup controller detection is not implemented for Podman. If you see cgroup-related errors, you might need to set systemd property "Delegate=yes", see https://kind.sigs.k8s.io/docs/user/rootless/
Creating cluster "confluence-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-confluence-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-confluence-cluster

Thanks for using kind! 😊
```

Checking for clusters:
```bash
kind get clusters
```

We should see our confluence-cluster KIND cluster:
```bash
confluence-cluster
```

Checking with `kubectl`:
```bash
kubectl get nodes
```

We should see our single-node KIND cluster:
```bash
NAME                               STATUS   ROLES           AGE     VERSION
confluence-cluster-control-plane   Ready    control-plane   4m17s   v1.25.3
```

Taking a peek under the hood:
```bash
kubectl get all -A
```

We can see all the pieces that make up our Kubernetes cluster-in-a-container:
```bash
NAMESPACE            NAME                                                           READY   STATUS    RESTARTS   AGE
kube-system          pod/coredns-565d847f94-25q72                                   1/1     Running   0          6m17s
kube-system          pod/coredns-565d847f94-62km2                                   1/1     Running   0          6m17s
kube-system          pod/etcd-confluence-cluster-control-plane                      1/1     Running   0          6m30s
kube-system          pod/kindnet-9dxzc                                              1/1     Running   0          6m17s
kube-system          pod/kube-apiserver-confluence-cluster-control-plane            1/1     Running   0          6m30s
kube-system          pod/kube-controller-manager-confluence-cluster-control-plane   1/1     Running   0          6m30s
kube-system          pod/kube-proxy-ctwqv                                           1/1     Running   0          6m17s
kube-system          pod/kube-scheduler-confluence-cluster-control-plane            1/1     Running   0          6m30s
local-path-storage   pod/local-path-provisioner-684f458cdd-7hj9p                    1/1     Running   0          6m17s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  6m32s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6m31s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      1         1         1       1            1           <none>                   6m27s
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   6m30s

NAMESPACE            NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          deployment.apps/coredns                  2/2     2            2           6m31s
local-path-storage   deployment.apps/local-path-provisioner   1/1     1            1           6m26s

NAMESPACE            NAME                                                DESIRED   CURRENT   READY   AGE
kube-system          replicaset.apps/coredns-565d847f94                  2         2         2       6m17s
local-path-storage   replicaset.apps/local-path-provisioner-684f458cdd   1         1         1       6m17s
```

***Ok, we have a Kubernetes cluster.  What next?***

## Deploying Our Confluence Pod to Kubernetes

Now that we have our KIND cluster deployed, let's try deploying our manifest to it.

Double-checking for our YAML manifest:
```bash
ls -la *.yaml
```

We should see:
```bash
-rw-rw-r-- 1 tdean tdean 1343 Mar 22 19:46 confluence-pod.yaml
```

Deploy our `confluence-pod` pod using `kubectl`:
```bash
kubectl create -f confluence-pod.yaml
```

We should see:
```bash
pod/confluence-pod created
```

Checking for our confluence-pod pod using kubectl:
```bash
kubectl get pods
```

We see:
```bash
NAME             READY   STATUS              RESTARTS   AGE
confluence-pod   0/2     ContainerCreating   0          63s
```

Let's give it a minute.

We can use the `watch` command to keep an eye on our pod:
```bash
watch -n 1 kubectl get pods
```

***If we sit and watch, we'll notice that our pod stays in ContainerCreating status and never makes it to Running.  What could be wrong?***

## Debugging Our YAML Manifest

A good way to get to the bottom of an issue is to check the logs.  We can use `kubectl` to check the pod logs, as well as check the logs for the containers in the pod.

Checking the logs of the `confluence-pod` pod:
```bash
kubectl logs confluence-pod
```

We see:
```bash
Defaulted container "confluence-postgres" out of: confluence-postgres, confluence-server
Error from server (BadRequest): container "confluence-postgres" in pod "confluence-pod" is waiting to start: ContainerCreating
```

Informative, but not terribly helpful at the pod level.  Let's check the container logs.

Checking the logs for the `confluence-server` container:
```bash
kubectl logs confluence-pod -c confluence-server
```

We see:
```bash
Error from server (BadRequest): container "confluence-server" in pod "confluence-pod" is waiting to start: ContainerCreating
```

Checking the logs for the `confluence-postgres` container:
```bash
kubectl logs confluence-pod -c confluence-postgres
```

We see:
```bash
Error from server (BadRequest): container "confluence-postgres" in pod "confluence-pod" is waiting to start: ContainerCreating
```

*Again, informative, but not terribly helpful.  So much for the logs.*

Another source of truth we should check is the output provided by the `kubectl describe` command.  This command gives us a single source of truth, brimming with information on our pod.

Using the `kubectl describe` command:
```bash
kubectl describe pod/confluence-pod
```

We see a plethora of information on our `confluence-pod` pod:
```bash
Name:             confluence-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             confluence-cluster-control-plane/10.89.0.2
Start Time:       Thu, 23 Mar 2023 17:00:44 +0000
Labels:           app=confluence-pod
Annotations:      <none>
Status:           Pending
IP:
IPs:              <none>
Containers:
  confluence-postgres:
    Container ID:
    Image:         docker.io/library/postgres:latest
    Image ID:
    Port:          8090/TCP
    Host Port:     8290/TCP
    Args:
      postgres
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/lib/postgresql/data from home-tdean-confluence-site1-database-host-0 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wzzc6 (ro)
  confluence-server:
    Container ID:
    Image:         docker.io/atlassian/confluence:latest
    Image ID:
    Port:          <none>
    Host Port:     <none>
    Args:
      /entrypoint.py
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/atlassian/application-data/confluence from home-tdean-confluence-site1-data-host-0 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wzzc6 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  home-tdean-confluence-site1-database-host-0:
    Type:          HostPath (bare host directory volume)
    Path:          /home/tdean/confluence/site1/database
    HostPathType:  Directory
  home-tdean-confluence-site1-data-host-0:
    Type:          HostPath (bare host directory volume)
    Path:          /home/tdean/confluence/site1/data
    HostPathType:  Directory
  kube-api-access-wzzc6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason       Age                    From               Message
  ----     ------       ----                   ----               -------
  Normal   Scheduled    5m53s                  default-scheduler  Successfully assigned default/confluence-pod to confluence-cluster-control-plane
  Warning  FailedMount  103s (x10 over 5m53s)  kubelet            MountVolume.SetUp failed for volume "home-tdean-confluence-site1-database-host-0" : hostPath type check failed: /home/tdean/confluence/site1/database is not a directory
  Warning  FailedMount  103s (x10 over 5m53s)  kubelet            MountVolume.SetUp failed for volume "home-tdean-confluence-site1-data-host-0" : hostPath type check failed: /home/tdean/confluence/site1/data is not a directory
  Warning  FailedMount  94s (x2 over 3m50s)    kubelet            Unable to attach or mount volumes: unmounted volumes=[home-tdean-confluence-site1-database-host-0 home-tdean-confluence-site1-data-host-0], unattached volumes=[home-tdean-confluence-site1-database-host-0 kube-api-access-wzzc6 home-tdean-confluence-site1-data-host-0]: timed out waiting for the condition
```

We can see from the Events that we're having issues with our volumes.  There are no `/home/tdean/confluence/site1/dat`a and `/home/tdean/confluence/site1/database` directories on our KIND container node.

***We're going to need to change our storage strategy!***

## Changing Our Storage Volumes to Work With Kubernetes

[Kubernetes Documentation: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)

There are a lot of volume types supported by Kubernetes, as we can see in the Kubernetes documentation.  For this demonstration, we'll go with the tried-and-true emptyDir.

I'm going to change the volume type to emptyDir, with a maximum size of 1Gi, and am going to clean up the volume names a bit.

First, let's delete the pod:
```bash
kubectl delete pod/confluence-pod
```

We should see:
```bash
pod "confluence-pod" deleted
```

*It might take a minute, let it run.*

Checking our work:
```bash
kubectl get pods
```

We should see:
```bash
No resources found in default namespace.
```

Next, we want to edit our YAML manifest and make our volume configuration more Kubernetes-friendly.

Our original volume configuration looks like this:
```bash
  volumes:
  - hostPath:
      path: /home/tdean/confluence/site1/database
      type: Directory
    name: home-tdean-confluence-site1-database-host-0
  - hostPath:
      path: /home/tdean/confluence/site1/data
      type: Directory
    name: home-tdean-confluence-site1-data-host-0
```

Again, I'm going to change the volume type to emptyDir, with a maximum size of 1Gi, and am going to clean up the volume names a bit:

```bash
  volumes:
    - name: home-tdean-confluence-site1-database
      emptyDir:
        sizeLimit: 1Gi
    - name: home-tdean-confluence-site1-data
      emptyDir:
        sizeLimit: 1Gi
```

Let's edit our manifest, using `vi`:
```bash
vi confluence-pod.yaml
```

We should end up with something similar to the following:
```bash
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.4.4
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: confluence-pod
  name: confluence-pod
spec:
  containers:
  - args:
    - postgres
    image: docker.io/library/postgres:latest
    name: confluence-postgres
    ports:
    - containerPort: 8090
      hostPort: 8290
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/lib/postgresql/data
      name: home-tdean-confluence-site1-database
  - args:
    - /entrypoint.py
    image: docker.io/atlassian/confluence:latest
    name: confluence-server
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/atlassian/application-data/confluence
      name: home-tdean-confluence-site1-data
  restartPolicy: Never
  volumes:
    - name: home-tdean-confluence-site1-database
      emptyDir:
        sizeLimit: 1Gi
    - name: home-tdean-confluence-site1-data
      emptyDir:
        sizeLimit: 1Gi
status: {}
```

*Ok, we made the changes.  Let's test!*

Creating our `confluence-pod` pod, using `kubectl`:
```bash
kubectl create -f confluence-pod.yaml
```

We see:
```bash
pod/confluence-pod created
```

Checking the status of our pod:
```bash
kubectl get pods
```

We see:
```bash
NAME             READY   STATUS              RESTARTS   AGE
confluence-pod   0/2     ContainerCreating   0          35s
```

If we wait, we will see the status go to `Error`:
```bash
NAME             READY   STATUS   RESTARTS   AGE
confluence-pod   1/2     Error    0          79s
```

Let's see what a `kubectl describe` tells us about our pod:
```bash
kubectl describe pod/confluence-pod
```

It looks like our volumes are sorted:
```bash
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  94s   default-scheduler  Successfully assigned default/confluence-pod to confluence-cluster-control-plane
  Normal  Pulling    93s   kubelet            Pulling image "docker.io/library/postgres:latest"
  Normal  Pulled     79s   kubelet            Successfully pulled image "docker.io/library/postgres:latest" in 13.521752095s
  Normal  Created    79s   kubelet            Created container confluence-postgres
  Normal  Started    79s   kubelet            Started container confluence-postgres
  Normal  Pulling    79s   kubelet            Pulling image "docker.io/atlassian/confluence:latest"
  Normal  Pulled     39s   kubelet            Successfully pulled image "docker.io/atlassian/confluence:latest" in 40.235172825s
  Normal  Created    39s   kubelet            Created container confluence-server
  Normal  Started    39s   kubelet            Started container confluence-server
```

We're not seeing anything else that might be an issue here, so let's check the pod logs:
```bash
kubectl logs confluence-pod
```

We can see:
```bash
Defaulted container "confluence-postgres" out of: confluence-postgres, confluence-server
Error: Database is uninitialized and superuser password is not specified.
       You must specify POSTGRES_PASSWORD to a non-empty value for the
       superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".

       You may also use "POSTGRES_HOST_AUTH_METHOD=trust" to allow all
       connections without a password. This is *not* recommended.

       See PostgreSQL documentation about "trust":
       https://www.postgresql.org/docs/current/auth-trust.html
```

We can see that Postgres is complaining about needing the POSTGRES_PASSWORD.  Did we supply this in the manifest?

[Define an environment variable for a container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)

Let's edit our manifest, using `vi`:
```bash
vi confluence-pod.yaml
```

We need to add the environment variables in the container spec for the confluence-postgres container:
```bash
...
spec:
  containers:
  - args:
    - postgres
    image: docker.io/library/postgres:latest
    name: confluence-postgres
    env:
      - name: POSTGRES_USER
        value: "confluenceUser"
      - name: POSTGRES_PASSWORD
        value: "confluencePW"
      - name: POSTGRES_DB
        value: "confluenceDB"
    ports:
    - containerPort: 8090
      hostPort: 8290
    resources: {}
...
```

We'll save our manifest, with the environment variables in it.

Let's delete the pod:
```bash
kubectl delete pod/confluence-pod
```

We see:
```bash
pod "confluence-pod" deleted
```

*Ok, we made the changes.  Let's test again!*

Creating our `confluence-pod` pod, using `kubectl`:
```bash
kubectl create -f confluence-pod.yaml
```

We see:
```bash
pod/confluence-pod created
```

Checking our work:
```bash
kubectl get pods
```

We see:
```bash
NAME             READY   STATUS    RESTARTS   AGE
confluence-pod   2/2     Running   0          5s
```

***Our Confluence pod is up and in the Running status!***

Let's take a deeper look at our pod:
```bash
kubectl describe pod/confluence-pod
```

We get a bunch of information about our `confluence-pod` pod:
```bash
Name:             confluence-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             confluence-cluster-control-plane/10.89.0.2
Start Time:       Thu, 23 Mar 2023 19:17:25 +0000
Labels:           app=confluence-pod
Annotations:      <none>
Status:           Running
IP:               10.244.0.9
IPs:
  IP:  10.244.0.9
Containers:
  confluence-postgres:
    Container ID:  containerd://00aae808328eb31366c5b3930e71241dc9b336a1c55581283c59d9fd622e2b6f
    Image:         docker.io/library/postgres:latest
    Image ID:      docker.io/library/postgres@sha256:542f83919d1aa39230742a01a7dbfbe84a5c7c84c27269670ff84c0e8bb656e8
    Port:          8090/TCP
    Host Port:     8290/TCP
    Args:
      postgres
    State:          Running
      Started:      Thu, 23 Mar 2023 19:17:26 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      POSTGRES_USER:      confluenceUser
      POSTGRES_PASSWORD:  confluencePW
      POSTGRES_DB:        confluenceDB
    Mounts:
      /var/lib/postgresql/data from home-tdean-confluence-site1-database (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pxh4n (ro)
  confluence-server:
    Container ID:  containerd://ddde189a66faae362088e28b72d33a44caf1d1fe59e7a258e6dffc75b6f7a6b2
    Image:         docker.io/atlassian/confluence:latest
    Image ID:      docker.io/atlassian/confluence@sha256:e93aaee7e2bae2cdb041720b8e2af9fb59e8073f0a0b557a02e8936ef6615b71
    Port:          <none>
    Host Port:     <none>
    Args:
      /entrypoint.py
    State:          Running
      Started:      Thu, 23 Mar 2023 19:17:26 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/atlassian/application-data/confluence from home-tdean-confluence-site1-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pxh4n (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  home-tdean-confluence-site1-database:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  1Gi
  home-tdean-confluence-site1-data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  1Gi
  kube-api-access-pxh4n:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m18s  default-scheduler  Successfully assigned default/confluence-pod to confluence-cluster-control-plane
  Normal  Pulling    2m18s  kubelet            Pulling image "docker.io/library/postgres:latest"
  Normal  Pulled     2m17s  kubelet            Successfully pulled image "docker.io/library/postgres:latest" in 287.175429ms
  Normal  Created    2m17s  kubelet            Created container confluence-postgres
  Normal  Started    2m17s  kubelet            Started container confluence-postgres
  Normal  Pulling    2m17s  kubelet            Pulling image "docker.io/atlassian/confluence:latest"
  Normal  Pulled     2m17s  kubelet            Successfully pulled image "docker.io/atlassian/confluence:latest" in 266.104183ms
  Normal  Created    2m17s  kubelet            Created container confluence-server
  Normal  Started    2m17s  kubelet            Started container confluence-server
```

Since our `confluence-pod` pod is only available *inside* our KIND cluster, we will use a container to run a `curl` test.  There's a `curl` container image, with the `curl` command baked in, available.  We'll deploy it in a pod, then open a shell in it, where we can run a `curl` against the status URL for our Confluence server, which is `http://<k8s_node_ip>:8290/status`.

First, we'll launch our `curlpod` pod, using the `curlimages/curl container` image:
```bash
 kubectl run curlpod --image=curlimages/curl -i --tty -- sh
```

Once our `curl` container is launched and we're in the shell, let's try a `curl` command against the status URL, *as shown in the example below*:
```bash
If you don't see a command prompt, try pressing enter.
/ $ curl http://10.89.0.2:8290/status
{"state":"FIRST_RUN"}/ $ exit
Session ended, resume using 'kubectl attach curlpod -c curlpod -i -t' command when the pod is running
```

Awesome!  We get the status of `{"state":"FIRST_RUN"}`, which is what we would expect!

Let's go ahead and clean up!

Deleting our pods:
```bash
kubectl delete pod/curlpod
kubectl delete pod/confluence-pod
```

Checking our work:
```bash
kubectl get pods
```

We see:
```bash
No resources found in default namespace.
```

*All our pods are cleaned up!*

Let's delete our KIND cluster:
```bash
kind delete cluster -n confluence-cluster
```

We see:
```bash
Deleting cluster "confluence-cluster" ...
```

Checking our work:
```bash
kind get clusters
```

We see:
```bash
No kind clusters found.
```

We're all cleaned up.

***With a little tweaking and troubleshooting, we deployed our confluence pod pod in Kubernetes.  How cool is that?***

## Summary

So, we've shown that deploying a self-hosted Confluence server doesn't have to be a pain.  We stood up a hosted instance of Confluence in short order, using the power of Podman, pods and containers.  And we used that to generate a YAML manifest, based on our pod's running configuration in Podman, and used Podman to test our YAML manifest.  Finally, we tested and tweaked our YAML manifest to get it working in Kubernetes, learning some K8s troubleshooting along the way.

*What about the `systemd` unit file we created for our `confluence-pod` pod?  When are we going to get to that?*

In the final article in our series, we're going to circle back around and deploy our `confluence-pod` pod `systemd` unit files so that our pod is managed by `systemd`.

See you soon!

*Tom Dean*
