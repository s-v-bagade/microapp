# HOW-TO-RUN — Phase 2 Deployment

This document is written to be read, not just followed. Every major mechanism (the Dockerfiles, the Jenkinsfile, the Ingress routing, the Helm chart) has a dedicated explanation of *why* it's built the way it is, not just the commands to run it. If you only want the commands, the numbered sections below are still fully sequential and runnable on their own — the deep-dive subsections are clearly marked and can be skipped on a first read.

Assumes familiarity with the Phase 1 login-app setup — this reuses the same `kubeadm`/Jenkins/RBAC patterns; only what's genuinely different for a 3-service Helm-based deployment is explained in full here.

## Related repositories

- **This repo** — application source (forked `microservice-kubernetes-demo`, 3 services + rewritten Dockerfiles)
- **Terraform config** — `https://github.com/s-v-bagade/tf-config-microapp.git`
- **Kubernetes manifests / Helm chart** — `https://github.com/s-v-bagade/k8s-manifests-microapp.git`

---

## 1. Infrastructure

```bash
terraform plan -out=tfplan   # read every line — confirm no unexpected -/+ replacements
terraform apply tfplan
```

Provisions the VPC, subnets, security groups, and 5 EC2 instances (control plane, 3 workers, Jenkins). The control plane's security group must include a general intra-VPC allow rule (`protocol = "-1"`, `cidr_blocks = [var.vpc_cidr]`) — without it, Calico's cross-node pod traffic is silently dropped. See FINDINGS.md #1 for the full diagnosis of what happens if this rule is missing, and **always review the plan output before applying** — see FINDINGS.md #2 for why a seemingly small change can trigger a full destroy-and-recreate.

---

## 2. Cluster bootstrap — control plane + 3 workers

On all 4 nodes: install `containerd` (or `docker`, which on Amazon Linux 2023 pulls in `containerd` as a dependency and produces an identical result — `kubeadm` auto-detects whichever CRI socket exists), then `kubelet`/`kubeadm`/`kubectl` pinned to the same version across all nodes. **Explicitly `systemctl enable` both `kubelet` and `containerd`** — a disabled-on-boot service was a real Phase 1 incident after an EC2 stop/start.

