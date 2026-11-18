# 🧪 DEVOPS PRACTICAL 10

## Container Orchestration Using Kubernetes with Docker Desktop

> **Objective:** Deploy a containerized Node.js application using
> Kubernetes, manage Pods with a Deployment, expose the application
> through a Service, and demonstrate scaling and self-healing.

------------------------------------------------------------------------

## 📌 Practical at a glance

  Item                                Value
  ----------------------------------- -------------------------
  Platform                            Docker Desktop
  Kubernetes CLI                      `kubectl`
  Docker image                        `my-node-app`
  Application port inside container   `5000`
  Deployment                          `myapp-deployment`
  Service                             `myapp-service`
  Local browser address               `http://localhost:8080`

**Assumption:** Your existing `my-node-app` image runs the Node.js
server on port `5000` and the server listens on `0.0.0.0`.

------------------------------------------------------------------------

# 1. Understand the architecture

``` text
                  YOUR WINDOWS COMPUTER
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   Browser: http://localhost:8080                           │
│                  │                                         │
│                  ▼                                         │
│          kubectl port-forward                              │
│                  │                                         │
│                  ▼                                         │
│          Kubernetes Service                                │
│             myapp-service                                  │
│                  │                                         │
│          ┌───────┴────────┐                                │
│          ▼                ▼                                │
│       ┌───────┐        ┌───────┐                            │
│       │ Pod 1 │        │ Pod 2 │                            │
│       │ Node  │        │ Node  │                            │
│       │  App  │        │  App  │                            │
│       └───────┘        └───────┘                            │
│          ▲                ▲                                │
│          └────────┬───────┘                                │
│                   │                                        │
│          Deployment: 2 replicas                            │
│          myapp-deployment                                  │
└────────────────────────────────────────────────────────────┘
```

### Key terms

  -----------------------------------------------------------------------
  Term                                Simple meaning
  ----------------------------------- -----------------------------------
  **Cluster**                         The Kubernetes environment that
                                      manages workloads.

  **Node**                            A machine in the Kubernetes
                                      cluster.

  **Pod**                             The smallest deployable Kubernetes
                                      unit; it can contain one or more
                                      containers.

  **Deployment**                      Maintains the desired number of Pod
                                      replicas.

  **Service**                         Provides a stable network endpoint
                                      for a group of Pods.

  **kubectl**                         Command-line tool used to
                                      communicate with Kubernetes.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 2. Start Kubernetes in Docker Desktop

1.  Open **Docker Desktop**.
2.  Find **Kubernetes** in the sidebar or settings.
3.  If your version shows **Create Cluster**, create a local cluster.
    The `Kubeadm` option is suitable for a simple single-node setup.
4.  If your version has the older settings interface, open **Settings →
    Kubernetes**, enable Kubernetes, and select **Apply & Restart**.
5.  Wait until the Kubernetes cluster reports that it is running.

> Menu names vary between Docker Desktop versions. Use the Kubernetes
> screen/settings available in your installation.

## Verify the cluster

Open PowerShell or the VS Code terminal.

### ▶ COPY & RUN

``` powershell
kubectl config use-context docker-desktop
kubectl version --client
kubectl get nodes
```

**Expected:** A node should appear with status `Ready`.

If `kubectl` is not recognized, confirm Kubernetes is enabled and that
`kubectl` is installed and available on your PATH.

------------------------------------------------------------------------

# 3. Check your Docker image

Make sure the image from Practical 7 is available.

### ▶ COPY & RUN

``` powershell
docker images
```

Confirm that `my-node-app` appears in the list.

> **Image availability note:** The Kubernetes cluster must be able to
> access the image. With the standard Docker Desktop Kubernetes setup, a
> locally built image is commonly available. If a Pod reports
> `ImagePullBackOff` or `ErrImagePull`, inspect the Pod events.
> Depending on the cluster type, you may need to load the image using
> the supported workflow or push it to a container registry.

------------------------------------------------------------------------

# 4. Create the Kubernetes Deployment

In your project folder (for example, `E:\DockerPractical`), create a
file named **`deployment.yaml`**.

