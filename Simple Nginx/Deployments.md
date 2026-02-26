# The "Deployment" Upgrade
Now, we have seen how painful it is to manually `--force` delete a Pod every time you want to change a simple command. In a professional environment, we use Deployments.

A **Deployment** is a higher-level object that manages Pods for you. It's like a "Manager" who is told, "Ensure 3 NGINX pods are running." If you change the code, the Manager automatically handles the replacement.

First we create the deployment: *nginx-deployment.yaml*

*YAML*
`apiVersion: apps/v1`
`kind: Deployment`
`metadata:`
`  name: nginx-deployment`
`spec:`
`  replicas: 3 # The "Desired State"`
`  selector:`
`    matchLabels:`
`      app: nginx`
`  template:`
`    metadata:`
`      labels:`
`        app: nginx`
`    spec:`
`      containers:`
`      - name: nginx-container`
`        image: nginx:latest`
`        ports:`
`        - containerPort: 80`

**Why this is better:**
**No more --force:** If you change the image or command in a Deployment, you just run `kubectl apply`. The Deployment Controller (inside the Controller Manager) sees the change and performs a Rolling Update.

**Self-Healing:** If you manually delete one of the 3 pods, the Controller Manager will notice the "Actual State" (2) doesn't match the "Desired State" (3) and will immediately spin up a new one.

## Steps
- First, let's clean up our broken pod: `kubectl delete pod my-nginx-pod`
- Save the code above as nginx-deployment.yaml.
- Run `kubectl apply -f nginx-deployment.yaml.`
- Once you do that, run `kubectl get pods`

## Test the "Self-Healing"
This is the moment where you see the Controller Manager earn its paycheck. Because you told Kubernetes the Desired State is replicas: 3, it will fight to keep it that way.

### The Sabotage:
Pick one of your pod names (run `kubectl get pods` to see them) and manually delete it.

### The Command:

*Bash*
`kubectl delete pod <your-pod-name>`
### The Observation:

Immediately after running the delete, run `kubectl get pods`.

### What you will see:
You'll see one pod in the Terminating state, but you will also see a brand new pod already in Pending or ContainerCreating. The ReplicaSet Controller saw the count drop to 2 and didn't even wait for you to ask—it just fixed the "drift" automatically.

`nginx-deployment-6586c5b5fb-nr9h7   1/1     Running   0          3m7s`
`nginx-deployment-6586c5b5fb-rwqmz   1/1     Running   0          74s`
`nginx-deployment-6586c5b5fb-xbnp5   1/1     Running   0          3m7s`
`nginx-deployment-6586c5b5fb-nr9h7   1/1     Terminating   0          3m50s`
`nginx-deployment-6586c5b5fb-zpf55   0/1     Pending       0          0s`
`nginx-deployment-6586c5b5fb-nr9h7   1/1     Terminating   0          3m50s`
`nginx-deployment-6586c5b5fb-zpf55   0/1     Pending       0          0s`
`nginx-deployment-6586c5b5fb-zpf55   0/1     ContainerCreating   0          0s`
`nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m51s`
`nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s`
`nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s`
`nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s`
`nginx-deployment-6586c5b5fb-zpf55   1/1     Running             0          4s`

The Rolling Update: Deploying "Code"
Now that you know how the cluster heals itself, let's see how it handles a change to your application. This is exactly what happens when your GitLab CI/CD pipeline pushes a new version.

We’re going to change the NGINX version from latest to 1.25.1. In a Deployment, this triggers a Rolling Update. Kubernetes won't just kill all 3 pods at once (which would cause downtime); it will replace them one by one.

The Command:
You can do this by editing your YAML and running apply, or you can do it directly with this command:

Bash
kubectl set image deployment/nginx-deployment nginx-container=nginx:1.25.1 --record
What to watch for in your -w terminal:

A new pod starts up (Pending -> Running).

Once the new one is healthy, an old one starts Terminating.

Another new one starts up... and so on.

