# Phase 2 — Findings & Postmortems

Real incidents encountered while building the 3-service deployment, documented in the same structured retrospective format used throughout this project: **symptom → misleading signal → command run → wrong hypothesis → actual root cause → fix and verification.** These are kept here rather than smoothed out of the main README because the debugging process is the actual point of an infrastructure engineering portfolio — not just the fact that it eventually worked.

---

## 1. Cross-node pod networking silently broken by an incomplete security group

**Symptom:** `helm upgrade --install` failed with `failed calling webhook "validate.nginx.ingress.kubernetes.io" ... context deadline exceeded`, immediately after `--atomic` rolled back the release.

**Misleading signal:** the ingress-nginx controller pod, checked immediately after the failure, showed `1/1 Running` with a restart that had happened hours earlier — not during the failure window. This produced a plausible but wrong first hypothesis: "the pod must have been mid-restart when the webhook call happened."

**Command that disproved the wrong hypothesis:** `curl -k https://<pod-ip>:8443 --max-time 10`, run directly from the control plane to the webhook pod's own IP, in a completely calm, stable moment — and it still timed out completely. A pod-readiness theory cannot explain a persistent, reproducible failure at network layer, hours after the pod stabilized.

**Actual root cause:** the control plane's Terraform-defined security group had only two inbound rules — `6443` (API server) and `22` (SSH). It had **no general intra-VPC allow rule**, unlike the worker security group, which explicitly allowed all traffic within the VPC CIDR. Calico's default pod networking mode (`ipipMode: Always`) encapsulates cross-subnet pod traffic as IP-in-IP (IP protocol 4, not TCP/UDP) — a protocol whose AWS Security Group connection-tracking behavior is documented by Calico itself as unreliable. The control plane's workers (spread across 2 subnets/AZs for anti-affinity) are never on the same L2 segment as the control plane, so this encapsulation is always in play, and the missing SG rule silently dropped the return traffic.

**Fix:** added an explicit `ingress { protocol = "-1", cidr_blocks = [var.vpc_cidr] }` rule to the control plane's security group in Terraform, mirroring what the worker SG already had.

**Verification:** re-ran the same direct pod-IP curl test — result changed from a silent timeout to a fast `Connection refused` (port not listening, but packet delivered) confirming the tunnel path itself was now open. Cross-node traffic confirmed working before any further layers were built on top.

**Why this matters beyond this one bug:** this is a real example of a Kubernetes CNI behavior (encapsulation requirement driven by subnet topology) colliding with cloud-provider security semantics (SG connection tracking) in a way neither layer's own documentation would surface in isolation — the failure only becomes visible by testing the actual cross-layer interaction directly.

---

## 2. A one-line Terraform change destroyed the entire environment

**Symptom:** after adding the single missing security group rule above, `terraform apply` destroyed and recreated all 5 EC2 instances, the cluster, Jenkins, and the manually-created ALB/Target Group — all state lost.

**What actually happened:** `terraform apply` was run directly, without first reviewing `terraform plan` output for `-/+` (destroy-and-recreate) versus `~` (in-place update) markers. Something about the change or existing state drift caused Terraform to treat the security group modification as requiring replacement rather than an in-place update.

**The real lesson, stated precisely:** `terraform state` (list/show/mv/rm) inspects what Terraform *believes* exists — it is not a preview tool. The correct habit is `terraform plan -out=tfplan`, reading every resource's change symbol before ever applying, and applying only the reviewed plan file (`terraform apply tfplan`) rather than a freshly-recomputed one.

**Fix:** full infrastructure rebuild from a corrected Terraform config (SG fix included from the start), this time applying only after a full, symbol-by-symbol plan review.

**Cost of the mistake:** a full re-run of cluster bootstrap, Calico install, Jenkins/Tomcat setup (as a properly non-root `tomcat` user this time), RBAC re-creation, ALB/Target Group re-creation, and pipeline re-verification — effectively repeating several hours of prior work. Documented here specifically because the *recovery* process is itself a legitimate demonstration of the project's repeatability: every step was re-derivable from documentation already written during the first pass, nothing had to be rediscovered from scratch.

