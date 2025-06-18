## 📦 Mapping Project Code to Kubernetes Pods & Nodes

Imagine your project has this structure:

```
project-root/
├── main.py
├── config.json
└── src/
    ├── preprocess.py
    ├── model.py
    └── utils.py
```

### 🔗 Step-by-Step: How Kubernetes Works with This

1. **Containerization**

   * You package the whole project (code + configs) into a Docker image using a `Dockerfile`.
   * Example:

     ```Dockerfile
     FROM python:3.10
     COPY . /app
     WORKDIR /app
     RUN pip install -r requirements.txt
     CMD ["python", "main.py"]
     ```

2. **Pods**

   * Each Pod runs **one or more containers** (usually one) using the above Docker image.
   * It contains everything: your Python scripts, `config.json`, and dependencies.
   * Think of a Pod as a **running instance of your project**, like a Python process that launched your app.

3. **Nodes**

   * A Node is the VM (Linux host) that runs one or more Pods.
   * Kubernetes Scheduler places your Pod on a Node with enough CPU/memory.
   * Example:

     * Node 1 runs Pods for web app
     * Node 2 runs Pods for background tasks like `preprocess.py`

4. **Real-World Distribution**

   * You might define multiple Deployments in Kubernetes:

     * `web-app-deployment`: runs `main.py`
     * `worker-deployment`: runs `preprocess.py`
     * `model-deployment`: runs `model.py`
   * Each has its own Pod template in YAML and may run on separate Nodes.

### 🔗 Visual Overview

```
+-----------------+        +-----------------+
|   Node 1        |        |   Node 2        |
|-----------------|        |-----------------|
| Pod: Web App    |        | Pod: Worker     |
| Image: main.py  |        | Image: preprocess.py |
+-----------------+        +-----------------+
```

### 🔗 Summary

* **Nodes** = VMs where containers run.
* **Pods** = Encapsulated containers running your project.
* **Your code** lives inside a Docker image, which Pods run.
* **Different parts** of the project can run in **different Pods**, scheduled on **different Nodes**.

---

### 🔗 Kubernetes YAML Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app-container
        image: your-dockerhub-username/web-app:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /app/config.json
          subPath: config.json
      volumes:
      - name: config-volume
        configMap:
          name: web-app-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  labels:
    app: web-app
data:
  config.json: |
    {
      "param1": "value1",
      "param2": "value2"
    }
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

---

### 🔗 Realistic Strategies

#### 🔗 1. One Pod per Script (Recommended)

* Each Pod runs a single Python script.
* All Pods are scheduled on the same Node if there's enough CPU/RAM.
* Kubernetes handles isolation, logs, restarts, and scaling per script.

```
+--------------------+
|      Node (VM)     |
|--------------------|
| Pod A → script1.py |
| Pod B → script2.py |
| ...                |
| Pod T → script20.py|
+--------------------+
```

#### 🔗 2. All Scripts in One Pod (Not Recommended)

* One container runs a master script calling all 20 scripts.
* Failure in one script affects the entire Pod.
* No isolated scaling/logging.

#### 🔗 3. Grouped Pods (Balanced)

* Group similar scripts (e.g., ETL, training, API) into shared Pods.

---

### 🔗 Industry Best Practice

Most modern production systems (AWS, GCP, Azure) treat **each Python script (or task)** as an **independent microservice**, deployed in **its own Pod** for better control and scalability.