```bash
sudo kubeadm init --apiserver-advertise-address=<control-plane-private-ip> --kubernetes-version=v1.35.0
mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# on each worker:
sudo kubeadm join <control-plane-private-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify cross-node pod networking before proceeding** — this is not optional, given FINDINGS.md #1:

```bash
kubectl get pods -n calico-system -o wide   # note a pod IP on a worker
curl -v http://<that-pod-ip>:9099/liveness --max-time 5   # must not silently time out
```

---

## 3. Ingress Controller

```bash
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.publishService.enabled=false
```

`NodePort` is required (not `LoadBalancer`) because there's no cloud-controller-manager wired to AWS on a self-managed `kubeadm` cluster — nothing would ever fulfill a `LoadBalancer`-type Service request. `publishService.enabled=false` disables a mechanism (writing a `LoadBalancer`'s external IP back onto Ingress status) that only applies in managed-cloud environments this cluster doesn't have.

---

## 4. ALB + Target Group (manual, console)

- Target Group: Instances, HTTP/30080, health check path `/healthz`, **success codes `200,404`**
- ALB: internet-facing, both public subnets, reuse the Terraform-managed ALB security group, HTTP:80 listener → the Target Group above

**Why `200,404` and not just `200`:** ingress-nginx's actual health/readiness endpoint lives on a separate port (`10254`), never exposed on the NodePort Service. Hitting `/healthz` on the main traffic port (`30080`) correctly returns nginx's own 404 default-backend response — proof the controller is alive and correctly declining to route an unmatched path, not a failure. AWS's health checker only accepts `200` by default, so it has to be told `404` counts as healthy here. Full incident writeup in FINDINGS.md #3.

---

## 5. Jenkins (Tomcat + WAR, not the RPM package)

```bash
sudo useradd -r -m -d /home/tomcat -s /bin/false tomcat
# install Tomcat under /mnt/server, drop jenkins.war into webapps/
sudo chown -R tomcat:tomcat /mnt/server/apache-tomcat-10.1.59
# create /etc/systemd/system/tomcat.service (User=tomcat, JENKINS_HOME=/home/tomcat/.jenkins)
sudo systemctl daemon-reload && sudo systemctl enable --now tomcat
```

`User=tomcat` (not root) matters beyond convention: whatever OS user runs Jenkins is the identity every pipeline `sh` step executes as. Running as root would mean a compromised or buggy Jenkinsfile has full host access — undermining the entire point of the namespace-scoped `jenkins-deployer` RBAC identity described below, since an attacker with root wouldn't need to go through Kubernetes RBAC at all.

Install `docker`, `kubectl`, `helm` on this box; `usermod -aG docker tomcat`; restart Tomcat.

---

## 6. RBAC + scoped kubeconfig for Jenkins

Apply cluster-side, using the cluster-admin kubeconfig — **never applied by Jenkins itself.** This is a deliberate ordering constraint, not an arbitrary choice: if Jenkins applied its own RBAC, it would need permissions to create `Role`/`RoleBinding` objects *before* it has any permissions at all — a chicken-and-egg problem solvable only by either giving it cluster-admin first (defeating the entire point) or having it fail on its first run.

```bash
kubectl apply -f jenkins-rbac.yaml
kubectl create token jenkins-deployer --duration=8760h -n default
```

**The Role's scope, and why each part is there:**

| Resource | Verbs granted | Why |
|---|---|---|
| `deployments`, `services` | get/list/watch/create/update/patch | Core objects `helm upgrade` creates/updates every deploy |
| `ingresses` | get/list/watch/create/update/patch | New in Phase 2 — Phase 1 had no Ingress object at all |
| `secrets` | get/list/watch/create/update/patch | **The non-obvious one.** Helm 3 stores each release's metadata as a Kubernetes `Secret` in the same namespace — without this, `helm upgrade --install` fails with an error that doesn't obviously point back to "Helm needs Secrets access" |
| `pods`, `pods/log`, `endpoints`, `replicasets` | get/list/watch only | Read-only visibility for debugging; never needed for a deploy to succeed |

**Deliberately excluded:** `delete` on anything, any access outside the `default` namespace, any cluster-scoped resource (`nodes`, etc.).

Assemble the token into a kubeconfig (bearer-token auth, not the admin's client-cert auth — the same four-section kubeconfig structure either way, only the `users` section's proof-of-identity mechanism differs), `chmod 600`, `chown tomcat:tomcat`.

**Verify before uploading anywhere — this is the actual proof of least-privilege, not just configuration:**

```bash
KUBECONFIG=<path> helm upgrade --install test-release <manifests-path> -n default --dry-run   # must succeed
KUBECONFIG=<path> kubectl get nodes                                                            # must be Forbidden
KUBECONFIG=<path> kubectl delete deployment order -n default                                   # must be Forbidden
```

A `Forbidden` response (not a connection/auth error) proves authentication succeeded but authorization correctly blocked the action — the actual demonstration of least-privilege in action.

Upload as Jenkins Secret-file credential `kubeconfig-cred-id`. Add `dockerhub-creds`, `github-cred`. Create the Pipeline job pointing at this repo's `Jenkinsfile`.

---

## Dockerfile — Build Logic Explained

Each of the 3 services has its own multi-stage Dockerfile, all following the identical pattern. Using `order`'s as the reference:

```dockerfile
FROM maven:3.8.6-openjdk-11 AS build
WORKDIR /build

COPY pom.xml .
RUN mvn -N install

COPY microservice-kubernetes-demo-order/pom.xml microservice-kubernetes-demo-order/pom.xml
WORKDIR /build/microservice-kubernetes-demo-order
RUN mvn dependency:go-offline -B

COPY microservice-kubernetes-demo-order/src ./src
RUN mvn clean package -DskipTests