---

## 3. Target group marked all nodes unhealthy despite the app working correctly

**Symptom:** after fixing #1 and rebuilding, the ALB's Target Group showed all 3 workers as `unhealthy`, and browser access timed out — despite `curl` from the control plane to the ingress-nginx NodePort returning a valid response.

**Misleading signal:** the "valid response" was actually the bug — `curl -v http://<worker-ip>:30080/healthz` returned `HTTP/1.1 404 Not Found`. Because the connection succeeded and a well-formed HTTP response came back, it was easy to misread this as "networking is fine, must be an AWS-side ALB config problem" and start investigating security groups again.

**Actual root cause:** `/healthz` is not a path ingress-nginx serves on its main traffic port. That path exists on a separate metrics/health port (`10254`), which was never exposed on the NodePort Service. Hitting `/healthz` on the main port (`30080`) correctly returns nginx's own 404 default-backend response — a *correct* response, just not a `200`, which is all AWS's health checker accepts by default.

**Fix:** changed the Target Group's health check "success codes" from `200` to `200,404`. This is a legitimate, documented pattern for fronting ingress-nginx with an ALB without the AWS Load Balancer Controller — a 404 from the default backend is valid proof the controller itself is alive and correctly declining to route an unmatched path.

**Verification:** targets returned to `healthy` within one health-check interval; browser access worked immediately after.

---

## 4. The Ingress path rewrite worked for root pages but broke every internal link

**Symptom:** `/orders` (later `/order`) rendered correctly, but clicking "Add Order" produced a 404, with the browser's address bar showing `<alb-dns>/form.html` — missing the `/order` prefix entirely.

**Root cause, in two layers (found sequentially, not simultaneously):**

1. **First layer:** the original Ingress used `rewrite-target: /` with plain `pathType: Prefix` matching. This strips the entire matched prefix unconditionally, discarding any sub-path — meaning internal navigation links inside the app (which have no knowledge of being served under a path prefix) always got forwarded incorrectly. Fixed with a regex capture group (`path: /order(/|$)(.*)`, `rewrite-target: /$2`, `pathType: ImplementationSpecific`) so sub-paths are preserved through the rewrite.
2. **Second layer, found only after applying the first fix and re-testing:** the regex capture pattern alone wasn't enough — ingress-nginx additionally requires the `nginx.ingress.kubernetes.io/use-regex: "true"` annotation to actually interpret the path as a regex with capture groups at all. Without it, the path matched literally, and `$2` was never populated correctly.

**A third, distinct issue found during the same testing pass:** even with both fixes applied, navigating directly to the bare path (`/order`, no trailing slash) still broke *subsequent* relative links on that page — a standard browser URL-resolution behavior (a URL with no trailing slash is treated as a file, not a directory, so relative links resolve against the parent). The textbook fix (an Ingress `configuration-snippet` issuing a 301 redirect to the trailing-slash form) was evaluated and **deliberately not applied**: `configuration-snippet` annotations have been disabled by default in ingress-nginx since v1.9, specifically due to a documented CVE (arbitrary NGINX config injection via a maliciously crafted Ingress object) — and enabling it cluster-wide would have handed that same capability to the already-broadly-scoped `jenkins-deployer` RBAC identity, a materially worse tradeoff than the cosmetic issue it would fix. **Decision: always navigate with the explicit trailing slash (`/order/`); no controller-level change made.**

**Verification:** confirmed via direct `curl` to both the bare path and a sub-path (`/order`, `/order/form.html`) before ever retesting in a browser — browser-level caching of the earlier 404 responses had briefly caused confusion where `curl` evidence and browser behavior appeared to disagree.

---

## 5. Inter-service failure simulation: NetworkPolicy block on `customer`

