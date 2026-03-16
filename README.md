# Kubernetes Deployment for a HTTP Service

---

# README Content

1. Overview  
2. Architecture  
3. Key Design Decisions and Trade-Offs  
4. Other Trade-offs 
5. Reliability, scalability and Security Considerations 
6. What Can Be Improved With More Time  
7. How to Deploy the Chart  

---

# 1. Overview

This simply project deploys a simple stateless HTTP service to Kubernetes. The deployment is packaged as a Helm Chart and includes:

- Deployment with resource limits and health probes  
- ClusterIP Service  
- Horizontal Pod Autoscaler for CPU based scaling  
- PodDisruptionBudget for availability during voluntary service disruption  
- Network Policy  
- Ingress with TLS termination  

---

# 2. Architecture

The application runs as a k8s Deployment behind a service. An ALB as Ingress allows for external access to the pods.

```
Internet
   │
   ▼ HTTPS:443
+----------------------+
| Client (Browser/curl)|
+----------------------+
          │
          ▼
+----------------------+
| AWS ALB              |
| TLS via ACM          |
+----------------------+
          │ HTTP:80
          ▼
+----------------------+
| Kubernetes Ingress   |
| (AWS LB Controller)  |
+----------------------+
          │
          ▼
+----------------------+
| ClusterIP Service    |
| (httpbin-service)    |
+----------------------+
      │       │       │
      ▼       ▼       ▼
   +------+ +------+ +------+
   | Pod  | | Pod  | | Pod  |
   |httpbin| |httpbin| |httpbin|
   +------+ +------+ +------+
```

---

# 3. Key Design Decisions and Trade-Offs

Here we cover the design decisons for the different helm templates and components.

---

## 3.1 Deployment

Because this is a stateless application with no external dependencies, a simple deployment with 3 replicas model is used.

---

## 3.2 Resource Sizing Decisions

The service is a lightweight stateless HTTP endpoint, so the resource values assume low to moderate traffic with occasional short spikes rather than a sustained heavy load.