nginx-deployment-6586c5b5fb-nr9h7   1/1     Running   0          3m7s
nginx-deployment-6586c5b5fb-rwqmz   1/1     Running   0          74s
nginx-deployment-6586c5b5fb-xbnp5   1/1     Running   0          3m7s
nginx-deployment-6586c5b5fb-nr9h7   1/1     Terminating   0          3m50s
nginx-deployment-6586c5b5fb-zpf55   0/1     Pending       0          0s
nginx-deployment-6586c5b5fb-nr9h7   1/1     Terminating   0          3m50s
nginx-deployment-6586c5b5fb-zpf55   0/1     Pending       0          0s
nginx-deployment-6586c5b5fb-zpf55   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m51s
nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s
nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s
nginx-deployment-6586c5b5fb-nr9h7   0/1     Completed           0          3m53s
nginx-deployment-6586c5b5fb-zpf55   1/1     Running             0          4s
nginx-deployment-5d9ccb8656-flvxz   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-flvxz   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-flvxz   0/1     ContainerCreating   0          0s
nginx-deployment-5d9ccb8656-flvxz   1/1     Running             0          13s
nginx-deployment-6586c5b5fb-xbnp5   1/1     Terminating         0          15m
nginx-deployment-6586c5b5fb-xbnp5   1/1     Terminating         0          15m
nginx-deployment-5d9ccb8656-zgtg6   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-zgtg6   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-zgtg6   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-xbnp5   0/1     Completed           0          15m
nginx-deployment-6586c5b5fb-xbnp5   0/1     Completed           0          15m
nginx-deployment-6586c5b5fb-xbnp5   0/1     Completed           0          15m
nginx-deployment-5d9ccb8656-zgtg6   1/1     Running             0          2s
nginx-deployment-6586c5b5fb-rwqmz   1/1     Terminating         0          13m
nginx-deployment-5d9ccb8656-grlc8   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-rwqmz   1/1     Terminating         0          13m
nginx-deployment-5d9ccb8656-grlc8   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-grlc8   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-rwqmz   0/1     Completed           0          13m
nginx-deployment-6586c5b5fb-rwqmz   0/1     Completed           0          13m
nginx-deployment-6586c5b5fb-rwqmz   0/1     Completed           0          13m
nginx-deployment-5d9ccb8656-grlc8   1/1     Running             0          3s
nginx-deployment-6586c5b5fb-zpf55   1/1     Terminating         0          11m
nginx-deployment-6586c5b5fb-zpf55   1/1     Terminating         0          11m
nginx-deployment-6586c5b5fb-zpf55   0/1     Completed           0          11m
nginx-deployment-6586c5b5fb-zpf55   0/1     Completed           0          11m
nginx-deployment-6586c5b5fb-zpf55   0/1     Completed           0          11m


Why this is a "Cloud Engineer" Power Move
If you were doing this with Terraform on raw EC2 instances, you’d have to manage the Load Balancer, wait for connection draining, and manually swap instances. Kubernetes does all of that "Traffic Management" for you.

High Availability: At no point during that update was your website "down."

The Safety Net: If the new version was broken (e.g., if you used a fake image name), Kubernetes would stop the rollout automatically, leaving your old, working pods alone.

In a real-world scenario—like if you were deploying a new Python update to your São Paulo region—and suddenly the logs started showing errors, you wouldn't want to spend 20 minutes rewriting YAML. You want to go back to the last known "Good State" immediately.

The Magic of Rollouts
Because you used a Deployment, Kubernetes keeps a history of your changes (Revisions).

1. Check the History:
Run this to see your past "Apply" actions:

Bash
kubectl rollout history deployment/nginx-deployment
2. The Emergency Rollback:
If you realized that version 1.25.1 was a mistake, you can undo it with one command:

Bash
kubectl rollout undo deployment/nginx-deployment
What to watch for:
If you still have your kubectl get pods -w terminal open, you'll see the dance happen again, but in reverse! The "new" pods will be terminated, and the "old" version will be brought back to life.

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5d9ccb8656-flvxz   1/1     Running   0          4m22s
nginx-deployment-5d9ccb8656-grlc8   1/1     Running   0          4m7s
nginx-deployment-5d9ccb8656-zgtg6   1/1     Running   0          4m9s
nginx-deployment-6586c5b5fb-vxtzs   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-vxtzs   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-vxtzs   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-vxtzs   1/1     Running             0          2s
nginx-deployment-5d9ccb8656-zgtg6   1/1     Terminating         0          4m29s
nginx-deployment-6586c5b5fb-699v7   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-zgtg6   1/1     Terminating         0          4m29s
nginx-deployment-6586c5b5fb-699v7   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-699v7   0/1     ContainerCreating   0          0s
nginx-deployment-5d9ccb8656-zgtg6   0/1     Completed           0          4m30s
nginx-deployment-5d9ccb8656-zgtg6   0/1     Completed           0          4m30s
nginx-deployment-5d9ccb8656-zgtg6   0/1     Completed           0          4m30s
nginx-deployment-6586c5b5fb-699v7   1/1     Running             0          2s
nginx-deployment-5d9ccb8656-flvxz   1/1     Terminating         0          4m44s
nginx-deployment-6586c5b5fb-bh9x7   0/1     Pending             0          0s
nginx-deployment-5d9ccb8656-flvxz   1/1     Terminating         0          4m44s
nginx-deployment-6586c5b5fb-bh9x7   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-bh9x7   0/1     ContainerCreating   0          0s
nginx-deployment-5d9ccb8656-flvxz   0/1     Completed           0          4m45s
nginx-deployment-5d9ccb8656-flvxz   0/1     Completed           0          4m46s
nginx-deployment-5d9ccb8656-flvxz   0/1     Completed           0          4m46s
nginx-deployment-6586c5b5fb-bh9x7   1/1     Running             0          3s
nginx-deployment-5d9ccb8656-grlc8   1/1     Terminating         0          4m32s
nginx-deployment-5d9ccb8656-grlc8   1/1     Terminating         0          4m32s
nginx-deployment-5d9ccb8656-grlc8   0/1     Completed           0          4m32s
nginx-deployment-5d9ccb8656-grlc8   0/1     Completed           0          4m33s
nginx-deployment-5d9ccb8656-grlc8   0/1     Completed           0          4m33s