**Scenario:** a Calico `NetworkPolicy` denying all ingress to the `customer` service's pods, simulating an overly broad policy accidentally cutting off a real dependency — chosen deliberately over simpler alternatives (scaling to 0, breaking a Service selector) specifically to exercise the CNI enforcement capability Calico was selected for in this project, but never previously used.

**Prediction made before running the test:** based on Spring Boot Actuator's default health indicator not checking downstream dependencies, `order`'s pod would stay `Ready` throughout (confirmed correct) — but the *blast radius* was initially mis-predicted as limited to the "Add Order" form only.

**Actual blast radius, found from evidence:** the order **list** page also calls `CustomerClient` (to resolve each order's customer name for display), so the entire `order` service became unusable, not just one form — a wider blast radius than predicted, because a single downstream dependency turned out to be load-bearing across the whole service's UI, not one isolated feature.

**Failure signature:** `java.net.ConnectException: Connection timed out`, distinct from the `UnknownHostException` seen in an earlier, unrelated standalone-Docker test. DNS resolution succeeded (CoreDNS is unaffected by NetworkPolicy); the TCP handshake itself was silently dropped by Calico's enforcement — no `RST`, no fast rejection, just silence until timeout.

**Measured impact, not assumed:** the browser-facing request took a full 60 seconds to fail, ending in a `504 Gateway Time-out` **from the ALB itself** (its own idle timeout firing first) — not from `order`. The underlying `RestTemplate` call had no explicit connect-timeout configured, so the true hang duration inside `order` was likely longer; the ALB simply gave up and disconnected first.

**Compounding risk identified, not just the direct symptom:** every hung request holds an `order` server thread for the full duration of the hang. Under real concurrent load during an outage like this, `order`'s entire thread pool could exhaust — a cascading failure where `order` stops responding to *any* request, including ones with no dependency on `customer` at all.

**Decision — documented gap, not fixed:** the correct fix (explicit `RestTemplate` connect/read timeouts, ideally paired with a circuit-breaker library) is application code, outside this project's declared infra/deployment scope. The infra-level finding that *is* in scope: Kubernetes' readiness-probe model, using only local health checks, is structurally blind to this entire failure class — no infra-level mechanism (Service Endpoints, load balancing, autohealing) can mitigate it as currently configured. A genuinely infra-level fix exists (a service mesh sidecar — e.g. Istio `DestinationRule` with `outlierDetection` — enforcing connection timeouts and circuit-breaking transparently, with zero application code changes) but was not implemented in this phase; noted here as the correct next step if this project continues.

**Recovery:** `kubectl delete -f block-customer-ingress.yaml` — instant recovery, no pod restart required, confirming the failure was purely network-policy-imposed with no lasting application-level damage.

---

## Comparing Phase 1 and Phase 2 failure classes

| | Phase 1 (DB outage) | Phase 2 (inter-service block) |
|---|---|---|
| Detected by Kubernetes? | Yes — readiness probe pulled the pod from Service Endpoints in seconds | No — pod stayed `Ready` the entire time |
| User-facing impact | Zero — traffic never reached the broken pod | Full — every request needing `customer` hung for 60s, then failed |
| Failure signature | Fast `500`, JDBC exception, clean and immediate | Silent hang, `ConnectException` after full OS-level timeout |
| Root cause layer | External dependency (RDS) unreachable | Network policy blocking a Kubernetes-internal dependency |
| Fix location | Infra (probe already correctly scoped) | Application code (out of this project's scope) — infra-level mitigation exists but unimplemented |

This contrast is the actual point of Phase 2's failure simulation: proving that a single-app architecture's failure model (Phase 1) does not generalize to a distributed one, and that Kubernetes' built-in self-healing has real, specific blind spots that only become visible by deliberately testing for them.

---

## What could be improved — known gaps, stated plainly

Consistent with how this project has treated every deliberate tradeoff (NAT Gateway avoidance, spot-instance risk, ASG deferral) — these are known limitations of the current setup, not oversights discovered too late to matter. Listed in rough priority order if this project continued.

| Gap | Why it matters | What the real fix looks like |
|---|---|---|
| **No observability stack in Phase 2** | Phase 1 had `kube-prometheus-stack` on a dedicated monitoring node; Phase 2's cluster has zero metrics/dashboards. The NetworkPolicy failure simulation's 60-second hang and thread-pool risk were only visible through manual `curl` timing and log-grepping — in a real incident, this would need to be visible on a dashboard in real time, not reconstructed after the fact. | Re-apply the same Helm-based `kube-prometheus-stack` pattern from Phase 1, plus a Grafana panel specifically tracking `order`'s HTTP thread pool utilization and request latency — the exact metric that would have caught the cascading-failure risk before it became a full outage. |
| **No pod autoscaling (HPA)** | All 3 services run a fixed `replicas: 1`. Under the load conditions that make the NetworkPolicy failure genuinely dangerous (concurrent requests exhausting `order`'s thread pool), more replicas would directly reduce blast radius per-pod, though wouldn't fix the root cause. | Add a Horizontal Pod Autoscaler per service, scaling on CPU or (better) a custom metric tied to request latency/thread pool saturation. |
| **No cluster/node autoscaling (ASG)** | Explicitly deferred in this project's Phase 2 planning — `kubeadm` has no native node join/leave automation the way EKS managed node groups do; fixed-count workers were chosen instead. Still a real gap for anything beyond a fixed, known workload. | Either migrate to EKS managed node groups, or build the manual equivalent (bootstrap script + SSM-refreshed join token + an ASG lifecycle hook draining and removing the Node object before termination). |
| **No connect/read timeouts on inter-service HTTP calls** | The direct cause of the 60-second hang in the failure simulation. Deliberately left unfixed in this project since it's application code, but it's the single highest-leverage fix if scope ever expanded to include the app layer. | Explicit `setConnectTimeout()`/`setReadTimeout()` on each service's `RestTemplate`, ideally paired with a circuit-breaker library (Resilience4j) so repeated failures short-circuit instead of retrying into an already-known-bad dependency. |
| **No service mesh** | The one genuinely infra-level (no app code) fix for the failure simulation's actual danger — a sidecar proxy enforcing connection timeouts and circuit-breaking transparently. Not implemented in this phase; evaluated only as a design discussion. | Istio or Linkerd sidecar injection, with a `DestinationRule`/equivalent defining `connectionPool` timeouts and `outlierDetection` for `customer` and `catalog`. |
| **No `PodDisruptionBudget`** | With `replicas: 1` per service, a node drain (e.g. during a Kubernetes upgrade, or an EC2 stop/start) can take a service fully offline with no protection, unlike Phase 1's 2-replica login-app. | A `PodDisruptionBudget` per service once replica counts are raised above 1 — meaningless at `replicas: 1`, since any eviction already violates any non-zero `minAvailable`. |
| **Local Terraform state, no remote backend** | Accepted as reasonable for a single-operator project, per Phase 2's original planning — but the environment-loss incident (Findings #2) is exactly the kind of event state locking and versioned remote state (S3 + DynamoDB) are designed to make recoverable from, rather than requiring a full rebuild. | S3 backend with DynamoDB state locking, plus enabling S3 versioning specifically so a bad apply's prior state is recoverable, not just visible in `terraform plan`. |
| **`/customers` external path mismatch never fixed** | The external Ingress path for the customer service is `/customers` (plural), but the service's actual Spring Data REST collection lives at `/customer` (singular) — meaning the external path currently only reaches the API root document, not the real collection. Left as-is since it didn't block the core inter-service proof, but it's a real, fixable inconsistency. | One-line fix in the Helm chart's `values.yaml` (`path: /customers` → `path: /customer`), or add both as separate Ingress rules if plural is preferred as the public-facing convention. |