## 📄 deployment.yaml --- COPY THE COMPLETE FILE

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: my-node-app
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
```

### What does this YAML do?

  -----------------------------------------------------------------------
  YAML setting                        Meaning
  ----------------------------------- -----------------------------------
  `kind: Deployment`                  Creates a Deployment resource.

  `replicas: 2`                       Asks Kubernetes to maintain two
                                      Pods.

  `selector`                          Identifies the Pods managed by this
                                      Deployment.

  `template`                          Defines how each Pod should be
                                      created.

  `image: my-node-app`                Uses your existing Docker image.

  `containerPort: 5000`               Documents the application port
                                      inside the container.
  -----------------------------------------------------------------------

> **Important:** `containerPort` does not change your Node.js server's
> listening port. Your application must actually listen on port `5000`
> and bind to `0.0.0.0`.

## Apply the Deployment

Run this command from the folder containing `deployment.yaml`.

### ▶ COPY & RUN

``` powershell
kubectl apply -f deployment.yaml
```

**Expected output:**

``` text
deployment.apps/myapp-deployment created
```

(The message may say `configured` or `unchanged` if the resource already
exists.)

## Verify the Deployment and Pods

### ▶ COPY & RUN

``` powershell
kubectl get deployments
kubectl get pods
```

**Expected:** The Deployment should show `2/2` ready replicas. Both Pods
should eventually show `1/1` ready and `Running`.

------------------------------------------------------------------------

# 5. Create a Kubernetes Service

A Service gives you a stable way to reach the application Pods.

Create another file named **`service.yaml`** in the same folder.

## 📄 service.yaml --- COPY THE COMPLETE FILE

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30007
```

### Understand the Service ports

  -----------------------------------------------------------------------
  Setting                             Meaning
  ----------------------------------- -----------------------------------
  `type: NodePort`                    Exposes the Service through a port
                                      on the cluster node.

  `selector`                          Selects Pods labelled `app: myapp`.

  `port: 5000`                        Port exposed by the Service.

  `targetPort: 5000`                  Port where the application listens
                                      in the Pod.

  `nodePort: 30007`                   Requested external node port.
  -----------------------------------------------------------------------

## Apply and verify the Service

### ▶ COPY & RUN

``` powershell
kubectl apply -f service.yaml
kubectl get services
```

**Expected:** `myapp-service` should appear. Its port information should
include a mapping similar to `5000:30007/TCP`.

------------------------------------------------------------------------

# 6. Open the application in your browser

For a straightforward local test, use port forwarding.

### ▶ COPY & RUN

``` powershell
kubectl port-forward service/myapp-service 8080:5000
```

**Keep this terminal open while testing.**

Now open your browser:

**http://localhost:8080**

Expected response (if your `app.js` returns this message):

``` text
Hello from Docker!
```

> Port forwarding connects your computer's port `8080` to Service port
> `5000`. Press `Ctrl + C` in the port-forward terminal to stop
> forwarding. This does not delete the Deployment or Service.

------------------------------------------------------------------------

# 7. Demonstrate scaling

Scaling means increasing or decreasing the number of application
replicas.

Initially, the Deployment requests two Pods. Now increase it to three.

### ▶ COPY & RUN

``` powershell
kubectl scale deployment myapp-deployment --replicas=3
```

Verify:

``` powershell
kubectl get deployments
kubectl get pods
```

**Expected:** Kubernetes creates a third Pod and works to maintain three
replicas.

------------------------------------------------------------------------

# 8. Demonstrate self-healing

Self-healing means Kubernetes works to restore the desired state when a
managed Pod is deleted or fails.

### Step A --- List the Pods

``` powershell
kubectl get pods
```

### Step B --- Copy one Pod name

Example only:

``` text
myapp-deployment-xxxxxxxxxx-abcde
```

### Step C --- Delete that Pod

Replace the placeholder with the actual Pod name from your terminal.

### ▶ COPY & RUN

``` powershell
kubectl delete pod <POD_NAME>
```

### Step D --- Check the Pods again

``` powershell
kubectl get pods
```

**Expected:** The Deployment controller creates a replacement Pod so
that the desired replica count is maintained. The replacement Pod will
usually have a different name.

✅ **This demonstrates Kubernetes self-healing.**

------------------------------------------------------------------------

# 9. Clean up after the practical

First, stop port forwarding by pressing **Ctrl + C** in the terminal
where it is running.

Delete the Service:

``` powershell
kubectl delete -f service.yaml
```

Delete the Deployment and its Pods:

``` powershell
kubectl delete -f deployment.yaml
```

Verify cleanup:

``` powershell
kubectl get services
kubectl get deployments
kubectl get pods
```