### Why these values

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "250m"
```

A request of 100m CPU and 128Mi memory guarantees enough resources for normal request handling, while limits of 250m CPU and 256Mi memory provide headroom for brief spikes without immediate throttling or restarts. It is assumed that traffic is expected to be intermittent, such as health checks, occasional API calls, or short bursts from the ingress layer.

---

## 3.3 Health Probes

The /get endpoint is used for both readiness and liveness probes because the assignment guarantees that it returns HTTP 200. In a real application, it would be a dedicated health endpoint.

### Readiness Probe

```yaml
        readinessProbe:
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
```

Since the readiness probe checks whether the pod is ready to receive traffic, a short initial delay of 5 seconds allows the container to start before checks begin. The probe runs every 10 seconds, and the pod is marked not ready after 3 consecutive failures, preventing traffic from being routed to an unhealthy instance.

### Liveness Probe

```yaml
        livenessProbe:
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
```
Since the liveness probe ensures the container is still functioning and not stuck in a failed state, a slightly longer initial delay of 10 seconds avoids restarting the container during startup. If the endpoint fails 3 consecutive checks, Kubernetes restarts the container to recover automatically. With multiple restarts, a troubleshooting will have to be performed to fix the issue

---

## 3.4 HorizontalPodAutoscaler

```yaml
minReplicas: 3
maxReplicas: 5
average CPU utilisation: 70%
```

A HPA is configured to scale Deployment based on CPU utilization. The target utilization level is set to 70%. A minimum of 3 replicas is maintained to ensure baseline availability, and allows scaling of up to 5 replicas.

Trade-off: CPU-only autoscaling is a reasonable default for an HTTP service. In production I'd also consider scaling on RPS (custom metrics via KEDA or Prometheus Adapter). CPU can lag behind request rate. I would also include behaviour that allows rapid scale up and slow scale down to prevent premature reduction of capacity.

---

## 3.5 PodDisruptionBudget

A PodDisruptionBudget is configured with `minAvailable: 2` to maintain service availability during voluntary disruptions such as node maintenance or pod eviction.

With three replicas running, this ensures that at least two pods remain available to continue serving traffic while one pod is affected.

---

## 3.6 Ingress and TLS Termination

An AWS ALB-backed Ingress was chosen because the intended deployment environment is Amazon EKS. Using the AWS Load Balancer Controller allows the service to be exposed through an internet-facing Application Load Balancer while integrating directly with AWS Certificate Manager (ACM) for TLS termination.

This approach simplifies certificate management by keeping private keys outside the Kubernetes cluster.

```
Client → [HTTPS/TLS] → AWS ALB → [HTTP] → Ingress → Service → httpbin Pods
```

This configuration reflects the intended production architecture on AWS, although it cannot be fully validated in a local Kubernetes environment because local clusters cannot provision AWS load balancers or attach ACM certificates.

Alternative options I considered include:

- Ingress-NGINX controller (more portable but requires TLS secrets)
- Gateway API (more expressive routing model but more complex)

Trade-off:  
Using AWS ALB aligns well with EKS and simplifies TLS management, but introduces cloud-provider dependency compared with more portable solutions like Ingress-NGINX.

---

## 3.7 Network Policy

A basic NetworkPolicy was added to restrict inbound traffic to the application pods on TCP port 80, which is the only port required by the service.

This reduces the exposed attack surface compared to the default Kubernetes networking model, which allows all pod-to-pod communication.

For simplicity, the policy allows traffic from any source on port 80. In a production environment, this would likely be tightened further by restricting access to specific namespaces or to the ingress controller only, and by defining explicit egress rules where appropriate.

---

# 4. Other Trade-offs

- RBAC was not added because the application does not need to interact with the Kubernetes API to function. For this specific workload, RBAC would not materially change runtime behavior. The trade-off is that the submission does not demonstrate least-privilege API access controls through a dedicated ServiceAccount and scoped permissions.

- cert-manager was not provisioned because TLS termination was designed around AWS Certificate Manager for the intended EKS deployment model. This keeps certificate management outside the cluster, but also makes the solution more AWS-specific and difficult to fully validate locally.

---

# 5. Reliability, Scalability and Security Considerations

Reliability

- The deployment runs with multiple replicas so the service is not dependent on a single pod.

- Readiness and liveness probes ensure traffic is only routed to healthy containers and allow Kubernetes to restart pods that become unresponsive.

- A PodDisruptionBudget ensures that at least two pods remain available during voluntary disruptions such as node maintenance or upgrades.

- Together these provide a reasonable reliability baseline for a stateless service without introducing unnecessary complexity.

Scalability

- A HorizontalPodAutoscaler is used to scale the deployment based on CPU utilisation so the service can react to traffic spikes automatically.

- CPU is not a perfect signal for HTTP load, but it is a simple and commonly used starting point.

- In a production system I would likely scale on request rate or latency using custom metrics (for example Prometheus Adapter or KEDA).

Security

- TLS termination is handled at the AWS Application Load Balancer using ACM, which keeps certificate management and private keys outside the cluster.

- This approach aligns well with typical EKS deployments and simplifies certificate management.

- A basic NetworkPolicy is included to reduce unnecessary pod exposure. For simplicity it currently allows traffic from any source on port 80.

- In a production environment I would further restrict ingress to the ingress controller and enforce stronger pod and container security contexts
---

# 6. Improvements With More Time

- Additional configuration that varies across environments (such as replica counts, resource limits, and ingress settings) would be exposed through Helm values to improve reusability.

- Namespaces would be used properly to keep application workloads separate from other cluster components.

- Observability and monitoring would be added so metrics, logs, and alerts help detect and troubleshoot issues.

- More meaningful HPA metrics would be used such as request rate or latency instead of relying only on CPU.

- I used the provided image reference for the assignment, but in production I would pin the image to a fixed version or digest to ensure reproducible deployments. The CI pipeline would update this reference as part of the release process.

- Rolling update strategy – Explicit RollingUpdate settings (maxSurge / maxUnavailable) would be added to control deployment behaviour and avoid capacity drops during upgrades.

- Pod and container security contexts (for example runAsNonRoot and allowPrivilegeEscalation: false) would be enforced to reduce container privilege levels.

---

# 7. How to Deploy the Chart

This section describes deployment to a **local Minikube cluster**.

If deploying to **EKS**, a certificate should be generated with AWS Certificate Manager and ingress enabled.

---

## Local Deployment

Start a local cluster:

```bash
minikube start --driver=<docker>
minikube addons enable metrics-server
```

Lint the chart:

```bash
helm lint .
```

Install the chart:

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

---

## Test Autoscaling (HPA)

Generate load:

```bash
kubectl run loadgen -n <namespace-name> --rm -it --image=busybox -- /bin/sh
```

Inside BusyBox:

```bash
while true; do wget -q -O- http://<service-name>/get > /dev/null; done
```

Watch scaling:

```bash
kubectl get hpa -n <namespace-name> -w
```

---

## Release Management

```bash
helm upgrade <release-name> . -n <namespace-name>
helm history <release-name> -n <namespace-name>
helm rollback <release-name> 1 -n <namespace-name>
```

---

## Cleanup

```bash
helm uninstall <release-name> -n <namespace-name>
kubectl delete namespace <namespace-name>
```