## Objective
* TBD

## Before you begin

We recommend that you use 
[Cloud Shell](https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app#option_a_use_google_cloud_shell) 
to run the commands in this tutorial. As an alternative, you can use your own workstation to run commands locally, in which
case you would need to install the
[required tools](https://cloud.google.com/kubernetes-engine/docs/tutorials/hello-app#option_b_use_command-line_tools_locally).

This tutorial requires a Google Cloud project. You can use an existing project or 
[create a new project](https://cloud.google.com/resource-manager/docs/creating-managing-projects#creating_a_project).

This tutorial follows the [Use surge upgrades to decrease disruptions from GKE node upgrades](https://cloud.google.com/community/tutorials/gke-less-disruptive-node-upgrades) tutorial.
We recommend that you complete the previous tutorial before starting this one. If you didn't complete the [Use surge upgrades to decrease disruptions from GKE node upgrades](https://cloud.google.com/community/tutorials/gke-less-disruptive-node-upgrades) tutorial and would like to immediately start this tutorial, then you can run the following commands to create a cluster with the `hello-app` application running on it:

1.  Clone the repository and navigate to the app directory:

        git clone https://github.com/GoogleCloudPlatform/kubernetes-engine-samples
        cd kubernetes-engine-samples/hello-app

1.  Update the `hello-app` source code to use a limited resource and provide health signals based on resource availability, you just need to download the updated version of
[`main.go`] and overwrite your local copy with it: 

        $ curl https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/gke-less-disruptive-node-upgrades-statefulset/main.go -O

    You can check [Use surge upgrades to decrease disruptions from GKE node upgrades](https://cloud.google.com/community/tutorials/gke-less-disruptive-node-upgrades) for more details.

1.  Set a variable for the
    [project ID](https://cloud.google.com/resource-manager/docs/creating-managing-projects#identifying_projects)
    by replacing `[PROJECT_ID]` with your project ID in this command:

        export PROJECT_ID=[PROJECT_ID]
        
1.  Build and push the image:

        docker build -t gcr.io/${PROJECT_ID}/hello-app:v1 .
        gcloud auth configure-docker
        docker push gcr.io/${PROJECT_ID}/hello-app:v1

1.  Create a cluster

        gcloud config set project $PROJECT_ID
        gcloud config set compute/zone us-central1-a
        gcloud container clusters create hello-cluster --machine-type=g1-small --num-nodes=3
	
1.  Deploy and expose the application:

        gcloud container clusters get-credentials hello-cluster
        kubectl create deployment hello-web --image=gcr.io/${PROJECT_ID}/hello-app:v1
        kubectl expose deployment hello-web --type=LoadBalancer --port 80 --target-port 8080
        
## Costs

In this tutorial, you create a GKE cluster with 3 g1-small virtual machine instances. The total cost of the resources used
for this tutorial is estimated to be less than $0.10. For details, see
[VM instances pricing](https://cloud.google.com/compute/vm-instance-pricing).

## Deploy a redis cluster

In this section, you will deploy a redis cluster with 3 master nodes and 3 slave nodes.

1.  Create a config map `redis-configmap.yaml` for redis’s configuration, remember to set `cluster-enabled` to `yes`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  redis.conf: |+
    cluster-enabled yes
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    appendonly yes
    protected-mode no
    dir /data
    port 6379
```

2.  Create a statefulset `redis-cluster.yaml` with [volumeClaimTemplate](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). Add pod anti-affinity rule to spread pods across kubernetes nodes.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis-service"
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: "redis"
        command:
          - "redis-server"
        args:
          - "/conf/redis.conf"
          - "--protected-mode"
          - "no"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

3.  Create a headless service redis-service.yaml for redis’ nodes (pods in kubernetes) connection. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  clusterIP: None
  ports:
  - name: redis-port
    port: 6379
    protocol: TCP
    targetPort: 6379
  selector:
    app: redis
    appCluster: redis-cluster
  sessionAffinity: None
  type: ClusterIP
```

4.  Apply `redis-configmap.yaml`,  `redis-cluster.yaml` and `redis-service.yaml` to GKE

```shell
$ kubectl apply -f redis-configmap.yaml
$ kubectl apply -f redis-cluster.yaml
$ kubectl apply -f redis-service.yaml
```

5.  Verify the pods are running, and PersistentVolumes created as expected

```shell
$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
redis-cluster-0        1/1     Running   0          100m
redis-cluster-1        1/1     Running   0          100m
redis-cluster-2        1/1     Running   0          105m
redis-cluster-3        1/1     Running   0          100m
redis-cluster-4        1/1     Running   0          96m
redis-cluster-5        1/1     Running   0          108m

$ kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM    STORAGECLASS   REASON   AGE
pvc-23e4af9e-4ba2-433d-a0c5-d9c7a22434d9   1Gi        RWO            Delete           Bound    default/data-redis-cluster-2   standard                169m
pvc-754765fe-fea2-409d-a0d4-d5912ed077df   1Gi        RWO            Delete           Bound    default/data-redis-cluster-1   standard                169m
pvc-b6f5d1ea-fc3a-41db-922c-e2540b1148fd   1Gi        RWO            Delete           Bound    default/data-redis-cluster-5   standard                167m
pvc-c36110d1-2d8a-4dc7-9d80-382a031364db   1Gi        RWO            Delete           Bound    default/data-redis-cluster-4   standard                168m
pvc-e59923e3-3759-4d2b-ad56-e17b1842ac4e   1Gi        RWO            Delete           Bound    default/data-redis-cluster-0   standard                169m
pvc-f719a535-7510-4e79-8ce4-89b4edd77d47   1Gi        RWO            Delete           Bound    default/data-redis-cluster-3   standard                168m
```

6. Form redis cluster, set first three redis nodes as master, and last three redis nodes as slave

```
$ kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
```

7. Verify the redis cluster successfully set up
```
$ kubectl exec -it redis-cluster-0 -- redis-cli cluster info
$ kubectl exec -it redis-cluster-0 -- redis-cli role

```

## Modify hello-app to work with redis-cluster

In this section, you modify the `hello-app` source code to interact with a 

If you don't want to edit the source code manually, you can download the updated version of
[`main.go`](https://github.com/GoogleCloudPlatform/community/blob/master/tutorials/gke-less-disruptive-node-upgrades-statefulset/main.go)
and overwrite your local copy with it:

```shell
$ curl https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/gke-less-disruptive-node-upgrades-statefulset/main.go -O
```

Add a call to redis-client in hello function in [`main.go`]. This function uses redis as a cache database, counts the number of requests it receives and prints out the number on the website. If the redis service works well, the number should keep increasing and never reset to zero. 

```go
	func hello(w http.ResponseWriter, r *http.Request) {
  
		log.Printf("Serving request: %s", r.URL.Path)
    
		if !pool.alloc() {
			w.WriteHeader(http.StatusServiceUnavailable)
			w.Write([]byte("503 - Error due to tight resource constraints in the pool!\n"))
			return
		} else {
			defer pool.release()
		}
		client := redis.NewClusterClient(&redis.ClusterOptions{
			Addrs: []string{"redis-service:6379"},
		})
    
		count, err := client.Incr("hits").Result()
		if err != nil {
			w.Write([]byte("503 - Error due to redis cluster broken!\n"))
			return
		}
        defer client.Close()
    
		fmt.Fprintf(w, "I have been hit [%v] times since deployment!", count)
	}
```

### Deploy the modified application

Since you are done with source code changes, you can build, push, and deply the new image.

1.  Modify the Dockerfile to get import from git

        FROM golang:alpine
        RUN apk add --no-cache git
        ADD . /go/src/hello-app
        WORKDIR /go/src/hello-app
        RUN go get -d -v ./...
        RUN go install hello-app

        FROM alpine:latest
        COPY --from=0 /go/bin/hello-app .
        ENV PORT 8080
        CMD ["./hello-app"]


1.  Set a variable for your project ID, replacing `[PROJECT_ID]` with your project ID:

        export PROJECT_ID=[PROJECT_ID]

1.  Build the new image:

        docker build -t gcr.io/${PROJECT_ID}/hello-app:v2-surge-stateful .
	
1.  Push the new image:

        docker push gcr.io/${PROJECT_ID}/hello-app:v2-surge-stateful

1.  Update the deployment to use the new image:

        kubectl set image deployment/hello-web hello-app=gcr.io/${PROJECT_ID}/hello-app:v2-surge-stateful
	
### Verify the modified application

Verify that the server can respond to requests and whether the health check reports the application being healthy. 

1.  Get information about the service:

        kubectl get service hello-web
	
    The output should look like this:
    
        NAME           TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
        hello-web   LoadBalancer   10.9.9.100   34.71.2.54   80:32309/TCP   1d
	
1.  Verify that the server can respond to requests:

        curl http://34.71.2.54
	
    The output should look like this:
    
        I have been hit [1] times since deployment!

1.  Check the health of the application:

        curl http://34.71.2.54/healthz
	
    The response should be `Ok`.

#### Add more replicas, configure pod anti-affinity, readiness probe


You can change the number of replicas to 3 in the `spec` section of the `hello_server_with_resource_pool.yaml` file:

```yaml
spec:
  replicas: 3 
  selector:
    matchLabels:
      app: hello-web
```

To ensure that each replica is scheduled on a different node, use
[pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/), as configured in this section of 
the `hello_server_with_resource_pool.yaml` file: 

[embedmd]:# (hello_server_with_resource_pool.yaml /^.*# Pod anti affinity config START/ /# Pod anti affinity config END/)
```yaml
      # Pod anti affinity config START
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - hello-web
            topologyKey: kubernetes.io/hostname
      # Pod anti affinity config END
```

To ensure that requests are routed to replicas that have capacity available, use
[readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes), as configured in this section of the `hello_server_with_resource_pool.yaml` file:

[embedmd]:# (hello_server_with_resource_pool.yaml /^.*# Readiness probe config START/ /# Readiness probe config END/)
```yaml
        # Readiness probe config START
        readinessProbe:
          failureThreshold: 1 
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 1
          periodSeconds: 1
          successThreshold: 1
          timeoutSeconds: 1
        # Readiness probe config END
```

### Run tests


#### Download shell scripts

Download two small shell scripts, one to generate load and another to measure error rate:

```shell
curl https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/gke-less-disruptive-node-upgrades-stateful/generate_load.sh -O
curl https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/gke-less-disruptive-node-upgrades-stateful/print_error_rate.sh -O
chmod u+x generate_load.sh print_error_rate.sh
```

#### Test success rate

Now you can start sending traffic with a given frequency, measured in queries per second (QPS). 

1.  Get information about the service:

        kubectl get service hello-web
	
    The output should look like this:
    
        NAME           TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
        hello-web   LoadBalancer   10.9.9.100   34.71.2.54   80:32309/TCP   25d

1.  Set variables for the IP address and number of queries per second:

        export IP=34.71.2.54
        export QPS=40

1.  Run the `generate_load` script, which uses `curl` to send traffic and sends the responses to a file for further
    processing:

        ./generate_load.sh $IP $QPS 2>&1

1.  Check the error rate with the `print_error_rate.sh` script, which calculates error rates based on the number of errors:

        watch ./print_success_rate.sh

#### Content of the `generate_load` script

[embedmd]:# (generate_load.sh)
```sh
# Usage:
#  generate_load.sh <IP> <QPS>_
#
# Sends QPS number of HTTP requests every second to http://<IP>/ URL.
# Saves the responses into the current directory to a file named "output".

IP=$1
QPS=$2

while true
  do for N in $(seq 1 $QPS)
    do curl -I -m 5 -s -w "%{http_code}\n" -o /dev/null http://${IP}/ >> output &
    done
  sleep 1
done

```

#### Content of the `print_success_rate` script

[embedmd]:# (print_error_rate.sh)
```
#!/bin/bash

TOTAL=$(cat output | wc -l); SUCCESS1=$(grep "200" output |  wc -l)
RATE=$(($SUCCESS1 * 100 / TOTAL))
echo "Success rate: $SUCCESS1/$TOTAL (${RATE}%)"
```

#### Generate traffic that the system can handle

Test the failure case by sending more traffic than a single pod can serve.
One pod has a resource pool of size 50 and ~1 second processing time. The health checks consider a pod healthy only if the 
node pool is less than 90% utilized (has less than 45 resources in use). This gives a total of ~3x45 = 135 QPS load that
the system can handle. Use 120 QPS for this test.

1.  Stop the `generate_load` script by pressing Ctrl+C.

1.  Reset the error rate statistics by deleting the output file:

        rm output

1.  Set the `QPS` variable to a higher value to increase the number of queries per second:

        export QPS=120

1.  Run the script with the new value:

        ./generate_load.sh $IP $QPS 2>&1

1.  Check the error rate, which should be greater:

        ./print_success_rate.sh
	
    You should see output like the following:
    
        Success rate: 2836/2846 (99%)

#### Demonstrate the failure case with a higher traffic rate

As demonstration for the failure case, you can increase QPS to 160.

1.  Set the number of queries per second:

        export QPS=160
	
1.  Run the `generate_load` script:

        ./generate_load.sh $IP $QPS 2>&1

1.  Check the error rate:

        ./print_success_rate.sh
	
    The error rate should have increased, with output like the following:

        Success rate: 7410/7840 (94%)


## Test the impact of upgrades on application availability

This section demonstrates how the loss of capacity may hurt the service during a node upgrade if there is no extra 
capacity made available in the form of surge nodes. Before surge upgrades were introduced, every upgrade of a node pool
involved the temporary loss of a single node, because every node had to be recreated with a new image with the new version. 

Some users were increasing the size of the node pool by one before upgrades and then restoring the size after a 
successful upgrade. However, this is error-prone manual work. Surge upgrades makes it possible to do this automatically and
reliably.

### Upgrade the node pool without surge nodes

In this section of the tutorial, you open multiple terminals. Where it is relevant, the commands are labeled. 
For example, the following precedes commands that you enter in the first terminal that you open: "In **terminal 1**:" 

1.  To begin without surge upgrades, make sure that your node pool has zero surge nodes configured:

        $ gcloud beta container node-pools update default-pool --max-surge-upgrade=0 --max-unavailable-upgrade=1 --cluster=hello-cluster

1.  Clear the output file from earlier runs and start watching the error rate.

    In **terminal 1**: 

        rm output
        watch ./print_success_rate.sh

1.  Watch the state of the nodes and pods to follow the upgrade process closely.

    In **terminal 2**:

        watch 'kubectl get nodes,pods -o wide'

1.  Start sending traffic.

    In **terminal 3**:

        export QPS=60
        ./generate_load.sh $IP $QPS 2>&1

1.  Find a lower minor version and then start an upgrade.

    In **terminal 4**:
    
        V=$(gcloud container node-pools  describe default-pool --cluster=hello-cluster | grep version | sed -E "s/version: 1\.([^\.]+)\..*/\1/" | tr -d '\n');  V=$((V-1)); echo "1.$V"
        gcloud container clusters upgrade hello-cluster --cluster-version=$V --node-pool=default-pool

    This operation may take 10-15 minutes to complete.

    Note on `cluster-version` used: It is not possible to upgrade nodes to a higher version than the master, but it is 
    possible to downgrade them. In this example, you most likely started with a cluster that had nodes already on the master 
    version. In that case, you can pick any lower version (minor or patch) to perform a downgrade. In the next section, you 
    can bring the nodes back to their original version with an upgrade.

    Notice that the pod on the first node to be updated gets evicted and remains unschedulable while GKE is re-creating the
    node (since there is no node available to schedule the pod on). This reduces the capacity of the entire cluster and 
    leads to a higher error rate (~20% instead of ~1% in your earlier tests with the same traffic). The same happens with 
    subsequent nodes, as well (that is, the evicted pod remains unschedulable until the node upgrade finishes and the node 
    becomes ready).

    When the upgrade is complete, you should see a higher error rate in **terminal 1**, like the following:

        Success rate: 30415/41880 (72%)
	
### Upgrade the node pool with surge nodes

With surge upgrades, you can upgrade nodes so that the node pool doesn’t lose any capacity. 

1.  Configure the node pool:

        gcloud beta container node-pools update default-pool --max-surge-upgrade=1 --max-unavailable-upgrade=0 --cluster=hello-cluster
	
1.  Clear the output from earlier runs and watch the error rate.

    In **terminal 1**:

        rm output
        watch ./print_error_rate.sh

1.  Watch the state of the nodes and pods to follow the upgrade process.

    In **terminal 2**:

        watch 'kubectl get nodes,pods -o wide'

1.  Start sending traffic:

    In **terminal 3**:

        export QPS=60
        ./generate_load.sh $IP $QPS 2>&1

1.  Find the master version and start an upgrade.

    In **terminal 4**:

        V=$(gcloud container clusters describe hello-cluster | grep "version:" | sed "s/version: //")
        gcloud container clusters upgrade hello-cluster --cluster-version=$V --node-pool=default-pool

    This operation may take 10-15 minutes to complete.

    Note on `cluster-version`: In the first command, you can use version of your master.

    When the upgrade is complete, you should see a lower error rate in **terminal 1**, like the following:

        Error rate: 20094/39900 (50%)

Notice that the pods remain in the running state while GKE is bringing up and registering a new node. Also notice that the 
error rate was still higher than the one we saw when there was no upgrade running. This points out an important detail that 
the *pods still need to be moved from one node to another*. Although we have sufficient compute capacity to schedule an
evicted pod, stopping and starting it up again takes time, which causes disruption. 

The error rate can be reduced further by increasing the number of replicas, so workload can be served even if a pod is
restarted. Also, you can declare the number of pods required to serve requests using
[PodDisruptionBudget (PDB)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/). With a pod disruption budget, 
you can cover more failure cases. For example, in the case of an involuntary disruption (like a node error), pod eviction 
won't start until the ongoing disruption is over (for example, the failed node is repaired), so restarting the pod won't 
cause the number of replicas to be lower than required by PDB.

## Conclusion

TBD

## Cleaning up

To avoid incurring charges to your Google Cloud account for the resources used in this tutorial, you can delete the 
resources that you created.

Deleting the cluster removes all resources used in this demo:

    gcloud container clusters delete hello-cluster

