# README Content

1. Overview
2. Architecture
3. Key Design Decisions and Trade-Offs
4. Reliability and Scaling and Security Considerations
5. Other Trade-offs
6. What can be Improve With More Time
7. How to Deploy the Chart

## Overview

This simply project deploys a simple stateless HTTP service to Kubernetes. The deployment is packaged as a Helm Chart  and includes:
 - Deployment with resource limits and health probes
 - ClusterIP Service
 - Horizontal Pod Autoscaler for CPU based scaling
 - PodDisruptionBudget for availability during voluntary service disruption
 - Network Policy
 - Ingress with TLS termination

 ## Architecture

The application runs as a k8s Deployment behind a service. An ALB as Ingress allows for external access to the pods. 
Internet
   │
   ▼ HTTPS:443
+--------------------+
| Client (Browser/curl)|
+--------------------+
   │
   ▼
+--------------------+  TLS via ACM
| AWS ALB            |
+--------------------+
   │
   ▼ HTTP:80
+--------------------+  AWS LB Controller
| Kubernetes Ingress |
| (httpbin-ingress)  |
+--------------------+
   │
   ▼
+--------------------+
| ClusterIP Service  |
| (httpbin-service)  |
+--------------------+
   │   │   │
┌──┴───┴───┴──┐
▼      ▼      ▼
+-----+ +-----+ +-----+
| Pod | | Pod | | Pod |
|httpbin|httpbin|httpbin|
+-----+ +-----+ +-----+

## Key Design Decisions

Here we cover the design decisons for the different helm templates and components. 

### Deployment
Because this is a stateless application with no external dependencies, a simple deployment with replicas model is used.

### Resource sizing decisions
The service is a lightweight stateless HTTP endpoint, so the resource values assume low to moderate traffic with occasional short bursts rather than a sustained heavy load.

#### Why these values
resources:
          requests:
            memory: "128Mi" 
            cpu: "100m" 
          limits:
            memory: "256Mi"
            cpu: "250m"
A request of 100m CPU and 128Mi memory guarantees enough resources for normal request handling, while limits of 250m CPU and 256Mi memory provide headroom for brief spikes without immediate throttling or restarts. I It is assumed that traffic is expected to be intermittent, such as health checks, occasional API calls, or short bursts from the ingress layer.

### Health probes

The /get endpoint is used for both readiness and liveness probes because the assignment guarantees that it returns HTTP 200. In a real application, it would be a dedicated /healthz endpoint.

#### Readiness probe

Since the readiness probe checks whether the pod is ready to receive traffic, a short initial delay of 5 seconds allows the container to start before checks begin. The probe runs every 10 seconds, and the pod is marked not ready after 3 consecutive failures, preventing traffic from being routed to an unhealthy instance.

#### Liveness probe

Since the liveness probe ensures the container is still functioning and not stuck in a failed state, a slightly longer initial delay of 10 seconds avoids restarting the container during startup. If the endpoint fails 3 consecutive checks, Kubernetes restarts the container to recover automatically.

### HorizontalPodAutoscaler

  minReplicas: 3
  maxReplicas: 5
  aveerage CPU utilisation of 70

A HPa is configured to scale Deployment based on CPU utilization. The target utilization level is set to 70%. A minimum of 3 relicas is manitained to ensure baseline availability, and allows scaling of up to 5 replicas

Trade-off: CPU-only autoscaling is a reasonable default for an HTTP service. In production I'd also consider scaling on RPS (custom metrics via KEDA or Prometheus Adapter) — CPU can lag behind request rate. I would also include abehaiour that allows rapid scale up and slow scale doen to prevent premature reduction of capacity.

### PodDisruptionBudget

A PodDisruptionBudget is configured with minAvailable: 2 to maintain service availability during voluntary disruptions such as node maintenance or pod eviction. With three replicas running, this ensures that at least two pods remain available to continue serving traffic while one pod is affected.


### Ingress and TLS Termination

A PodDisruptionBudget is configured with minAvailable: 2 to maintain service availability during voluntary disruptions such as node maintenance or pod eviction. With three replicas running, this ensures that at least two pods remain available to continue serving traffic while one pod is safely disrupted.

