# **KEDA HTTP Add-on Autoscaling on GKE**

In this repository, I will walk through the journey of setting up **KEDA** (Kubernetes Event-Driven Autoscaler) with the **HTTP Add-on** in **Google Kubernetes Engine (GKE)**. The goal was to scale a Python-based Rock-Paper-Scissors application based on incoming HTTP traffic. Along the way, I faced several challenges with cluster resource limits, scaling issues, and the limitations of the **HTTP Add-on** in KEDA, which I tackled through various strategies. This article details the steps, challenges, solutions, and configurations used during the process.

## **Steps Taken:**

### **1. Creating the GKE Cluster with Two Nodes**

The first step was to create a Kubernetes cluster in GKE. This cluster would host the Rock-Paper-Scissors application and later be configured for autoscaling using KEDA.

```bash
gcloud container clusters create my-cluster \
    --zone us-central1-a \
    --num-nodes 2
This created a GKE cluster with 2 nodes, providing enough resources for the application. After the cluster was created, I connected to the cluster using kubectl.

2. Installing KEDA with Server-Side Apply
I installed KEDA in the cluster using server-side apply due to an issue with the CRD YAML being too large, which was putting unnecessary load on the etcd component.

bash
Copy
Edit
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.4.0/keda-operator.yaml
Using server-side apply ensures that the cluster resources are managed in an efficient and non-blocking way. The KEDA operator was successfully installed, and I proceeded to install the necessary ScaledObject for my application.

3. Installing and Confirming ScaledObject
Once KEDA was installed, the next step was to deploy a ScaledObject. A ScaledObject defines the scaling rules for the application based on event sources. Here’s the initial YAML configuration for ScaledObject:

yaml
Copy
Edit
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rock-paper-scissors-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    kind: Deployment
    name: rock-paper-scissors
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: http-add-on
      metadata:
        endpoint: "/"
I confirmed that the ScaledObject was running correctly using kubectl get scaledobjects.

4. Sending HTTP Load and Scaling Issues
Next, I tried sending HTTP load to the application using curl, but the autoscaling did not trigger as expected. The HPA (Horizontal Pod Autoscaler) was not scaling the application, and the CRDs for scaling were stuck in a Pending state.



This led me to investigate the node status to check the cluster’s resource utilization.



5. Resource Constraints and Node Scaling
I ran kubectl top nodes and discovered that the cluster was at 90% CPU utilization. Given that I had already exceeded my free tier and billing wasn’t working (due to using a prepaid card), I wasn’t able to scale the nodes. Since I couldn’t scale up the cluster, I opted for a different strategy.

6. Making the Application Lightweight
In order to reduce CPU consumption, I modified the Python application to be more lightweight. This involved introducing a small delay in the response time to simulate a more realistic load.

Here’s the modified app.py with a delay:

python
Copy
Edit
from flask import Flask
import time

app = Flask(__name__)

@app.route('/')
def index():
    time.sleep(0.2)  # 200 ms delay
    return "Welcome to Rock-Paper-Scissors!"
This change significantly reduced the CPU usage and allowed the cluster to function under the free-tier constraints.

7. Incorrect ScaledObject Configuration (Using HTTPScaledObject)
After making the app lighter, I realized that the ScaledObject I had initially used was not the correct resource for HTTP-based scaling. Instead, I needed to use the HTTPScaledObject.

8. KEDA Slack Community: HTTPScaledObject Recommendation
I reached out to the KEDA community via Slack for clarification on the issue. One of the maintainers suggested that I should use HTTPScaledObject instead of the regular ScaledObject for HTTP-based scaling. This recommendation led me to revise the YAML configuration.



9. HTTPScaledObject Configuration
After understanding that HTTPScaledObject was needed, I updated my YAML to reflect the correct resource type. Below is the updated HTTPScaledObject.yaml:

yaml
Copy
Edit
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: rock-paper-scissors-http
  namespace: default
spec:
  hosts:
    - rock-paper-scissors.local
  scaleTargetRef:
    kind: Deployment
    apiVersion: apps/v1
    name: rock-paper-scissors
    service: rock-paper-scissors-service
    port: 80
  replicas:
    min: 1
    max: 10
Below is the image of the HTTP Add-on scaler and other CRDs running:



10. Running Load Tests and Confirming Scaling
After configuring the HTTPScaledObject, I used hey to simulate 1000 requests to the service:

bash
Copy
Edit
hey -n 1000 -c 50 http://rock-paper-scissors.local
This successfully triggered the Horizontal Pod Autoscaler (HPA) to scale up the pods, but KEDA’s HTTP Add-on did not scale the application working through ingress. The issue persisted despite proper configuration.



11. Reaching Out to KEDA Maintainer
I reached out again to the KEDA Slack channel, asking for help with the HTTP Add-on not scaling. The response from the KEDA maintainer was:

"Hard to tell, the add-on is still alpha/beta so not everything always works."

This made it clear that the HTTP Add-on was still in an unstable state, leading to the conclusion that the add-on might not be fully functional yet.

12. Hypothesis on Light Application Load
My hypothesis was that the app was too lightweight, causing it to process requests too quickly and not generate enough load for KEDA’s HTTP Add-on to scale. To test this, I increased the request rate to 1500 requests per second. However, even with this change, the scaling did not trigger.

Conclusion
Key Learnings:

HTTP-based scaling with KEDA requires the correct resource (HTTPScaledObject), not the generic ScaledObject.
The KEDA HTTP Add-on is still in an alpha/beta state, and its functionality may not always be stable.
Testing load and scaling behavior requires simulating real traffic and understanding the load characteristics of the application.
Challenges:

Scaling failure due to resource constraints in the cluster.
KEDA HTTP Add-on not scaling the app despite correct configuration.
Limited cloud resources due to the prepaid card and billing issues.
Future Steps:

Experiment with scaling triggers and other event-based scaling mechanisms.
Stay updated with the latest stable releases of KEDA and the HTTP Add-on.
Repositories and Resources
KEDA GitHub Repository
Google Kubernetes Engine
This README.md should be included in the GitHub repository as it outlines the journey, challenges, and solutions with KEDA and the HTTP Add-on for autoscaling a Python-based Rock-Paper-Scissors application on GKE.
