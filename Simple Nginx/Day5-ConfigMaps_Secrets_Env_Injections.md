# Day 5 — ConfigMaps, Secrets, and Environment Injection
***Tasks:***  
- Learn how to pass configuration and secrets to Pods  
- Understand difference between ConfigMaps and Secrets  
***Lab:***  
- Create a ConfigMap & Secret  
- Mount both into Pods as env variables and files  
- Update ConfigMap → observe Pod behavior  
***Output:***  
- Write-up: “How config & secrets are injected and updated”  
- Terminal logs  

# Tasks

## Learn how to pass configuration and secrets to Pods

### 1. The Strategy: Two Ways to Pass Data
*There are two primary ways we passed these configurations to your Nginx pods:*  

- **Environment Variables:** This is the most common method. Kubernetes takes a value from a ConfigMap/Secret and "injects" it as a standard Linux environment variable (like $PATH or $USER). Your application reads these at startup.

- **Volume Mounts:** Kubernetes takes the entire ConfigMap/Secret and turns it into a virtual folder. Each "Key" in your data becomes a "File" in that folder. This is how you pass large configuration files (like an nginx.conf or a prometheus.yml).

### 2. Why we separate them (Config vs. Secret)
*Even though the "passing" mechanism is the same, we used two different objects for a reason:*  

**ConfigMaps (The "Open" Data):** We used this for `APP_COLOR`. This is data that isn't dangerous if someone sees it. It’s stored in plain text, making it easy for DevOps teams to audit.

**Secrets (The "Locked" Data):** We used this for `ACCESS_TOKEN`. Kubernetes encodes this in Base64. While not "encrypted" in the strongest sense yet, it prevents accidental "shoulder surfing" or someone accidentally seeing a password in a log file or a kubectl get command.

### 3. The Workflow we followed:
**Step 1:** We created the "Identity" objects (`app-config.yaml` and `app-secret.yaml`).

**Step 2:** we updated the "Workload" (`nginx-deployment.yaml`) to "ask" for those objects.

**Step 3:** The **Kubelet** (the worker on your laptop) pulled those values from the API Server and prepared the container's environment before the Nginx process even started.

## What is the difference between `valueFrom` (Environment) vs. `volumeMounts` (Files)?

### 1. `valueFrom` (Environment Variables)
- When you use `valueFrom`, you are telling the Kubelet to inject data directly into the container's process environment.

**The Mechanism:** Kubernetes looks up the value in the ConfigMap/Secret before the container starts. It then passes that value as a string to the container runtime (Docker/containerd).

**Storage:** The data exists only in the Process Memory of the container. If you run env inside the pod, you see it.

**The "Static" Nature:** Once a Linux process starts, its environment variables are effectively "baked in." Even if you change the ConfigMap in the background, the running process has no way to "see" that the string in its memory should change.

**Best Use Case:** Simple flags, database hostnames, or API endpoints where the app only needs to know the value once at startup.

### 2. `volumeMounts` (File System Projection)
- When you use `volumeMounts`, you are treating the ConfigMap like a Virtual Hard Drive.

**The Mechanism:** Kubernetes creates a specialized directory on the Host Node. It then "mounts" (maps) that directory into the container's file system at your specified mountPath (like /etc/config).

**The "Symlink" Magic:** Kubernetes doesn't just copy the file. It creates a series of Symbolic Links (aliases). One link points to a timestamped folder containing your data.

**The "Dynamic" Nature:** When you update the ConfigMap, the Kubelet eventually notices. It creates a new timestamped folder with the new data and updates the symlink to point to the new folder. Since the application is looking at the symlink, it suddenly sees the new content.

**Best Use Case:** Large configuration files (like nginx.conf), certificates, or apps designed to "hot-reload" when a file change is detected.

## Understand difference between ConfigMaps and Secrets

*The next major piece is understanding why we bother having two different objects (ConfigMaps and Secrets) when the injection methods we just discussed work exactly the same for both.*

- The "Visible" vs. the "Veiled"  
In a Cloud Engineer's world, this is all about **Security Posture** and **Access Control**.  

### 1. ConfigMaps: The "Clear Text" Settings
- ConfigMaps are designed for Environment-Specific configuration.

**Storage:** Data is stored in plain text.

**Visibility:** If you `run kubectl get cm app-settings -o yaml`, you see the values exactly as you typed them.

**Use Case:** You use these for the "knobs and dials" of your app. Things like `LOG_LEVEL: "DEBUG"` or `DB_PORT: "5432"`.

**The Goal:** Transparency. You want your DevOps team to be able to see these at a glance to understand how the cluster is behaving.

### 2. Secrets: The "Obfuscated" Data
- Secrets are designed for Sensitive Information.

**Storage:** Data is **Base64 encoded**. When you run `kubectl get secret app-credentials -o yaml`, you won't see `super-secret-12345`. Instead, you'll see a string like `c3VwZXItc2VjcmV0LTEyMzQ1`.

**Visibility:** It requires an extra step to decode. This prevents "accidental" exposure (e.g., someone glancing at your screen or a password showing up in a plain-text log file).

