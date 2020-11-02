## Part 3: Kubernetes
### Description
You’ve been asked to containerize and deploy this API to GCP Kubernetes Engine. Please attach the Dockerfile and provide a written step-by-step guide on how you would build the docker image and deploy this to Kubernetes. This workload should be able to scale horizontally and serve 100k concurrent users. 

### Dependencies
#### Google Cloud SDK

- Install and initialize gcloud.
    - `./google-cloud-sdk/bin/gcloud init` 	
    - `gcloud auth configure-docker`

### Steps
 - Create or select a project on the Google Kurbernetes Engine page
	 - Wait for the API and services to come up
 -  Enable billing, if not enabled yet. 
 - Create a **Container Registry** in GCP with the same value as `PROJECT_ID`
 - Obtain the project-id value from Google.
 - Build the image
	 -  Follow the convention grc.io/${PROJECT_ID}/node-webapp:v1
	 - `docker build -t grc.io/${PROJECT_ID}/node-webapp:v1`
 - Push the image to the GCP registry
	 - `docker push grc.io/${PROJECT_ID}/node-webapp:v1`
 - Create GKE cluster
	 - Enable Kubernetes Engine API in GCP  before executing the following commands.
		 - `gcloud config set project $PROJECT_ID`
		 - `gcloud config set compute/zone us-west1-a`
		 - `gcloud container clusters create node-webapp-cluster`
	 - To verify that the K8s cluster was created
		 - `gcloud compute instances list`
 -  Deploy the app to GKE
	 - ` kubectl create deployment hello-app --image=gcr.io/${PROJECT_ID}/node-webapp:v1`
 -  Create an initial number of replicas for the app
	 - `kubectl scale deployment node-webapp --replicas=3`
 -  Add horizontal initial autoscaling based on average 60% CPU usage
	 - `kubectl autoscale deployment node-webapp --cpu-percent=60 --min=1 --max=10`
 
  NOTE: The following steps are not proven since I could not find definite guidance on how to scale based on traffic density and I have only worked with CPU-based performance for autoscaling. 

 - Install the Kubernetes Metrics Server
     - `git clone https://github.com/kubernetes-incubator/metrics-server.git`
     - `cd metrics-server`
     - `kubectl apply -f deploy/kubernetes/`
 -  Update the auto scaling file to include traffic 
	 - `kubectl edit HorizontalPodAutoscaler`
 - Edit the Horizontal Pod AutoScaler (HPA) to include the traffic related scaling
	 - ` kubectl edit hpa.v2beta2.autoscaling` 
	 - This will open a YAML file in an editor. Add the following blocks to
        - spec->metrics block, after Resource->CPU
            ```YAML
            - type: Pods
            pods:
            metric:
                name: packets-per-second
            target:
                type: AverageValue
                averageValue: 10k
            - type: Object
            object:
            metric:
                name: requests-per-second
            describedObject:
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                name: main-route
            target:
                type: Value
                value: 100k
            ```
        - status->currentMetrics block, after Resource->CPU
            ```YAML
            type: Object
            object:
            metric:
                name: requests-per-second
            describedObject:
                apiVersion: networking.k8s.io/v1beta1
                kind: Ingress
                name: main-route
            current:
                value: 100k
            ```

How would you choose:
- Container resource requests and limits 
    - This one is a matter of balancing the expected throughput for each pod vs the ability to horizontally scale if there are spikes in the demand for each. The pods can be made robust enough to handle tens of thousands of concurrent requests or light enough so that many replicas can be spun up to handle the increased demand. 
        - There is also the matter of how many pods will the cluster maintain as a minimum. 
        - I would choose a middle-of-the-road approach and give each pod enough capacity to distribute normal workload across the number of initial pods and make use of the autoscaler to add more capacity as needed. 
    - Without knowing the node instance specs and the scale of the app itself it's hard to give more specific numbers. 
      - An example would be for each node 10 core instance where with 32 GB of RAM 
          - Initial cluster node count is 1 and can scale up to 5.
          - Each node starts with 3 pods and can scale up to 100 pods. 
          - The worst-case scenario is having 100 pods per node.
          - For this, I would set each pod to have a CPU limit of 0.1 and 320Mi of memory. 
- Probes
    - Given that the emphasis of this design is in handling concurrent network  requests, I would select the `HTTPGetAction` probe to run after the pod reaches the `Running` state. 
- Autoscaler
    - If we go with a middle-of-the road approach, I would set up the HorizontalPodAutoscaler to spin up new pods when the CPU utilization goes above 60%, if the app does not have high resource demand per each request
- Cluster node Instance type        
    - To continue with the middle-of-the-road approach, I would select an instance type which is medium size with enough capacity to easily handle the maximum pod replica count before requiring a new replica to be spun up. 