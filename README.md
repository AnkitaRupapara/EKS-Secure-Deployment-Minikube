# Simulated EKS-Style Secure Deployment Using Minikube

## 🛠️ Environment Setup

1. **Install Prerequisites**
   - [Minikube](https://minikube.sigs.k8s.io/docs/start/)
   - [Docker](https://docs.docker.com/get-docker/)
   - [kubectl](https://kubernetes.io/docs/tasks/tools/)
   - [Helm](https://helm.sh/docs/intro/install/)
   - (Optional) k9s, OPA, Kyverno, Falco

2. **Start Minikube Cluster**
   ```sh
   minikube start --cpus=2 --memory=4096 --addons=ingress,metrics-server
   ```

---

## 🧱 Microservice Stack Deployment

1. **Create Namespaces**
   ```sh
   kubectl apply -f manifests/namespaces.yaml
   ```

2. **Deploy Microservices**
   - Each service (`gateway`, `auth-service`, `data-service`) has its own deployment and service YAML in its folder.
   - All are deployed to the `app` namespace.
   - Liveness/readiness probes and resource requests/limits are set in each deployment.

   ```sh
   kubectl apply -f gateway/deployment.yaml
   kubectl apply -f gateway/service.yaml
   kubectl apply -f gateway/ingress.yaml
   kubectl apply -f auth-service/deployment.yaml
   kubectl apply -f auth-service/service.yaml
   kubectl apply -f data-service/deployment.yaml
   kubectl apply -f data-service/service.yaml
   ```

3. **Ingress Setup**
   - Only the `gateway` service is exposed via Ingress.
   - Access via `http://<minikube-ip>/` or configured host.

---

## 🛡️ Simulate IAM with MinIO

1. **Deploy MinIO in `system` Namespace**
   ```sh
   kubectl apply -f manifests/minio.yaml
   ```

2. **Create Secret for MinIO Credentials**
   - Included in `minio.yaml`.

3. **Create ServiceAccount for `data-service`**
   - Included in `minio.yaml`.

4. **Apply RBAC to Restrict MinIO Access**
   ```sh
   kubectl apply -f manifests/minio-rbac.yaml
   ```

5. **Apply NetworkPolicy to Restrict Access**
   ```sh
   kubectl apply -f manifests/networkpolicy.yaml
   ```

6. **Update `data-service` Deployment**
   - Add `serviceAccountName: data-service-sa` under `spec.template.spec`.

7. **Test Access**
   - `data-service` can access MinIO; `auth-service` cannot.

---

## 🚨 Security Incident Simulation

1. **Investigate Authorization Header Leaks**
   - Use `kubectl logs` to check `auth-service` logs.
   - Simulate with:
     ```sh
     kubectl port-forward svc/auth-service 8080:80 -n app
     curl -H "Authorization: secret-token" http://localhost:8080/headers
     ```

2. **Fix the Problem**
   - (Documented) In a real app, update code/config to avoid logging sensitive headers.

3. **Apply NetworkPolicy to Restrict Egress**
   ```sh
   kubectl apply -f manifests/auth-service-networkpolicy.yaml
   ```

4. **(Optional) Apply Kyverno Policy**
   ```sh
   kubectl apply -f kyverno/block-auth-logs.yaml
   ```

---

## 📊 Observability

1. **Install Prometheus & Grafana**
   ```sh
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm install monitoring prometheus-community/kube-prometheus-stack \
     --namespace system --create-namespace \
     --set prometheus.prometheusSpec.maximumStartupDurationSeconds=300
   ```

2. **Access Grafana**
   - Get password:
     ```sh
     kubectl get secret --namespace system monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
     ```
   - Expose Grafana (NodePort example):
     ```sh
     kubectl -n system patch svc monitoring-grafana -p '{"spec": {"type": "NodePort"}}'
     kubectl -n system get svc monitoring-grafana
     ```
   - Access via `http://<server-ip>:<NodePort>`

3. **Create/Export Dashboards**
   - Pod CPU/memory, HTTP request rates/errors, pod restarts.
   - Export as JSON for deliverables.

---

## 🧪 Failure Simulation

1. **Delete a Pod**
   ```sh
   kubectl get pods -n app
   kubectl delete pod <pod-name> -n app
   ```

2. **Observe Recovery**
   - Kubernetes will recreate the pod.
   - Check in Grafana for restarts and metrics.

---

## 📦 Deliverables

- All YAMLs, Helm charts, and policy files
- README.md (this file)
- Architecture diagram (add as image or ASCII)
- Exported Grafana dashboard (JSON)
- Documentation of security incident and failure simulation

---

## 📝 **Design Decisions & Notes**

- Used separate namespaces for system and app workloads.
- Applied least-privilege RBAC and NetworkPolicies.
- Used Helm for observability stack for easy management.
- Documented security incident and prevention steps.
- (Optional) Used Kyverno for runtime policy enforcement.

---

## 🛑 **Assumptions & Known Issues**

- NodePort is used for Grafana for demo purposes; not recommended for production.
- MinIO is used as a mock S3; not for production use.
- Some policies (e.g., Kyverno) are templates and may need adjustment for real-world use.