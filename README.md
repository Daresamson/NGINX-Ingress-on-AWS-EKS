# NGINX-Ingress-on-AWS-EKS
# **README: Deploying a Web App with NGINX Ingress on AWS EKS**  

## **Introduction**  
This project focuses on deploying an NGINX-based web application on AWS EKS. It implements Kubernetes best practices, including **autoscaling, load balancing, and health probes** to ensure high availability and performance.  

## **Prerequisites**  
Before starting, I ensured the following:  
- **Kubernetes Cluster:** A running AWS EKS cluster.  
- **kubectl:** Installed and configured to access the cluster.  
- **Docker & Minikube:** Installed and used for local testing before EKS deployment.  

---
![Screenshot (139)](https://github.com/user-attachments/assets/cfdd0148-6756-4f7a-b36c-a2d0228ff3b5)

## **Project Directory Structure**  
I created a dedicated directory for organizing Kubernetes manifests:  
```bash
mkdir k8s-lab && cd k8s-lab
mkdir configs
```

---

## **Step 1: Deploying the NGINX Web Application**  
![Screenshot (140)](https://github.com/user-attachments/assets/c5f13b26-1453-4f06-996c-d1118f0f18ce)

### **1.1 Create the Deployment YAML**  
I created `configs/deployment.yaml` with the following content:  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### **1.2 Apply the Deployment**  
![Screenshot (152)](https://github.com/user-attachments/assets/c410696f-2009-4b46-84a5-f84076b27624)

I executed:  
```bash
kubectl apply -f configs/deployment.yaml
```
This created the **nginx-deployment** in my cluster.  

### **1.3 Verify the Deployment**  

To confirm, I ran:  
```bash
kubectl get deployments
```
The deployment was **successfully running**.

---

## **Step 2: Exposing the Application with a Service**  
![Screenshot (141)](https://github.com/user-attachments/assets/b392fce8-4824-4920-a430-382b8f5c4b2e)

### **2.1 Create a ClusterIP Service**  
To expose the NGINX application within the cluster, I used:  
```bash
kubectl expose deployment nginx-deployment --type=ClusterIP --port=80
```

### **2.2 Retrieve the Cluster IP**  
![Screenshot (142)](https://github.com/user-attachments/assets/2e107a09-5026-4edf-b4ae-ffeeeac5e148)

To get the service details, I ran:  
```bash
kubectl get svc nginx-deployment
```
I noted the **Cluster-IP**, which was needed for testing.  

### **2.3 Test Service from Within a Pod**  
![Screenshot (148)](https://github.com/user-attachments/assets/4b626694-5981-44f9-96e7-69a2f2008e47)
![Screenshot (149)](https://github.com/user-attachments/assets/8d5ea113-bf6d-49dc-8938-69c6087fe8c0)

I accessed a running NGINX pod and tested connectivity:  
```bash
kubectl exec -it <nginx-pod> -- curl http://localhost:80
```
The NGINX welcome page confirmed **successful deployment**.

---

## **Step 3: Implementing Horizontal Pod Autoscaling (HPA)**  

### **3.1 Update Deployment with CPU Requests & Limits**  
![Screenshot (152)](https://github.com/user-attachments/assets/c42af025-0153-496e-9651-0f21e34d76de)
![Screenshot (153)](https://github.com/user-attachments/assets/20b0fe3a-9ab6-44de-9084-c4262c3250c5)

I modified `configs/deployment.yaml` to include:  
```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```
Then, I re-applied:  
```bash
kubectl apply -f configs/deployment.yaml
```

### **3.2 Create the HPA**  
![Screenshot (151)](https://github.com/user-attachments/assets/e2618aea-8a1f-4b67-ae0a-291f0174e988)

To enable auto-scaling based on CPU utilization, I ran:  
```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=5
```

### **3.3 Verify HPA Status**  
To confirm, I executed:  
```bash
kubectl get hpa
```
Initially, the **target CPU utilization was low**, so scaling had not yet occurred.  

### **3.4 Simulate Load to Trigger Autoscaling**  
![Screenshot (149)](https://github.com/user-attachments/assets/784834c4-4673-4c97-aefb-b1d7afa276b4)


I simulated high traffic:  
```bash
for i in {1..100}; do curl http://<Cluster-IP>; done
```
After some time, I checked:  
```bash
kubectl get hpa
```
The replicas **scaled up** as expected.  

---
![Screenshot (154)](https://github.com/user-attachments/assets/13a66c2f-7fae-455a-98f2-53023999f6e5)
![Screenshot (155)](https://github.com/user-attachments/assets/e64ae82f-e5d3-4da0-9bf7-b665a34fd6de)

## **Step 4: Configuring Load Balancer for External Access**  

### **4.1 Change Service to LoadBalancer**  
![Screenshot (154)](https://github.com/user-attachments/assets/2b323e3e-a498-4de9-b851-c5e4f73c6ed1)

To expose the application publicly, I first deleted the existing service:  
```bash
kubectl delete svc nginx-deployment
```
Then, I recreated it as a LoadBalancer:  
```bash
kubectl expose deployment nginx-deployment --type=LoadBalancer --port=80
```

### **4.2 Retrieve the External IP**  
I obtained the **public IP**:  
```bash
kubectl get svc nginx-deployment
```

### **4.3 Test External Access**  
I accessed NGINX via browser or curl:  
```bash
curl http://<External-IP>
```
The NGINX welcome page **loaded successfully**, confirming public availability.

---

## **Step 5: Adding Liveness and Readiness Probes**  
![Screenshot (152)](https://github.com/user-attachments/assets/df597ed0-a353-470e-a5ed-95bd57628f7a)

### **5.1 Update Deployment with Probes**  
![Screenshot (153)](https://github.com/user-attachments/assets/e3b996ff-dad8-4464-9412-d0f3bed45016)

I modified `configs/deployment.yaml` to include:  
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```
Then, I re-applied:  
```bash
kubectl apply -f configs/deployment.yaml
```

### **5.2 Verify Probes**  
![Screenshot (154)](https://github.com/user-attachments/assets/2450d8c3-5502-4395-9c7a-d5373a04ce11)
![Screenshot (155)](https://github.com/user-attachments/assets/50da0b72-a713-451b-bec5-a9e953a1dcab)


To check if the probes were active, I described a pod:  
```bash
kubectl describe pod <pod-name>
```

### **5.3 Simulate Failure and Observe Recovery**  
![Screenshot (156)](https://github.com/user-attachments/assets/7275342f-4de0-43a1-a457-022dbe84b5c3)


To test self-healing, I manually stopped NGINX in one pod:  
```bash
kubectl exec <pod-name> -- pkill nginx
```
The **liveness probe detected the failure** and restarted the pod automatically.

---

## **Step 6: Cleaning Up Resources**  

### **6.1 Delete All Kubernetes Objects**  
![Screenshot (156)](https://github.com/user-attachments/assets/1292a770-2507-4d7d-8fe1-c91aff92982f)
![Screenshot (157)](https://github.com/user-attachments/assets/c7026ed7-9982-4a30-bef7-8c028811de3d)

To remove all resources, I executed:  
```bash
kubectl delete hpa nginx-deployment
kubectl delete svc nginx-deployment
kubectl delete deployment nginx-deployment
```

### **6.2 Verify Cleanup**  
![Screenshot (158)](https://github.com/user-attachments/assets/1192c09d-19bf-4bb4-a6ab-e071257d1e90)


I confirmed that all resources were deleted:  
```bash
kubectl get all
```
The output confirmed that **no resources remained**.

---

## **Final Thoughts & Key Learnings**  
In this project, I successfully:  
✅ Deployed an **NGINX web application** on AWS EKS.  
✅ Implemented **ClusterIP, LoadBalancer, and NodePort services** for internal and external access.  
✅ Configured **Horizontal Pod Autoscaling (HPA)** to handle increased traffic loads.  
✅ Added **liveness and readiness probes** for self-healing and high availability.  

This setup ensures **scalability, reliability, and resilience** for a production-grade Kubernetes application. 🚀