**The "AWS Level" Secret**: In your AWS configurations, we’ll talk about how EKS (Amazon's Kubernetes) can encrypt these at rest using AWS KMS. This means even if someone hacked the underlying database (etcd), they still couldn't read your passwords.

### The Base64 "Misconception"  
**Base64 is NOT encryption.** It is a way of formatting data. Anyone with access to the `kubectl` command and the right permissions can decode a secret in one second using:  
`echo "c3VwZXItc2VjcmV0LTEyMzQ1" | base64 --decode`  

**Why bother then?** It acts as a "safety railing." It tells the system (and other engineers): *"This is a sensitive value; don't print this in logs, don't show it in basic UI views, and handle it with care."*

**ConfigMaps are for "The Way the App Runs" (Behavior).**

**Secrets are for "What the App Needs to Access" (Identity/Permission).**

# Lab

## Create a ConfigMap & Secret

- To decouple the application code from its environment-specific settings, I created two Kubernetes objects: a **ConfigMap** for non-sensitive UI settings and a **Secret** for sensitive API credentials.

### 1. ConfigMap (app-config.yaml)
- Used to store non-confidential configuration data in key-value pairs.

*YAML*  
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  APP_COLOR: "green"
  UI_MESSAGE: "Welcome to the Lab Cluster"
```

### 2. Secret (app-secret.yaml)
- Used to store sensitive information. While the YAML uses stringData for easy authoring, Kubernetes automatically converts these to Base64 strings upon creation.

*YAML*  
```
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
type: Opaque
stringData:
  API_KEY: "super-secret-12345"
```

### 3. Deployment Commands
- To instantiate these objects in the cluster, I used the following commands:

*Bash*  
```
kubectl apply -f app-config.yaml
kubectl apply -f app-secret.yaml
```

## Mount both into Pods as env variables and files

- To make the application "aware" of its configuration, I implemented two distinct injection methods within the nginx-deployment.yaml.

### 1. Environment Variable Injection (`valueFrom`)
- I mapped specific keys from the ConfigMap and Secret directly into the container's process environment. This allows the application to read settings as standard Linux variables.  

*YAML*  
```
env:
- name: THEME_COLOR
  valueFrom:
    configMapKeyRef:
      name: app-settings
      key: APP_COLOR
- name: ACCESS_TOKEN
  valueFrom:
    secretKeyRef:
      name: app-credentials
      key: API_KEY
```

### 2. File Mount Injection (volumeMounts)
- I "projected" the entire app-settings ConfigMap as a virtual directory. This creates a file for every key in the ConfigMap data.

*YAML* 
```
# Define the Volume at the Pod level
volumes:
  - name: config-volume
    configMap:
      name: app-settings

# Mount the Volume inside the container
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

- To confirm the injection was successful, I executed the following commands inside the running pod:

**Check Environment Variables:**

*Bash*  
`kubectl exec -it <pod-name> -- env | grep -E "THEME_COLOR|ACCESS_TOKEN"`
**Check Mounted Files:**  

*Bash*  
`kubectl exec -it <pod-name> -- ls /etc/config`  
`kubectl exec -it <pod-name> -- cat /etc/config/APP_COLOR`  

## Update ConfigMap → observe Pod behavior

- The core of this lab was observing how Kubernetes handles updates to configuration data. I updated the `app-settings` ConfigMap from `green` to `yellow` and monitored the two injection points.

### The Experiment
**Modify:** Changed `APP_COLOR: "yellow"` in `app-config.yaml`.

**Apply:** Ran `kubectl apply -f app-config.yaml`.

**Monitor:** Checked both the Environment Variable and the File Mount.

### 1. Environment Variables (Static)
**Command:**

*Bash*  
`kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "env | grep THEME_COLOR"`  

**Observation:** The output remained `THEME_COLOR=green.`
```
kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "env | grep THEME_COLOR"
THEME_COLOR=green
```

**Reasoning:** Environment variables are injected into the container's process at startup. They are immutable for the life of the process. To update these, a pod restart is required.

### 2. File Mounts (Dynamic)
**Command:**  

*Bash*  
`kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "cat /etc/config/APP_COLOR"`  

**Observation:** After approximately 60 seconds, the output changed to yellow.
```
kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "cat /etc/config/APP_COLOR"
green
kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "cat /etc/config/APP_COLOR"
green
kubectl exec -it nginx-deployment-7796c868c5-9x884 -- //bin/sh -c "cat /etc/config/APP_COLOR"
yellow
```
**Reasoning:** Kubernetes uses symbolic links to manage ConfigMap volumes. When the ConfigMap is updated, the Kubelet updates the underlying files on the node, and the container sees the new data through the mount point without a restart.

### Cloud Engineer Lesson: 
 If your application is designed to "watch" for file changes (like Prometheus or Nginx), use Volume Mounts for zero-downtime config updates. If your app only reads config at startup (like many Python/Go apps), use Environment Variables, but be prepared to perform a `kubectl rollout restart`