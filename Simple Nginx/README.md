# Setup:
&nbsp;&nbsp;&nbsp;&nbsp;First since we are running this on our local desktop, we need a local "Control Plane" running.
1. Ensure Docker Desktop is running:
- Go into docker desktop, go to settings, enable Kubernetes
2. Check where `kubectl` thinks it is supposed to be going:
- **Run this command:** `kubectl cluster-info`
- **If it says:** `Kubernetes control plane is running at....` then the connection is good.
3. Switch your Context
- We need to tell `kubectl` to use the `docker-desktop` context
- **Run this command:** `kubectl config use-context docker-desktop`
- **Now, verify the connection:** `kubectl cluster-info`
- If you see **"Kubernetes control plane is running at https://kubernetes.docker.internal:6443"**, you are good to go.


# Step 1: Create the "Desired State" (The YAML)
&nbsp;&nbsp;&nbsp;&nbsp; First, we need to create the manifest file. Since you are comfortable with VS Code or a terminal, create a file named nginx-pod.yaml.

*YAML*  
```
apiVersion: v1   
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
    app: webserver
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
  restartPolicy: Always
```

# Step 2: Send the Intent to the API Server
&nbsp;&nbsp;&nbsp;&nbsp; Now, use the kubectl tool to hand this "wish" over to the cluster. This is the moment the API Server validates your code and writes it to etcd.

**Run this command:**

*Bash*  
`kubectl apply -f nginx-pod.yaml`  
**What to look for:** You should see `pod/my-nginx-pod created.`

# Step 3: Watch the "Hand-off"
&nbsp;&nbsp;&nbsp;&nbsp;While the Scheduler finds a node and the Kubelet pulls the image, you can watch the state change in real-time.

**Run this command:**

*Bash*  
`kubectl get pods -w`  
## The Phases you'll see:

**Pending:** The Scheduler is picking an EC2 node and the Kubelet is starting the image pull.

**ContainerCreating:** The Kubelet is talking to the container runtime (containerd/Docker).

**Running:** The app is officially alive!

# Step 4: Verify the "Actual State"
Now we look inside the Pod to see the details that the Kubelet reported back to the API Server.

**Run this command:**

*Bash*  
`kubectl describe pod my-nginx-pod`  
**Scroll to the bottom (Events): You will see the chronological history of the pod:**

**Scheduled:** The Matchmaker did its job.

**Pulling:** The Kubelet is grabbing the image.

**Started:** The process is live.

# Step 5: Test the Application
Even though the Pod is running, it's inside the cluster's private network. For a quick test, we can "tunnel" into it.

**Run this command:**

*Bash*  
`kubectl port-forward my-nginx-pod 8080:80`  
Now, open your browser and go to `localhost:8080`. You should see the "Welcome to nginx!" page.

# Step 6: Cleanup (The Deletion)
In Kubernetes, deleting the resource is just as important as creating it. This tells the API Server to update etcd, which triggers the Kubelet to kill the container.

**Run this command:**

*Bash*  
`kubectl delete -f nginx-pod.yaml`

# Shutdown

### 1. Delete the Pod
This tells the API Server to update etcd, which triggers the Kubelet to gracefully stop your container.

*Bash*  
`kubectl delete -f nginx-pod.yaml`  
### 2. Stop the Control Plane
Since Kubernetes is a resource-intensive "always-on" service, it's best to toggle it off in Docker Desktop if you aren't using it.

- Open Docker Desktop Settings > Kubernetes.

- Uncheck Enable Kubernetes and click Apply & Restart.

**Alternatively:** You can just right-click the Docker icon in your system tray and select Quit Docker Desktop.

### 3. Clear the WSL Engine (Optional but Recommended)
To make sure Windows fully reclaims that 4GB of RAM we allocated in the .wslconfig, run this in PowerShell:  

*PowerShell*  
`wsl --shutdown`  

# Troubleshooting

## Error 1: The Syntax "Gatekeeper"
The kube-apiserver validates your YAML before it ever touches etcd. If the formatting is wrong, it rejects it immediately.

### The Sabotage:
Open your nginx-pod.yaml and purposely misspell containers (e.g., change it to containerrs) or delete a colon.

### The Test:

*Bash*  
`kubectl apply -f nginx-pod.yaml`
### The Error:

`error: error parsing nginx-pod.yaml: error converting YAML to JSON: yaml: line 9: mapping values are not allowed in this context`

### The Fix:
Kubernetes is "schema-driven." This error means you sent a field that isn't in the official "dictionary." Always check your indentation and spelling against the official Kubernetes API Reference.