Scaling: The "Traffic Jam" Scenario
Before we wrap up this lab, let's look at one more superpower. Imagine your Nginx web server is getting slammed with traffic because of a marketing campaign in your Tokyo region. You need more power, and you need it now.

The Scaling Command:

Bash
kubectl scale deployment/nginx-deployment --replicas=10
Watch your terminal: You will see 7 new pods instantly spring into existence. The Scheduler and Kubelet work together to find space for them across your nodes.

nginx-deployment-6586c5b5fb-699v7   1/1     Running   0          16m
nginx-deployment-6586c5b5fb-bh9x7   1/1     Running   0          16m
nginx-deployment-6586c5b5fb-vxtzs   1/1     Running   0          16m
nginx-deployment-6586c5b5fb-nhc22   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-tps5z   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-6tsvw   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-nhc22   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-526qz   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-9t6lb   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-dxhvp   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-kxvzv   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-tps5z   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-6tsvw   0/1     Pending   0          0s
nginx-deployment-6586c5b5fb-nhc22   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-dxhvp   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-526qz   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-9t6lb   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-kxvzv   0/1     Pending             0          0s
nginx-deployment-6586c5b5fb-tps5z   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-6tsvw   0/1     ContainerCreating   0          0s
nginx-deployment-6586c5b5fb-dxhvp   0/1     ContainerCreating   0          1s
nginx-deployment-6586c5b5fb-526qz   0/1     ContainerCreating   0          1s
nginx-deployment-6586c5b5fb-9t6lb   0/1     ContainerCreating   0          1s
nginx-deployment-6586c5b5fb-kxvzv   0/1     ContainerCreating   0          1s
nginx-deployment-6586c5b5fb-nhc22   1/1     Running             0          4s
nginx-deployment-6586c5b5fb-526qz   1/1     Running             0          5s
nginx-deployment-6586c5b5fb-dxhvp   1/1     Running             0          6s
nginx-deployment-6586c5b5fb-tps5z   1/1     Running             0          6s
nginx-deployment-6586c5b5fb-9t6lb   1/1     Running             0          7s
nginx-deployment-6586c5b5fb-6tsvw   1/1     Running             0          9s
nginx-deployment-6586c5b5fb-kxvzv   1/1     Running             0          9s
nginx-deployment-6586c5b5fb-526qz   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-nhc22   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-699v7   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-tps5z   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-dxhvp   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-bh9x7   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-vxtzs   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-kxvzv   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-9t6lb   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-6tsvw   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-526qz   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-699v7   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-nhc22   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-6tsvw   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-dxhvp   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-bh9x7   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-vxtzs   1/1     Terminating         0          18m
nginx-deployment-6586c5b5fb-9t6lb   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-kxvzv   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-tps5z   1/1     Terminating         0          108s
nginx-deployment-6586c5b5fb-526qz   0/1     Completed           0          109s
nginx-deployment-6586c5b5fb-kxvzv   0/1     Completed           0          110s
nginx-deployment-6586c5b5fb-6tsvw   0/1     Completed           0          110s
nginx-deployment-6586c5b5fb-9t6lb   0/1     Completed           0          110s
nginx-deployment-6586c5b5fb-nhc22   0/1     Completed           0          110s
nginx-deployment-6586c5b5fb-tps5z   0/1     Completed           0          111s
nginx-deployment-6586c5b5fb-526qz   0/1     Completed           0          111s
nginx-deployment-6586c5b5fb-526qz   0/1     Completed           0          111s
nginx-deployment-6586c5b5fb-699v7   0/1     Completed           0          18m
nginx-deployment-6586c5b5fb-vxtzs   0/1     Completed           0          18m
nginx-deployment-6586c5b5fb-bh9x7   0/1     Completed           0          18m
