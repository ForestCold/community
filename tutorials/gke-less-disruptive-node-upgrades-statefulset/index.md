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