FROM openjdk:11.0.2-jre-slim
WORKDIR /app
COPY --from=build /build/microservice-kubernetes-demo-order/target/microservice-kubernetes-demo-order-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-Xmx400m", "-Xms400m", "-jar", "app.jar"]
```

**Why two stages at all.** `FROM ... AS build` starts a temporary environment with the full JDK and Maven — everything needed to compile, but nothing that should ship. `FROM openjdk:11.0.2-jre-slim` starts a completely separate, second image containing only a JRE (no compiler, no Maven). The single `COPY --from=build` line is the only bridge between them — it reaches back into the build stage and pulls out just the finished jar. Everything else from stage 1 (Maven itself, all downloaded dependency jars, the full `src` tree) never makes it into the final shipped image. This is a real security and size benefit, not just a style preference — a smaller image has a smaller attack surface and nothing to reverse-engineer build tooling from.

**Why the parent POM has to be handled specially.** The three services are not independent Maven projects — each one's `pom.xml` declares the root `microservice-kubernetes-demo/pom.xml` as its `<parent>`, meaning Maven cannot resolve any of the three modules in isolation. Two consequences, both visible in the Dockerfile above:

- The Docker build **context** has to be the *parent* folder (`microservice-kubernetes-demo/`), not the individual service subfolder — otherwise `COPY pom.xml .` would grab the wrong file entirely.
- `RUN mvn -N install` installs *only* the parent artifact into the local Maven repo — the `-N` flag means non-recursive, so this doesn't try to also build the other two sibling services, which don't exist in this build's copied file tree at all.

**Why the COPY order matters — this isn't arbitrary.** Notice `pom.xml` files are copied and their dependencies resolved (`mvn dependency:go-offline`) *before* the actual `src` folder is copied. Docker caches each instruction as a layer; if a layer's inputs haven't changed, Docker reuses the cached result instead of re-running it. Since dependency resolution only depends on `pom.xml` (not source code), editing a Java file and rebuilding will skip re-downloading every dependency from scratch — a real, meaningful build-time saving that only works because of this specific ordering.

**Why `-Xmx400m -Xms400m` are set explicitly.** The JVM, left to its own defaults, sizes its heap based on the **node's total memory**, not the container's cgroup memory limit — a well-known cause of unexpected `OOMKilled` events, since a JVM can allocate a heap far larger than what Kubernetes has actually granted the pod. Pinning explicit heap flags sidesteps this regardless of which physical node the pod lands on. In the Helm chart, this value is deliberately kept below the pod's `resources.limits.memory` (not equal to it) — non-heap JVM overhead (thread stacks, metaspace, JIT code cache) sits outside the heap but still counts against the container's memory limit, so headroom is needed even after fixing the heap-sizing issue itself.

---

## Jenkinsfile — Pipeline Logic Explained

**Docker Build stage — 3 images, one shared context:**

```groovy
dir('microservice-kubernetes-demo') {
    sh """
        docker build -f microservice-kubernetes-demo-order/Dockerfile -t ${DOCKERHUB_REPO}-order:${IMAGE_TAG} .
        docker build -f microservice-kubernetes-demo-customer/Dockerfile -t ${DOCKERHUB_REPO}-customer:${IMAGE_TAG} .
        docker build -f microservice-kubernetes-demo-catalog/Dockerfile -t ${DOCKERHUB_REPO}-catalog:${IMAGE_TAG} .
    """
}
```

`dir('microservice-kubernetes-demo')` changes Jenkins' working directory to the parent folder for exactly the reason explained above — the multi-module POM resolution requires it. `-f <service>/Dockerfile` points at each service's specific Dockerfile while the build context (the trailing `.`) stays at the shared parent level.

**Deploy to Kubernetes stage — tag substitution, then a single atomic Helm release:**

```groovy
withCredentials([file(credentialsId: 'kubeconfig-cred-id', variable: 'KUBECONFIG')]) {
    sh """
        sed -i "s|__IMAGE_TAG__|${IMAGE_TAG}|g" manifests/values.yaml
        helm upgrade --install microapp manifests/ -n default --atomic --timeout 180s
    """
}
```

**Why `sed` on `values.yaml`, not `--set`.** Helm's `--set` merges at the *list* level for any array in `values.yaml` — since all 3 services live in one `services:` list, setting a field on one index (`--set services[0].tag=...`) silently replaces the *entire list* with just that one partial item, wiping every other field on all three services. `sed`-templating the placeholder string directly avoids this entirely, and updates all three services' tags in a single pass since they share the same literal placeholder.

**Why `variable: 'KUBECONFIG'` has to be that exact name.** `kubectl` and `helm` don't accept a config path as a command-line argument by default — they look for an environment variable literally named `KUBECONFIG`. `withCredentials` writes the uploaded credential file to a temp path and sets that exact environment variable for the duration of the `sh` block. Naming it anything else would mean `kubectl`/`helm` silently fall back to `~/.kube/config` (which doesn't exist for the `tomcat` user), producing a connection error that looks unrelated to the actual cause.

**Why `--atomic` replaces Phase 1's separate rollback logic.** Phase 1 used `kubectl rollout undo` inside a `post { failure { ... } }` block as an explicit, separate command. `--atomic` (which implies `--wait`) makes Helm itself detect a failed upgrade and automatically revert to the last successful release, as one property of the `helm upgrade` command rather than a separate step — and because the entire multi-service deploy (all 3 Deployments + Services + the Ingress) is one Helm *release*, there's no risk of a partial rollback landing on an inconsistent intermediate state the way raw `kubectl apply` calls could.

---

## Ingress — Routing Logic Explained

```yaml
annotations:
  nginx.ingress.kubernetes.io/use-regex: "true"
  nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /order(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: order
            port:
              number: 8080
```

**Why the path is a regex with a capture group, not a plain prefix.** A plain `pathType: Prefix` match with `rewrite-target: /` strips the entire matched prefix unconditionally — meaning any sub-page (`/order/form.html`) would get forwarded to the `order` pod stripped down to just `/`, losing `form.html` entirely. `(/|$)(.*)` matches either a trailing slash or end-of-string, then captures everything after into a group; `rewrite-target: /$2` substitutes that captured group back in, preserving the sub-path through the rewrite.

**Why `pathType: ImplementationSpecific`, not `Prefix`.** The core Kubernetes Ingress spec's `Prefix` match type is a plain literal-prefix comparison — it has no concept of regex capture groups at all. `ImplementationSpecific` tells Kubernetes "don't validate this against the generic spec, let the specific controller (nginx) interpret the path string however it wants" — the deliberate escape hatch that lets nginx apply its own regex engine to this path.

**Why `use-regex: "true"` is required in addition to the capture-group syntax.** Without this annotation, nginx doesn't actually treat the path string as a regex at all, regardless of `pathType` — the capture groups are never populated, and `$2` resolves to nothing.

**A known, deliberately unfixed gap:** navigating directly to a bare path with no trailing slash (`/order` rather than `/order/`) still breaks *subsequent* relative links on that page — a standard browser URL-resolution behavior with no Ingress-level involvement. The standard fix (an Ingress `configuration-snippet` issuing a redirect) was deliberately not applied, since `configuration-snippet` annotations are disabled by default in ingress-nginx since v1.9 due to a real CVE (arbitrary NGINX config injection via a crafted Ingress object) — and this project's Jenkins RBAC already grants `create`/`update` on `ingresses`, meaning enabling snippets cluster-wide would hand that injection capability to the CI/CD identity. Full reasoning in FINDINGS.md #4. **Always use the trailing-slash form when testing manually.**

---

## Helm Chart — Templating Logic Explained

`values.yaml` holds one list, `services`, with one entry per microservice (`name`, `image`, `tag`, `port`, `path`, `memoryLimit`, `jvmHeap`). Every template (`deployment.yaml`, `service.yaml`, `ingress.yaml`) loops over this same list with `{{- range .Values.services }}`, so **one template file produces three concrete Kubernetes objects** — adding a fourth microservice in the future would mean adding one entry to `values.yaml`, not writing new YAML files.

**Why Service names are load-bearing, not cosmetic.** `order`'s Java code calls `http://catalog:8080/catalog/` and `http://customer:8080/customer/` directly — plain Kubernetes Service DNS names, hardcoded as `@Value` defaults in the source. This means the Helm chart's `service.yaml` template **must** produce Services named exactly `catalog` and `customer` — not `catalog-service`, not any other convention — or `order` throws `UnknownHostException` at runtime. This was confirmed directly from a stack trace and the `CustomerClient.java` source during initial testing, not assumed.

**Why `type: ClusterIP`, not `NodePort`, for the three application Services.** None of the three services need to be reached directly from outside the cluster — `order`↔`catalog`/`customer` traffic is entirely internal (resolved via CoreDNS, routed via `kube-proxy`), and external browser traffic reaches all three only through the Ingress Controller, which itself only needs `ClusterIP`-level reachability to forward to them. Only ingress-nginx's own Service needs `NodePort`, since it's the one thing an external ALB has to reach.

**Why probes are split the same way Phase 1's were.** `readinessProbe` checks `/actuator/health` (Spring Boot Actuator's built-in endpoint); `livenessProbe` checks `/actuator/health/liveness` — a distinct, narrower endpoint Spring Boot exposes specifically to answer "is the process itself alive," independent of any dependency's health. This mirrors the exact reasoning behind Phase 1's readiness/liveness split (avoiding a restart-loop when a downstream dependency, not the app itself, is the thing that's broken) — with one important caveat: Actuator's default health indicator does **not** check `catalog`/`customer` reachability at all, meaning this readiness probe cannot detect the inter-service failure class documented in FINDINGS.md #5. This is a known, explicitly scoped-out gap, not an oversight.

---

## Verifying it's actually working

```bash
kubectl get pods -n default            # 3 pods, 1/1 Running
kubectl get svc -n default             # Services named exactly catalog, customer, order
helm list -n default                   # release "microapp", status deployed
```

```bash
curl -v http://<alb-dns>/order/        # trailing slash required — see FINDINGS.md #4
```

Should render the order list, including real customer names — proof `order` is successfully calling `customer`/`catalog` over live Kubernetes Service DNS. Watch it happen live:

```bash
kubectl logs -f -l app=order -n default
```

`/catalog` and `/customer` (external paths) return Spring Data REST's raw HAL+JSON API root, not an HTML page — this is correct, expected behavior: these two services are treated as internal, API-only dependencies from the outside, with `order` as the only real human-facing surface.

---

## Reproducing the failure simulation

```bash
cat <<'EOF' > block-customer-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-customer-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: customer
  policyTypes:
  - Ingress
  ingress: []
EOF

kubectl apply -f block-customer-ingress.yaml
kubectl get pods -n default                       # customer stays 1/1 Running — the policy doesn't break the pod itself
curl -v http://<alb-dns>/order/ --max-time 65      # expect ~60s hang, then a 504 from the ALB
kubectl logs -l app=order -n default --tail=200 | grep -A5 "ERROR"   # ConnectException: Connection timed out

kubectl delete -f block-customer-ingress.yaml
curl -v http://<alb-dns>/order/ --max-time 10      # immediate recovery, no restart needed
```

Full analysis of why this specific NetworkPolicy shape was chosen, and what the results actually mean, is in FINDINGS.md #5.

---

## Stopping/starting instances between sessions

Private IPs, kubeconfig, and RBAC all persist across a stop/start — no reconfiguration needed. Public IPs are reassigned on start; re-run `terraform output` for fresh SSH/Jenkins URLs. Target Group will briefly show all targets `unhealthy` until workers finish rejoining — expected, resolves on its own within a couple of health-check intervals. Confirm `kubectl get nodes`, `calico-system` pods, and `ingress-nginx` pods are all healthy **before** triggering any new Jenkins build after a restart.