> Do not use `docker rm` to remove Kubernetes-managed Pods. Kubernetes
> manages those resources.

------------------------------------------------------------------------

# 10. Expected result

The Node.js Docker image was deployed to a local Kubernetes cluster
running in Docker Desktop. A Deployment maintained multiple Pod
replicas, a Service provided access to the application, scaling
increased the number of replicas, and deleting a Pod demonstrated
automatic replacement (self-healing).

------------------------------------------------------------------------

# ⚡ Quick command sheet

Use this section for quick revision during the practical.

  ----------------------------------------------------------------------------------------------
  Task                                Command
  ----------------------------------- ----------------------------------------------------------
  Select Docker Desktop context       `kubectl config use-context docker-desktop`

  Check cluster nodes                 `kubectl get nodes`

  List Deployments                    `kubectl get deployments`

  List Pods                           `kubectl get pods`

  List Services                       `kubectl get services`

  Create/update Deployment            `kubectl apply -f deployment.yaml`

  Create/update Service               `kubectl apply -f service.yaml`

  Access app locally                  `kubectl port-forward service/myapp-service 8080:5000`

  Scale to 3 Pods                     `kubectl scale deployment myapp-deployment --replicas=3`

  Inspect a Pod                       `kubectl describe pod <POD_NAME>`

  View Pod logs                       `kubectl logs <POD_NAME>`

  Delete Service                      `kubectl delete -f service.yaml`

  Delete Deployment                   `kubectl delete -f deployment.yaml`
  ----------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 🛠️ Troubleshooting

### 1. `kubectl` is not recognized

-   Ensure Kubernetes is enabled in Docker Desktop.
-   Check that `kubectl` is installed and available on PATH.
-   Restart the terminal after installation/configuration.

### 2. Node status is not `Ready`

-   Wait for the cluster to finish starting.
-   Check Kubernetes status in Docker Desktop.
-   Ensure Docker Desktop has sufficient system resources.

### 3. Pod shows `ImagePullBackOff` or `ErrImagePull`

Inspect the Pod:

``` powershell
kubectl describe pod <POD_NAME>
```

Check the **Events** section. Confirm the image name and make sure the
cluster can access the image. Use a registry or the image-loading method
supported by your cluster type if necessary.

### 4. Pod shows `CrashLoopBackOff`

Check application logs:

``` powershell
kubectl logs <POD_NAME>
```

Confirm that: - The Node.js application starts successfully. - The
application listens on port `5000`. - The server binds to `0.0.0.0`.

### 5. Browser cannot connect

Check:

``` powershell
kubectl get pods
kubectl get services
kubectl get endpoints myapp-service
```

Confirm that Pods are Ready, the Service selects the correct Pods, and
the `port-forward` command is still running.

------------------------------------------------------------------------

# 🎓 Viva questions and answers

**Q1. What is Kubernetes?**\
Kubernetes is an open-source platform for automating the deployment,
scaling, and management of containerized applications.

**Q2. What is a Pod?**\
A Pod is the smallest deployable unit in Kubernetes. It can contain one
or more containers.

**Q3. What is a Deployment?**\
A Deployment manages Pods and maintains the desired number of replicas.

**Q4. What is a Service?**\
A Service provides a stable network endpoint for accessing a group of
Pods.

**Q5. What is kubectl?**\
`kubectl` is the command-line tool used to communicate with a Kubernetes
cluster.

**Q6. What is self-healing?**\
Kubernetes controllers work to restore the desired state, for example,
by replacing a deleted Pod.

**Q7. What is scaling?**\
Scaling means increasing or decreasing the number of application
replicas.

**Q8. What is the difference between Docker and Kubernetes?**\
Docker provides tools to build and run containers. Kubernetes
orchestrates containerized workloads across a cluster.

------------------------------------------------------------------------

## ✅ Practical completion checklist

-   [ ] Kubernetes enabled and cluster status is Ready.
-   [ ] `my-node-app` image verified.
-   [ ] `deployment.yaml` created and applied.
-   [ ] Two Pods verified as Running and Ready.
-   [ ] `service.yaml` created and applied.
-   [ ] Application opened at `http://localhost:8080`.
-   [ ] Scaling to three replicas demonstrated.
-   [ ] Pod deletion and automatic replacement demonstrated.
-   [ ] Kubernetes resources cleaned up.