An AWS ALB-backed Ingress was chosen because the intended deployment environment is Amazon EKS. Using the AWS Load Balancer Controller allows the service to be exposed through an internet-facing Application Load Balancer while integrating directly with AWS Certificate Manager (ACM) for TLS termination. This approach simplifies certificate management by keeping private keys outside the Kubernetes cluster.

```
Client →[HTTPS/TLS]→ AWS ALB →[HTTP]→ Ingress → Service → httpbin Pods
```

This configuration reflects the intended production architecture on AWS, although it cannot be fully validated in a local Kubernetes environment because local clusters cannot provision AWS load balancers or attach ACM certificates.

Alternative options I considered include using an Ingress-NGINX controller, which is more portable across Kubernetes environments but typically requires managing TLS certificates inside Kubernetes as secrets. Another emerging alternative is the Gateway API, which provides a more expressive routing model but introduces additional complexity for a small, time-boxed exercise like this one.

#### Trade-off:
Using AWS ALB aligns well with EKS and simplifies TLS management, but it introduces cloud-provider dependency compared with more portable solutions like Ingress-NGINX.

## Networking Policy

A NetworkPolicy was added to restrict inbound traffic to the application pods on TCP port 80, which is the only port required by the service. This reduces the exposed attack surface compared to the default Kubernetes networking model, which allows all pod-to-pod communication.

For simplicity, the policy allows traffic from any source on port 80. In a production environment, this would likely be tightened further by restricting access to specific namespaces or to the ingress controller only, and by defining explicit egress rules where appropriate.

## Other tradeofffs

- RBAC was not added because the application does not need to interact with the Kubernetes API to function. For this specific workload, RBAC would not materially change runtime behavior. The trade-off is that the submission does not demonstrate least-privilege API access controls through a dedicated ServiceAccount and scoped permissions.

- cert-manager was not provisioned because TLS termination was designed around AWS Certificate Manager for the intended EKS deployment model. This was validated during a demo run on EKS on an EKS cluster. This keeps certificate management outside the cluster, but it also means the solution is more AWS-specific and cannot be fully validated in a local Kubernetes environment. 


## Improvements with more time for a real prod environment
- More values would be parameterized in the Helm chart so the deployment can be reused easily across different environments.

- Namespaces would be used properly to keep application workloads separate from other cluster components.

- Observability and monitoring would be added so metrics, logs, and alerts help detect and troubleshoot issues.

- More meaningful HPA metrics would be used such as request rate or latency instead of relying only on CPU.



## How to deploy the chart

This is for deploying in a local minikube cluster.  if using an EKS cluster, generate a crtificate with AWS certificate manager and enable ingress. 


### Local Deployment

Start a local cluster:

```bash
minikube start --driver=<docker>
minikube addons enable metrics-server
```

Lint the chart:

```bash
helm lint .
```

Install the chart (Ingress disabled for local testing):

```bash
kubectl create namespace <namespace-name>
helm install <release-name> . -n <namespace-name> --set ingress.enabled=false
```

Verify the deployment:

```bash
helm list -n <namespace-name>
kubectl get pods -n <namespace-name>
kubectl get svc -n <namespace-name>
```

Test the service:

```bash
kubectl port-forward svc/<service-name> 8080:80 -n <namespace-name>
curl http://localhost:8080/get
```

#### Test autoscaling (HPA)

Generate load using a temporary BusyBox pod:

```bash
kubectl run loadgen -n <namespace-name> --rm -it --image=busybox -- /bin/sh
```

Inside the BusyBox shell:

```bash
while true; do wget -q -O- http://<service-name>/get > /dev/null; done
```

In another terminal, watch the HPA:

```bash
kubectl get hpa -n <namespace-name> -w
```

#### Release management

```bash
helm upgrade <release-name> . -n <namespace-name>
helm history <release-name> -n <namespace-name>
helm rollback <release-name> 1 -n <namespace-name>
```

#### Cleanup

Delete the Helm release and namespace:

```bash
helm uninstall <release-name> -n <namespace-name>
kubectl delete namespace <namespace-name>
```