## Error 2: The "Image Pull" Logic
This happens when the YAML is valid, but the Kubelet can't fulfill the request (like trying to buy a product that doesn't exist).

### The Sabotage:
Change the image name to something that doesn't exist, like image: nginx:999999 or image: my-fake-app.

### The Test:

*Bash*  
`kubectl apply -f nginx-pod.yaml`  
`kubectl get pods`  
### The Error:  
The status will change to ImagePullBackOff or ErrImagePull.  
`NAME           READY   STATUS         RESTARTS   AGE`  
`my-nginx-pod   0/1     ErrImagePull   0          87m`  

### The Fix:  

- Run `kubectl describe pod my-nginx-pod.`  
- Look at the Events at the bottom.
```
Events:
  Type     Reason   Age                 From     Message
  ----     ------   ----                ----     -------
  Normal   Killing  2m8s                kubelet  spec.containers{nginx-container}: Container nginx-container definition changed, will be restarted
  Normal   Pulling  79s (x3 over 2m7s)  kubelet  spec.containers{nginx-container}: Pulling image "nginx:99999999"
  Warning  Failed   78s (x3 over 2m7s)  kubelet  spec.containers{nginx-container}: Failed to pull image "nginx:99999999": Error response from daemon: failed to resolve reference "docker.io/library/nginx:99999999": docker.io/library/nginx:99999999: not found
  Warning  Failed   78s (x3 over 2m7s)  kubelet  spec.containers{nginx-container}: Error: ErrImagePull
  Normal   BackOff  39s (x3 over 2m6s)  kubelet  spec.containers{nginx-container}: Back-off pulling image "nginx:99999999"
  Warning  Failed   39s (x3 over 2m6s)  kubelet  spec.containers{nginx-container}: Error: ImagePullBackOff
  Warning  BackOff  1s (x6 over 91s)    kubelet  spec.containers{nginx-container}: Back-off restarting failed container nginx-container in pod my-nginx-pod_default(50877003-93d1-4428-b9c9-a449c1d77e71)
```
- It will say Failed to pull image... repository does not exist.
- Check your spelling or ensure you are logged into your registry (like AWS ECR).

## Error 3: The "CrashLoopBackOff" (Runtime)
The code is right, the image is there, but the application inside the container is "exploding" as soon as it starts.

### The Sabotage:
We'll give NGINX a command that makes no sense. Add a command block that tries to run a file that doesn't exist:

*YAML*
```
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    command: ["/bin/sh", "-c", "exit 1"] # Forces the app to crash immediately
```
### The Test:

*Bash*  
`kubectl replace --force -f nginx-pod.yaml`  
`kubectl get pods`  
### The Error:
- CrashLoopBackOff. This is the most famous error in Kubernetes. It means the container started, crashed, and Kubernetes is waiting (backing off) before trying again.
```
NAME           READY   STATUS             RESTARTS     AGE
my-nginx-pod   0/1     CrashLoopBackOff   1 (5s ago)   9s
my-nginx-pod   0/1     Error              2 (16s ago)   20s
my-nginx-pod   0/1     CrashLoopBackOff   2 (16s ago)   36s
my-nginx-pod   0/1     Error              3 (30s ago)   50s
my-nginx-pod   0/1     CrashLoopBackOff   3 (12s ago)   61s
my-nginx-pod   0/1     Error              4 (54s ago)   103s
```
### The Fix:
You need to see the `"stdout"` of the container.

*Bash*  
`kubectl logs my-nginx-pod`  
- This is how you see your Python or Nginx error logs to figure out why the code itself is failing.
- You should see nothing
- You’ve just discovered the "Silent Failure." (Why the logs were empty)
- The `kubectl logs` command shows you the STDOUT (Standard Output) and STDERR (Standard Error) of the process inside the container.
- Your command was `exit 1`.
- That tells the shell to quit with an error code, but it doesn't tell it to say anything.
- Because the process didn't print "I am crashing!" to the screen before it died, the Kubelet has nothing to report back to you.

**How to make the failure "Talk"**
- If you want to see how a real application error looks, let's change the command to something that actually complains. Update the YAML to this:

*YAML*  
`command: ["/bin/sh", "-c", "echo 'Starting up...'; sleep 5; echo 'Something went wrong!'; ls /folder-that-does-not-exist"]`  
- Run the replace command again:

*Bash*  
`kubectl replace --force -f nginx-pod.yaml`  
Now check the logs:

*Bash*  
`kubectl logs my-nginx-pod`  
You should see your custom messages followed by a real error from the Linux system saying the directory doesn't exist.  
`Starting up...`  
`Something went wrong!`  
`ls: cannot access '/folder-that-does-not-exist': No such file or directory`  
