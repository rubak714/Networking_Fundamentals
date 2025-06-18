# 🔗 Deep Dive: How Kubernetes Networking Works

This section explains **how Kubernetes networking works** by breaking down each component of the Kubernetes control plane and its communication process using **IP addresses**, **services**, and **pod-to-pod communication** on an Ubuntu VM.

---

## 🔗 Core Components in Kubernetes Networking

### 1. **API Server** (the central brain 🧠)

* **What it is:** Acts like the receptionist of a Kubernetes cluster.
* **Role:** Receives commands (`kubectl apply`, etc.) and updates the cluster state.
* **IP:** Usually `localhost` inside master node (`127.0.0.1` or `10.x.x.x` if external).
* **Communicates with:** Scheduler, Controller Manager, kubelets, etc.

### 2. **Scheduler**

* **What it is:** Decides **where (which node)** a pod will run.
* **Role:** Based on CPU, RAM availability, affinity rules, etc.
* **How it works:**

  * Talks to the API server to get pending pods
  * Assigns them to nodes by writing binding info back to the API server

### 3. **Nodes (Workers)**

* **What they are:** The VMs or machines that run your **actual applications**.
* Each node runs:

  * **kubelet**: Talks to the API server
  * **kube-proxy**: Handles networking
  * **Container runtime**: (e.g. containerd, Docker)

### 4. **Pods**

* **What they are:** The smallest unit in Kubernetes. Like 1 or more containers sharing an IP and filesystem.
* **Each pod gets a unique IP**, usually in `10.244.0.x` range.
* Communicates directly with other pods **without NAT**.

### 5. **CNI (Container Network Interface)**

* **What it is:** A plugin responsible for assigning IPs to pods.
* **Popular CNI tools:** Flannel, Calico, Weave

---

## 🔗 Visual Diagram: Kubernetes Communication Flow

```mermaid
flowchart TD
    User -->|kubectl apply| API[API Server]
    API --> Scheduler
    Scheduler --> Node1
    Scheduler --> Node2
    Node1 -->|Creates| Pod1[Pod: 10.244.0.2]
    Node2 -->|Creates| Pod2[Pod: 10.244.0.3]
    API --> Kubelet1
    API --> Kubelet2
    Pod1 -->|Direct IP| Pod2
```

---

## 🔗 How IP Addresses Work

| Component  | Description                      | IP Example                       |
| ---------- | -------------------------------- | -------------------------------- |
| API Server | Cluster brain                    | 127.0.0.1 (internal) or 10.x.x.x |
| Pod        | Runs your app                    | 10.244.0.2                       |
| Service    | Stable front for a group of pods | 10.96.0.1                        |
| Node       | The Ubuntu VM or worker          | 192.168.1.100                    |
| NodePort   | Port exposed outside cluster     | 30080                            |

---

## 🔗 Example: What Happens When You Deploy an App

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

### Step-by-step:

1. You run `kubectl create` → API server receives this request
2. Scheduler picks a node and tells kubelet to run the Pod
3. CNI plugin assigns an IP like `10.244.0.2`
4. You expose the pod with `kubectl expose` → creates a **Service** with IP like `10.96.0.1`
5. Kubernetes assigns a **NodePort** like `30080`
6. You access the app at `http://<Ubuntu_VM_IP>:30080`

---

## 🔗 Detailed: Step-by-Step Kubernetes Networking Flow

### 1. 🔗 `kubectl create deployment nginx --image=nginx`

* You (the user) issue a command to create a deployment.
* This command is sent to the **API Server**, which is the front desk of the Kubernetes control plane.
* The API Server validates the command and stores the new deployment in **etcd** (the cluster database).

### 2. 🔗 Scheduler Selects a Node

* The **Scheduler** checks all available **worker nodes** in the cluster.
* It looks at available **CPU**, **memory**, and other criteria.
* It then selects a node (say, Node-1) and updates the deployment info in etcd.

### 3. 🔗 Kubelet Triggers Pod Creation

* On Node-1, the **kubelet** agent is constantly checking the API server.
* It sees it has a new pod to create and instructs the container runtime (e.g., `containerd`) to start the container.

### 4. 🔗 CNI Assigns a Pod IP

* The **Container Network Interface (CNI)** plugin (e.g., Flannel or Calico) assigns the new Pod an IP address.
* Example: `10.244.0.2`
* This IP is routable **within the cluster** but not accessible directly from outside.

### 5. 🔗 You Expose the Pod with a Service

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

* This creates a Kubernetes **Service** that acts like a stable virtual IP (VIP).
* The service is assigned a **ClusterIP** like `10.96.0.1`.
* It acts as a load balancer to forward traffic to the actual pod(s).

### 6. 🔗 NodePort Created

* Because you used `--type=NodePort`, Kubernetes now assigns a high external port (e.g., `30080`) on **every node** in the cluster.
* The **kube-proxy** component watches for this and ensures traffic to `NodeIP:30080` routes to the ClusterIP service, and then to the pod.

### 7. 🔗 Accessing the App from Outside

* Your Ubuntu VM has a public IP like `192.168.1.100`.
* You access the app in your browser:

```
http://192.168.1.100:30080
```

* The request hits the node → matches NodePort → routed to the Service (10.96.0.1) → forwarded to the Pod (10.244.0.2)

---

## 🔗 Summary Diagram

```mermaid
flowchart TD
    User -->|kubectl create| APIServer
    APIServer --> Scheduler
    Scheduler -->|Selects| Node1
    Node1 --> Kubelet
    Kubelet -->|Creates| Pod1[Pod IP: 10.244.0.2]
    Pod1 -->|App Ready| Service[ClusterIP: 10.96.0.1]
    Service -->|Exposed via| NodePort[NodeIP:30080]
    Browser -->|Access| NodePort
```

## 🔗 In Summary:

* **Pods talk to each other directly** using their IPs, thanks to CNI.
* **Services group pods** and provide **stable IPs**.
* **NodePorts expose services** to the outside world.
* The API server orchestrates the entire communication using control-plane components.

Would you like this section merged into your main IP Networking README with visuals?
