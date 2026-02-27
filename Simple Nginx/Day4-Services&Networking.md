# Day 4 — Services & Networking Basics
***Tasks:***  
-Learn Service types: ClusterIP, NodePort, LoadBalancer  
-Understand: kube-proxy role and basic CNI concept  
***Lab:***  
-Expose your NGINX Deployment via ClusterIP → NodePort  
-Test connectivity using curl inside cluster (kubectl exec)  
-Try to break the service and debug  
***Output:***  
-Diagram showing traffic flow  
-Written notes explaining why traffic might fail  

# Tasks
## Learn Service Types: ClusterIP, NodePort, LoadBalancer

**1. ClusterIp(Default):** 
- The service is only reachable from within the cluster.
- Used for internal communication between your apps.

**2. NodePort:**
- opens a static port on every Node's IP
- The service allows external traffic to reach the service via `<NodeIP>:<NodePort>`.

**3. LoadBalancer:**
- when used in AWS, Kubernetes talks to the AWS API to spin up an external Load Balancer (like an NLB or ALB) that automatically routes traffic to your NodePorts.
- the production standard

## Understand Kube-Proxy  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *How does traffic get from the Service IP to the Pod IP?*  
- **kube-proxy** is a small process that runs on every single node and maintains the network rules on the host.
- When you create a Service, `kube-proxy` sees the service and creates 'IP Tables' or 'IPVS' rules on the node's OS.
- When traffic hits the Service IP, the Linux kernel (managed by kube-proxy) intercepts it, realizes it is a Kubernetes service, and redirects the packet to one of the Pod IPs.

## Understand CNI (Container Network Interface)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*Kubernetes does not actually know how to create a network, the CNI plugin does that.*
- Every time a pod is created, the CNI is responible for assigning it an IP address and making sure it can talk to pods on other nodes. 
- It ensures that every pod has a unique IP and can communicate with every other pod without using NAT
- On Docker Desktop the CNI is built-in. In AWS you use the AWS VPC CNI, which gives the pods an IP address from your VPC's subnet.


# Lab:
&nbsp;&nbsp;&nbsp;&nbsp;In Kubernetes, Pods are **ephemeral**—they live and die, and their IP addresses change constantly. A **Service** is a stable "Front Door" that stays the same even if the pods behind it are replaced.

## Step 1: Create the ClusterIP Service
A **ClusterIP** is the default service type. It gives your Nginx pods a single, internal IP address that **only other pods in the cluster can talk to**.

Create a file named nginx-service.yaml:

*YAML*  
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-internal-service
spec:
  type: ClusterIP
  selector:
    app: nginx # This must match the labels on your Deployment pods!
  ports:
    - protocol: TCP
      port: 80 # The port the SERVICE listens on
      targetPort: 80 # The port the POD listens on
```

Apply it: `kubectl apply -f nginx-service.yaml`

Find the ClusterIP: `kubectl get svc`

```
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes               ClusterIP   10.96.0.1       <none>        443/TCP   24h
nginx-internal-service   ClusterIP   10.107.135.76   <none>        80/TCP    9m35s
```

## Step 2: Test Connectivity (The "Cloud Engineer" Way)
&nbsp;&nbsp;&nbsp;&nbsp; Since a ClusterIP isn't reachable from your Windows browser, we need to "remote" into a pod to test it. This is exactly how you'd troubleshoot a networking issue in an AWS EKS environment.

Get your pod names: `kubectl get pods`

Exec into a pod: `kubectl exec -it <name-of-one-of-your-pods> -- //bin/sh`  

Inside the pod, try to reach the service: `curl nginx-internal-service`  
*Note: Kubernetes has built-in DNS, so you don't even need the IP address!*

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
working. Further configuration is required.</p>
working. Further configuration is required.</p>
working. Further configuration is required.</p>
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Step 3: Upgrade to NodePort
&nbsp;&nbsp;&nbsp;&nbsp;Now, let's make it reachable from your laptop. A **NodePort** opens a specific port on your "Node" (the Docker Desktop VM) and forwards that traffic to the Service.

Edit `nginx-service.yaml` and change `type: ClusterIP` to `type: NodePort`.

Apply the change: `kubectl apply -f nginx-service.yaml`

Find the port: `kubectl get svc`  
*Look under PORT(S). You will see 80:XXXXX/TCP. That 5-digit number (usually 30000-32767) is your NodePort.*

Test it: Open your browser to `http://localhost:<YOUR-NODEPORT>`.

## Step 4: The "Sabotage" (Debug Time)
&nbsp;&nbsp;&nbsp;&nbsp;Let's break the link between the Service and the Pods. This is the #1 networking issue in Kubernetes.

### The Sabotage:
Edit `nginx-service.yaml` and change the `selector` from `app: nginx` to `app: wrong-label`.

### The Test:

`kubectl apply -f nginx-service.yaml`

Try to refresh your browser or curl the service from inside the pod. It will hang or fail.

### The Debugging Tool:
Run this command to see if the Service actually "sees" your pods:

*Bash*  
`kubectl get endpoints nginx-internal-service`  
If it says `<none>:` Your selector is wrong. The Service is a front door with no hallway behind it.

```
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                     ENDPOINTS   AGE
nginx-internal-service   <none>      34m
```

If it lists IPs: The connection is healthy.

# Outputs

## Traffic Flow

### Path of the Packet

**Web Browser --> NodePort --> Kube-proxy --> Service --> Pod/Container**  
*When we typed in `localhost:31234` the traffic went through this path:*  
1. **Origin:** Your Web Browser (Windows).
2. **The Entry (NodePort):** Hits the Docker Desktop VM on port `31234`.
3. **The Routing (Kube-proxy):** Kube-proxy sees the traffic on `31234` and uses its internal rules to map it to the Service (ClusterIP).
4. **The Load Balancer (Service):** The Service checks its Endpoints list (the Pod IPs) and picks one to send the traffic to.
5. **The Destination (Pod/Container):** The traffic arrives at the Pod on `targetPort: 80.`

## Why Traffic Might Fail (Troubleshooting Notes)

&nbsp;&nbsp;&nbsp;&nbsp; *You will use these three areas to diagnose `Connection Refused` or `Timeout` errors:*

### 1. Label Mismatch (Service-to-Pod Break)
**The Symptom:** The Service exists, but `curl` or browser requests just hang.

**The Cause:** The Service `selector` (e.g., `app: nginx`) does not exactly match the `labels` on your Pods.

**The Check:** Run `kubectl get endpoints`. If it’s `<none>`, the "bridge" is broken.

### 2. TargetPort Mismatch (Service-to-Container Break)
**The Symptom:** Connection is refused.

**The Cause:** The Service is sending traffic to port 80, but your NGINX container is actually listening on a different port (like 8080), or the application inside the container crashed.

**The Check:** Run `kubectl describe pod` to verify the container's listening port.

### 3. Firewall/Network Policy (External-to-Node Break)
**The Symptom:** `"Site cannot be reached"` immediately.

**The Cause:** On a local machine, this is usually because Docker Desktop isn't running or the port is blocked by Windows Firewall. In AWS, this is usually a Security Group rule missing on your EC2 instances.

**The Check:** Verify the service is type `NodePort` or `LoadBalancer` and that the port is open in your cloud provider's firewall.

### 4. Readiness/Liveness Probe Failures
**The Symptom:** Pods are "Running," but the Service isn't sending traffic to them.

**The Cause:** If a pod fails its "Health Check," Kubernetes removes it from the Service's Endpoints list so that users don't get errors.

**The Check:** Run `kubectl get pods` and check the `READY` column (e.g., 0/1 means it’s running but not healthy).