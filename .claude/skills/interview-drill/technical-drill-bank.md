---
framework_version: 1.5.0
---

# Technical Drill Question Bank (Senior DevOps / SRE / Platform)

Curated and deduplicated from real reported interview question sets (LinkedIn interview-prep
posts, real onsite loops reported for Amazon-style flash-sale scenarios, and general senior
DevOps rounds) — not invented ones. `/drill` treats this as its primary seed source across its
topic menu, pulling from here before generating anything from scratch.

If you've filled in `profile.md` (see the template in this repo), `/drill` builds every model
answer so it's traceable to a fact in that file, or — for a real skill gap your profile
exposes — uses an honest "acknowledge → bridge → learning path" pattern instead of inventing
experience. If you haven't created a `profile.md`, `/drill` still runs: it answers from general
senior-level best practice instead, clearly labeled as an illustrative answer rather than your
personal history. Either way, this file supplies the **questions** and a one-line pointer to
what a strong answer touches, not full scripted answers — answers here would go stale and
defeat the point of a live drill.

Original repeats across source lists (e.g. "pods healthy but users get 502/503" appeared 5+
times) are collapsed into one entry. A `(stretch)` tag marks a question above what most
candidates will have hands-on experience with (Istio, service mesh, Kafka-at-hyperscale,
principal/staff-level org design) - `/drill` should score these on reasoning quality and use
the acknowledge-bridge pattern for the parts genuinely outside your experience, not penalize
the whole answer as a gap.

---

## 1. Linux & Networking

- Explain the complete request flow from a browser to a Kubernetes pod. → DNS, LB/ingress, service, kube-proxy/iptables or IPVS, pod
- TCP vs UDP - where have you actually used each? → reliability/ordering vs latency/loss-tolerance tradeoff, concrete examples
- How does DNS resolution work internally? → resolver → root → TLD → authoritative, caching/TTL
- A production server is at 95% CPU (or high memory/disk). What do you check, in order? → `top`/`htop`, `iostat`, `vmstat`, per-process breakdown, is it real load or a leak/runaway process
- Load Balancer vs Reverse Proxy vs Ingress, with real examples → L4 vs L7, where each sits, why Ingress is Kubernetes-native reverse proxy config
- Public vs private IP; what ports do (22/80/443/53); Subnets and default gateway; how NAT lets private IPs reach the internet; how a firewall/security group allows or denies traffic
- The server is reachable (ping/SSH works) but the application on it isn't - what do you check? (isolates to the app layer, not network: is the process running, is it listening on the expected port, its own logs for a crash, whether it's stuck on a downstream dependency, resource limits/OOM)
- Disk and network troubleshooting as its own walkthrough: disk via `df -h`/`du -sh`, specifically checking inode exhaustion (`df -i`, fails writes even with free space showing - usually a log-rotation misconfiguration); network via ping → traceroute → DNS resolution → the actual port (`nc -zv`/`curl -v`) → security groups/NACLs/firewall → packet loss on the path
- Text-processing/scripting practicals: extract a specific word from a file efficiently (`awk`/`cut`/`grep -o`); delete a specific line/name from a file and move it to another (`grep -v` + redirection, or `sed`); list/display files in reverse order (`ls -tr`, `ls -r`)

## 2. Docker & Kubernetes

- What happens internally when you run `docker run`? → image layers, container runtime, namespaces/cgroups, network setup
- Difference between `CMD` and `ENTRYPOINT`
- A pod is stuck in `CrashLoopBackOff` (or continuously restarting) - how do you troubleshoot it, and would you just restart/delete the pod first? → `kubectl describe`, `kubectl logs` / `--previous`, events, exit code meaning; a 90-minute-session version of this question expects you to go further with `kubectl debug --target <container> -it --image=<toolbox-image>` to attach an ephemeral debug container into the pod's namespace when the app image itself has no shell/tools - restarting/deleting without diagnosing just resets the clock on the same failure
- A pod is stuck in `Pending` (never reaches Running) - how do you troubleshoot it? → distinct failure mode from CrashLoopBackOff: insufficient node CPU/memory to schedule, a `nodeSelector`/affinity rule with no matching node, an unbound PersistentVolumeClaim, or a taint with no matching toleration - `kubectl describe pod` shows the scheduler's actual rejection reason
- A pod is stuck in `ImagePullBackOff` - how do you troubleshoot it? → wrong image tag/typo, private registry auth missing or expired (`imagePullSecrets`), registry unreachable from the node (network/DNS/firewall), or the tag was deleted from the registry
- Kubernetes Service isn't exposing a pod at all (not "pods healthy but 502/503" - traffic never reaches the Service in the first place) → check Service selector matches pod labels exactly, check `kubectl get endpoints` is non-empty, check `targetPort` matches the container's actual listening port, check the Service `type` (ClusterIP/NodePort/LoadBalancer) matches how you're trying to reach it
- Difference between Deployment, StatefulSet, DaemonSet, and Job - when is each the right choice
- Readiness Probe vs Liveness Probe - what happens if you get them backwards
- **Pods are Running/healthy but users see 502/503 errors** - walk the full request path to debug it (the single most-repeated question across every source list - know this cold: readiness vs. liveness, Service selectors/endpoints, upstream/ingress timeout mismatches, pod termination while still receiving traffic)
- How does Kubernetes Service Discovery work (kube-dns/CoreDNS, ClusterIP, endpoints)
- ConfigMaps vs Secrets - how do you manage them consistently across environments
- What is `imagePullPolicy` and when does the wrong setting bite you
- What are the worker-node components (kubelet, kube-proxy, container runtime)
- How would you optimize a large Docker image (multi-stage builds, minimal base images, layer caching)
- PersistentVolume vs PersistentVolumeClaim
- Docker volumes vs bind mounts vs Kubernetes PersistentVolumes - what's the actual difference and when do you reach for each
- Design advanced readiness/liveness probes that catch a business-logic failure, not just an HTTP 200 (e.g. a health endpoint that checks a downstream dependency is actually serving correct data, not just "process is up")
- Kubernetes deployment strategies: Rolling vs Blue-Green vs Canary - be ready to name a real cloud-native implementation, not just the concept: **blue-green on EKS via ALB weighted target groups** (two target groups, shift the ALB's weight from 0/100 toward 100/0 and back on failure) is the concrete AWS-native version of this, distinct from an Istio/mesh-based canary
- Design zero-downtime deployment for a **stateful** app on Kubernetes, including a schema or serialization-format change other services depend on → PodDisruptionBudgets, readiness probes, `preStop` hooks, and for the data side specifically: the **expand/contract pattern** - add the new field/format alongside the old (expand), dual-write to both during migration, backfill, cut readers over, only then remove the old field/format (contract). Feature flags gate which format a service reads, and a kill switch lets you revert the flag instantly if the new path misbehaves - much cheaper than rolling back a schema migration
- Upgrade a Kubernetes cluster with zero downtime → cordon, drain, rolling node upgrade, version-skew compatibility checks
- **Walk through upgrading an Amazon EKS cluster across several minor versions (e.g. v1.34 → v1.36 → v1.37) with zero downtime** - full runbook shape, worth having memorized:
  1. *Pre-checks:* EKS Upgrade Insights, deprecated API usage, Helm chart/app compatibility, VPC CNI/CoreDNS/kube-proxy add-on versions, AWS Load Balancer Controller, PDBs and replica counts, monitoring coverage - test the full upgrade in a lower environment first, never start in prod
  2. *One minor version at a time* (EKS doesn't allow skipping): upgrade the control plane, validate cluster health, then upgrade the required add-ons, before moving to the next version
  3. *Worker nodes via blue-green node groups, not in-place upgrade:* create a new managed node group on an EKS-optimized AMI for the new version; once new nodes are `Ready`, cordon the old nodes and drain **one at a time** (never all at once) so pods reschedule gradually onto the new group
  4. *Why this avoids downtime:* multiple replicas + PodDisruptionBudgets + readiness probes + multi-AZ spread mean draining one node still leaves healthy pods serving traffic elsewhere - the upgrade procedure doesn't create availability by itself, the workload's HA design does
  5. *Monitor throughout:* node health, pod restarts, pending pods, CPU/memory, ALB target health, application latency, 5xx rate
  6. *Validate before decommissioning the old node group:* smoke-test critical flows end to end once everything's on the new nodes, only then remove the old group - then repeat for the next version
  <!-- If you haven't personally run a Kubernetes minor-version upgrade project, present this
  as "here's exactly how I'd run it," not "in one of my projects" - the methodology is sound
  regardless of whether you've shipped it yourself, but don't claim ownership you don't have. -->
- HPA won't scale despite high CPU, or scales on stale metrics → metrics-server lag, wrong metric source, scaling policy/stabilization window misconfigured
- Pod anti-affinity to survive a single-AZ failure during peak traffic
- A Kubernetes node shows `NotReady` - how do you handle it? → `kubectl describe node` first to see the actual condition (kubelet not reporting / network unavailable / disk-memory-PID pressure), which determines the fix rather than guessing; `cordon` + `drain` (respecting PodDisruptionBudgets) to move workloads off safely rather than force-deleting pods; in an ASG-backed node group, letting the ASG terminate and replace the unhealthy instance is often simpler than hand-debugging one node

## 3. AWS & Cloud Infrastructure

- Spot vs Reserved instances - when to use each
- What is egress traffic (and why it's often the hidden cost driver)
- Public vs private subnet routing - when do you use each
- AWS Organizations and SCPs - can an SCP have both Allow and Deny statements
- IAM Roles vs IAM Policies
- Explain VPC, Subnets, NAT Gateway, and Internet Gateway together
- Why choose one AWS service over a comparable alternative (general judgment - be ready to justify a real choice you made)
- An EC2 instance suddenly becomes unreachable - what do you check first
- A developer can't access an S3 bucket - what do you check (trust policy, permission policy, bucket policy, explicit Deny)
- AWS bill jumps 40% (or 65% vs. 25% traffic growth) - how do you find the root cause and fix it → Cost Explorer/CUR, tagging, right-sizing, spot, autoscaling tuning, storage lifecycle
- Lambda is failing internally - what are your debugging steps
- What's CloudFront's role in an architecture
- Design disaster recovery for RDS
- Design multi-region resilience for a business-critical platform with minimal downtime/data loss - broader than the RDS-specific DR question above: application-tier failover architecture, cross-region data replication strategy and its consistency tradeoffs, and how you'd actually *test* the failover (scheduled game-days, not just a design on paper)
- How would you design a highly available and fault-tolerant application on AWS in general (multi-AZ by default, stateless app tier behind a load balancer, managed multi-AZ data tier, health-check-driven failover, no single-instance dependency anywhere)
- Design a reliable, scalable system to process millions of URLs a day (crawl → process → store) - focused on the infra/DevOps decisions: a queue to decouple crawl-rate from processing-rate (SQS), horizontally scalable stateless workers behind autoscaling tied to queue depth (not CPU), idempotent processing so a retried message doesn't duplicate work, and tiered storage for the output (hot/cold based on access pattern)
- Debugging latency issues in an EKS cluster
- An EC2 instance is healthy (passes its own health check) but the application on it is unreachable from outside - troubleshoot in order: security group allows the port from the caller's source → if behind an ELB/ALB, the target group's health-check status specifically, not just instance status → NACL at the subnet level (stateless, both directions) → the instance's own OS-level firewall if one runs → confirm the app is bound to `0.0.0.0` and not just `localhost` (a healthy process listening only on loopback is unreachable externally regardless of every network layer being correct)
- Load balancer is receiving traffic but requests aren't reaching backend servers - walk the path: target health (healthy/unhealthy) → health-check config (path/port/protocol/expected response code) → `curl localhost:<port>` on the instance itself to rule out the app → security group/firewall allowing the LB's traffic on that port

## 4. Terraform / Infrastructure as Code

*(the most-repeated topic across every source list - expect deep, scenario-style questions, not definitions)*

- What happens when two engineers run `terraform apply` simultaneously? → state locking (S3 + DynamoDB mechanics specifically)
- `terraform plan` shows no changes, but `apply` still modifies resources - explain a real scenario this happens (e.g. a provider-side default drifting, a data source returning fresh values, `-refresh=false` masking drift)
- Someone manually deletes/changes a resource outside Terraform - what happens on the next `plan`/`apply`? Same question with a specific scare scenario: **`plan` unexpectedly shows a production RDS instance will be destroyed, though you didn't touch RDS** - how do you investigate before running `apply`
- How do you design Terraform for 100+ microservices / hundreds of teams? → module boundaries, workspace vs. directory-per-env, remote state data sources between layers
- Terraform state is huge (200MB) and `plan` takes 12 minutes - how do you fix it? → state splitting by boundary, smaller modules, remote backend performance, `-target` as a last resort not a habit
- What happens if `terraform apply` fails halfway (a partial apply)? How do you recover safely?
- Three environments (dev/staging/prod) have all drifted from Terraform state, and prod is live - how do you fix it without breaking anything? → `terraform plan` per environment to see the actual drift, audit *why* each diverged before touching anything, `terraform state mv` for anything renamed rather than destroy/recreate, targeted `apply`s scoped to just the drifted resources rather than a blanket apply. **Never run `terraform force-unlock` in production unless you are absolutely certain no other process holds the lock** - force-unlocking a lock that's actually held risks two concurrent applies corrupting state, which is the exact failure the lock exists to prevent
- How do you recover deleted or corrupted state? (versioned S3 backend, `terraform state pull`/`push`, importing back in)
- Workspaces vs. separate state files - and why workspaces get dangerous at large-org scale (shared code path hides environment-specific drift risk)
- `terraform plan` shows **no drift**, but you know the infrastructure was actually changed manually - what now? → a "no drift" result only means Terraform sees no difference between what it manages and what it can see, not that nothing changed: the manual change may have touched an attribute the resource's schema doesn't track (write-only/computed fields Terraform never compares), the changed resource may not be in Terraform's state at all (created fully out-of-band), or state may simply be stale from before the change - `terraform plan -refresh-only` forces a real comparison rather than trusting a stale "no changes"
- Managing state across multiple AWS accounts/regions
- What happens when you rename a Terraform resource? → without `moved` blocks or `state mv`, Terraform destroys and recreates; with them, it doesn't
- `count` vs `for_each` - and why switching between them can destroy/recreate every resource (index-based vs. key-based identity)
- `depends_on` vs. implicit dependency - when does Terraform get the ordering wrong on its own
- How do you prevent accidental production destruction? → `prevent_destroy` lifecycle, restricting `Delete*` IAM actions to a small admin set, mandatory plan review in CI before apply
- How do you safely upgrade Terraform providers? → pin versions, upgrade in a lower environment first, read the changelog for breaking schema changes
- How do you manage secrets in Terraform without exposing them in the state file? → state itself isn't a secrets store even encrypted; use a secrets manager/Vault data source and mark outputs `sensitive`, encrypt state at rest regardless
- How do you refactor a Terraform codebase without destroying production resources? → `moved` blocks / `terraform state mv`, plan-diff review before every step
- Multiple modules reference the same underlying resource - what breaks (two modules both trying to manage/import it, drift between them), and how do you design ownership boundaries instead (one module owns a resource; others consume it via a `data` source or remote-state output, never a second `resource` block for the same object)
- How do you design Terraform modules to be reusable across teams without becoming tightly coupled to one team's specific environment assumptions (parameterize via variables, avoid hardcoded account/region/naming assumptions, version-pin the module source)
- A large `terraform apply` starts hitting cloud-provider API rate-limiting/throttling - how do you handle it? → provider-level retry/backoff config, tuning `-parallelism` down, or splitting the apply across smaller batches
- How do provider version mismatches break production, and how do you prevent it?
- Describe a real incident caused by Terraform state corruption - how did you fix it? (if you don't have one, say so honestly and describe how you'd handle it)
- Design a production Terraform CI/CD pipeline (plan on PR, human-approved apply, drift-detection schedule)
- How do you achieve zero-downtime infrastructure changes with Terraform specifically (vs. application-level zero-downtime)?
- Sample a Terraform provider block and a registry/module source block; explain the Terraform "lifecycle" block's purpose
- **State security hardening** (this was handed to you as a reference checklist, not a question - know the shape of the answer): dedicated least-privilege IAM role for Terraform, no long-lived keys - one concrete implementation is a dedicated Terraform/jump-server EC2 instance in a private subnet with that role attached via an IAM instance profile, so only that host (and whoever's authorized to reach it) ever runs `apply`, with temporary credentials instead of stored keys. Plus: state bucket private + versioned + encrypted (SSE-KMS) + TLS-enforced + Block Public Access on, lifecycle rules to expire old noncurrent versions, `Delete*` restricted to a small admin set (MFA for destructive ops), cross-region replication for critical environments. Note: **S3 MFA Delete cannot coexist with S3 Lifecycle management** - if asked, don't recommend both together.

## 5. CI/CD & Release Engineering

- Explain your CI/CD pipeline end to end, from commit to production
- How do you implement zero-downtime deployments (distinct from Terraform's version - this is the app/orchestration layer)
- A pipeline takes 18-25 minutes - how do you optimize it? → BuildKit layer caching, parallel/selective rebuilds (only affected images), test parallelization
- How do you implement rollback if a deployment fails?
- How do you manage secrets securely in a CI/CD pipeline? (never in plaintext env vars in the pipeline config; OIDC federation over long-lived cloud keys - a strong answer here names a real CI/CD pipeline you built with OIDC-based cloud auth, not just the concept)
- A Jenkins pipeline succeeds, but the deployed app doesn't work in production - where do you start? (pipeline success ≠ deployment health; check post-deploy health checks/smoke tests)
- CI/CD can't connect to GitLab/GitHub - what do you check (network egress, token expiry, webhook config)
- A CI/CD pipeline has been stable for months and suddenly starts failing with no pipeline changes on your side - how do you isolate the root cause? → something external moved: an expired/rotated credential or token, a base image or runner update upstream, a third-party action/plugin version that auto-updated, a dependency that published a breaking release without a pinned version - diff what changed in the *environment*, not the pipeline config, since the pipeline config is the one thing you know didn't change
- A Docker container works locally but fails in production - how do you investigate? (env var/config parity, base image mismatch, resource limits present in prod but not local, missing secrets)
- Git branching strategy - explain the one you use and why
- A developer accidentally pushes credentials to GitHub - what do you do immediately? (rotate the credential first - assume it's compromised the moment it's pushed, even after a history rewrite; then remove from history, audit for usage)
- Design GitOps for multiple teams with independent releases → ArgoCD App-of-Apps / ApplicationSets pattern, controlled infra repo boundaries
- Design a highly available CI/CD **platform** itself (the tooling, not the apps it deploys) → no single point of failure in the build system: multiple runners/agents rather than one build server, a runner pool that absorbs one runner's failure without blocking every team, the control plane's own config backed up and version-controlled, pipeline secrets managed the same way as any other production secret (OIDC federation, not static keys sitting in the CI system)
- Where do you integrate security scanning into a CI/CD pipeline (SAST/dependency scanning at build, image scanning before push, policy gate before deploy), and what actually ensures only tested and secure code reaches production (mandatory checks, required approvals - not just "we have a scanner")

## 6. Observability, SLOs & Incident Response

*(a dedicated SRE-focused source list here was unusually deep - treat this as the highest-signal bucket for most Senior SRE/DevOps roles)*

- How would you design SLIs and SLOs for a business-critical service?
- Who should decide an SLO - can an SRE unilaterally decide a service needs 99.9%? (no - it's a product/business tradeoff conversation SRE informs, not owns unilaterally)
- SLO is 99% - calculate the error budget (know the monthly-minutes math cold: 99% ≈ 7.2 hrs/month, 99.9% ≈ 43.2 min, 99.95% ≈ 21.6 min, 99.99% ≈ 4.3 min)
- How do you use the error budget to make deployment decisions (freeze/slow-down policy when budget is nearly exhausted)
- Product keeps pushing features - how do you get them to delay/control a deployment for reliability reasons? (make the budget visible and pre-agreed *before* the conflict, not argued live)
- Would you actually stop a production deployment because the error budget is being consumed? (yes, if the policy says so and it was agreed in advance - the point of the policy is removing this from being a personality argument)
- Describe a real instance of killing or delaying a planned feature specifically to protect an error budget (if you don't have one, say so honestly rather than inventing it - error budgets are a planning tool, not a bragging metric, and the interviewer is checking whether you've actually enforced the policy or only ever talked about it)
- SLI vs SLO vs SLA vs Error Budget - crisp definitions of all four
- A brand-new service has zero observability - how do you design it from scratch? (start with RED/USE, add traces, then dashboards, then alerts - in that order, not alerts first)
- What do you weigh when designing metrics, logs, traces, dashboards, and alerts together
- Why does metric cardinality matter, and what happens if developers want `user_id`/`request_id` as a metric label? (cardinality explosion breaks the metrics backend cost/perf; push those into logs/traces instead, keep labels bounded)
- RED metrics (Rate/Errors/Duration) vs. the USE method (Utilization/Saturation/Errors) vs. the Four Golden Signals - how they relate, when each fits (RED for request-driven services, USE for resources)
- P99 latency triples, error rate is flat, no recent deploy - how do you troubleshoot? (rules out the deploy as a variable; check for a slow downstream dependency, GC pauses, noisy-neighbor contention, or a traffic-mix shift toward a slower code path)
- How do you isolate whether a latency problem is app / Kubernetes / database / network / downstream dependency? (distributed tracing to find which hop's span grew; if no tracing, bisect layer by layer)
- Why can latency increase while the error rate stays completely flat? (slow-but-successful requests don't error - a classic blind spot if you only alert on error rate)
- Pod count is spiking and latency is up 400%, but CloudWatch shows nothing obviously wrong - walk your response → work the signal chain in order: **metrics → traces → logs → events**, not logs-first. Metrics tell you *that* something's wrong and roughly where; distributed tracing (e.g. Jaeger) tells you *which hop* in the request path is actually slow or looping; only then do logs/events on that specific narrowed-down component tell you *why*. (A real answer to this exact scenario: a misconfigured HPA scaling on the wrong metric, causing runaway pod count without fixing the actual latency driver.)
- The service is "slow" according to the CEO/a stakeholder - how do you prove or disprove that, and where do you look first? → a single dashboard number isn't proof by itself: check P50 *and* P99 (not just an average, which hides a bimodal distribution), run a synthetic check from the same vantage point the stakeholder is describing, and only pull a packet capture or read the actual code path if the metrics/traces don't explain it
- A service meets its availability SLO but users report poor latency/UX - how would you redesign the SLI/SLO framework and use error budgets to redirect engineering priority? (the existing SLO was measuring the wrong thing - availability alone doesn't capture experience; add a latency-percentile SLI, set a separate SLO for it, and let its own error budget compete for priority against feature work the same way the availability budget does)
- Engineering teams have fragmented logs/metrics/traces across many teams, making diagnosis slow - how would you establish a single enterprise-wide observability strategy and improve MTTD/MTTR? (standardize on one instrumentation approach/library across teams, a shared trace-correlation ID propagated end-to-end, a common dashboard/alerting convention - the goal is one story across team boundaries, not five different local ones)
- Write a blameless postmortem for an outage you caused (or would write for a real one) - timeline, contributing factors, and **one** action item you'd own end-to-end with a deadline → distinguish proximate cause (what directly triggered it) from root cause (the systemic gap that let it happen); a postmortem written for the next on-call engineer, not for a stage demo, is specific about both; one owned action item with a deadline is worth more than seven vague ones spread across a team
- Walk through a real Sev1/P1 across all four phases: detection, mitigation, resolution, prevention
- The same incident keeps recurring even after RCA and preventive action - what now? (the RCA likely found a symptom, not the root cause, or the fix wasn't actually verified under load)
- Mitigation vs. resolution - the difference (stop the bleeding vs. fix the actual cause)
- How do you measure MTTD and MTTR, and what should MTTR actually represent? (time-to-detect vs. time-to-restore service, not time-to-permanent-fix - a rollback that restores service is a valid MTTR stop even before the code fix ships)
- Realistically, can you expect a developer to fix a genuine code defect within 60-90 minutes during an incident? (no - the honest answer is mitigate first via rollback/feature-flag, let the real fix follow at normal pace with review)
- The SRE team is drowning in alerts - what do you do? (audit for actionability, cut anything that's never actioned, move noisy signals to dashboards instead of pages, tune thresholds/burn-rate windows)
- Thousands of alerts fire at once during a live incident - how do you find the real problem? (distinct from the alert-fatigue-hygiene question below - this is acute triage) → correlate rather than triage individually: group by affected service/dependency graph and look for the **earliest** alert, since one upstream failure commonly cascades into hundreds of derived alerts everywhere that depends on it - the root cause is usually the first domino, not the loudest one
- Logs look completely normal, but users are reporting real failures - what next? → points at a layer that never reaches your application logs: a CDN/edge cache serving stale content, a DNS issue routing some users elsewhere, a client-side failure that never generates a server-side log line, or log sampling/level filtering silently dropping the exact errors that matter - reproduce from the user's actual vantage point rather than trusting "logs are clean" as proof nothing failed
- How do you perform Root Cause Analysis, as a repeatable method? → build a timeline first (what changed, correlated against when symptoms started), then separate proximate cause from root cause via a "5 Whys"-style chain that keeps going past the first plausible answer, then verify the fix under the actual failure conditions - not just that the symptom disappeared, which is why a bad RCA lets the same incident recur
- What makes a good production paging alert? (actionable, tied to user impact, has a clear runbook, doesn't fire on transient blips)
- SLO-based alerting without alert fatigue → error-budget burn-rate alerts with multiple time windows (fast burn = page now, slow burn = ticket)
- Difference between logs, metrics, and traces
- General: explain your monitoring and alerting strategy; describe a real production issue you resolved
- A microservices platform experiences cascading failures when one dependency becomes unhealthy - how do you design resilience with timeouts, retries, circuit breakers, bulkheads and graceful degradation (not stretch - buildable directly from your production RCA reasoning even without a named framework like Hystrix/resilience4j; ground it in a real dependency-failure scenario you've handled)
- How would you present SRE KPIs to a CTO/exec audience (availability, incident trends, MTTD/MTTR, change failure rate, SLO compliance) to demonstrate reliability and engineering effectiveness - a communication/framing question as much as a technical one

## 7. Security & DevSecOps

- How do you secure container images before deployment? (scanning in CI, minimal base images, signed images)
- An image passed vulnerability scans but still got exploited - what did you miss? (scanning ≠ runtime security: RBAC over-permissioning, missing seccomp/AppArmor profile, writable root filesystem, secrets baked into env vars or layers)
- How do you implement RBAC in Kubernetes (least-privilege Roles/ClusterRoles, avoid wildcard verbs/resources)
- How do you manage secrets in production generally, and specifically **across 50+ services** without every service needing direct Vault access? (Vault Agent Injector or the External Secrets Operator, Kubernetes auth method for short-lived dynamic secrets rather than distributing static tokens)
- How do you design a disaster recovery strategy
- How do you reduce cloud cost without affecting availability (this overlaps cost-optimization - see below)
- General: how do you secure an AWS environment; what tools/services for security and compliance (a strong answer here names the specific posture-scanning/VAPT tools and a real PII-governance or compliance project you've run)

## 8. Cost Optimization / FinOps

- Reduce cloud cost by 40% without impacting performance - where do you start? (right-sizing from actual utilization data, spot instances for interruptible workloads, autoscaling tuning, cost observability/tagging before optimizing blindly)
- Cloud costs rose 65% while traffic grew only 25% - identify the drivers and fix it with FinOps practices (rightsizing, workload scheduling, storage tiering, autoscaling, governance) without compromising reliability
- Reduce a specific service's cost (e.g. a $42K/month bill) by 30% without touching application code or breaking its SLOs - where do you look? → right-sizing from real utilization data, spot/preemptible for interruptible workloads, storage tiering, log/index retention tuning, moving batch work off-peak. (The wrong move interviewers are listening for you to avoid: "rewrite it in a faster language" - that's not a DevOps/cost lever, and it risks the SLO you were told not to break.)
- How would you use Kubernetes resource requests/limits themselves as a cost-control lever? (right-sized requests let the scheduler bin-pack more workloads per node instead of over-reserving capacity nobody uses; limits prevent one noisy pod from forcing over-provisioned nodes to compensate)
- How would you optimize costs differently across Dev, QA, UAT, and Production? (non-prod: scheduled shutdown outside working hours, spot/preemptible by default, smaller instance sizes, shorter retention; prod: reliability and headroom take priority over squeezing the last dollar - different risk tolerance, different levers)
- **What makes this bucket land in an interview:** a specific before/after percentage and dollar figure from something you actually did - interviewers remember a concrete number far more than a general claim about cost-consciousness.

## 9. Scripting / Hands-on Practical (expect to type live)

- Extract and use only a specific word from a file efficiently (`awk '{print $N}'`, `grep -o`, `cut`)
- Delete a specific name/line from a file and move it to another file via script (`grep -v "$name" file > tmp && mv tmp file`, or `sed -i "/$name/d"` plus appending the removed line elsewhere first)
- List/display files in reverse order in Linux (`ls -tr`, `ls -r`, `ls -lt | tac`)
- Write an Ansible playbook that prints "Hello World" on 2 different servers (a `hosts` inventory with two entries, a play with `debug: msg="Hello World"`, or `command: echo "Hello World"`)
- Write a Terraform script creating an S3 bucket (`resource "aws_s3_bucket"`, provider block, a sensible name/tag set)
- What have you used Ansible for; walk through a playbook you wrote; how does Ansible reduce deployment time (idempotent, agentless, declarative); what is Jinja2 and why does Ansible use it (templating for dynamic config values); what is Ansible Tower/AWX (the web UI/RBAC/scheduling layer over core Ansible - **acknowledge-bridge if you haven't used it**: you've used Ansible itself, not Tower specifically)
- Write a script/describe the approach to parse a massive, live-streaming production log file, find all HTTP 5xx errors, aggregate them by API endpoint, and print the top 3 failing endpoints if total errors exceed a threshold → stream line-by-line (never load the whole file into memory - `tail -F`-style or a generator), regex/split out status code and path, `collections.Counter` to aggregate, sort and slice top 3, gate the print on the threshold
- Write a safe retry loop for a backend API call that returns a transient `503` → exponential backoff with jitter, a capped max-retry count, and awareness that hammering a struggling service faster just makes it worse (this is the "why" the earlier retry-script exercise was missing - avoiding overload, not just retrying)
- Write a script to free disk space: find `.log` files in `/var/log` over 1GB and older than 7 days, compress them, and safely clear the original file's content **without deleting the file** → `gzip` a copy, then truncate the original in place (`> file` or `truncate -s 0 file`) rather than `rm`, because deleting a file an active process still has open by file descriptor doesn't free the disk space until that process closes it, and can break the app if it's still writing to that path
- Detect a zombie-process loop, or find which process is leaking memory when `top` only shows high global memory without isolating the culprit → walk `/proc/<pid>/status` or `smem`/`ps aux --sort=-%mem` sampled over time to catch RSS growth per PID (zombie processes hold no memory themselves - they're already terminated, just unreaped; a genuine leak shows as one PID's RSS climbing steadily, not as zombie count climbing)

## 10. Helm

<!-- If your real experience is "Kubernetes manifests organized in a Helm-like structure"
rather than actual hands-on `helm install/upgrade/rollback` and Chart.yaml/values.yaml
templating, be honest about which one it is before an interview. If it's manifests-only, treat
everything below as acknowledge-bridge territory the same way Ansible is treated in section 9 -
bridge from real Kubernetes manifest experience (Deployments/Services/ConfigMaps/Secrets/
Ingress) rather than claiming Helm depth you don't have. -->

- What is Helm, and why use it instead of raw `kubectl apply` on plain manifests?
- Helm 2 vs Helm 3 - what changed, and why was Tiller removed? (Tiller was a cluster-side
  component with broad RBAC permissions - a security liability; Helm 3 talks to the
  Kubernetes API directly using the user's own credentials)
- Chart structure: `Chart.yaml` (metadata), `values.yaml` (default config), `templates/`
  (the actual manifest templates), `_helpers.tpl` (reusable template snippets/functions)
- Managing Dev/QA/Prod with one chart → per-environment `values-<env>.yaml` files overriding
  the chart's defaults at install/upgrade time, not separate charts
- Helm templating: `if`/`else`, `range` (loops), `with` (scoping), `include` (calling a
  named template), built-in functions - the point being config-driven manifests, not
  hardcoded YAML per environment
- `helm install` vs `helm upgrade` vs `helm rollback` vs `helm uninstall` - install creates a
  new release, upgrade applies a new revision to an existing release, rollback reverts to a
  prior revision, uninstall removes the release (and by default its history)
- How do you troubleshoot a failed Helm deployment, or pods in `CrashLoopBackOff` right after
  a `helm upgrade`? → `helm history` → compare revisions → check the values actually applied
  → render the templates locally (`helm template`/`helm get manifest`) to see what was
  actually sent to the cluster → check the resulting ConfigMaps/Secrets → `kubectl describe`
  / logs on the pod → fix config or `helm rollback` if the fix isn't quick → redeploy and
  validate
- Helm dependency management - how `Chart.yaml`'s `dependencies` block and the generated
  `Chart.lock` pin sub-chart versions, similar in spirit to a lockfile in any package manager
- Integrating Helm into a CI/CD pipeline (Jenkins/GitHub Actions) - lint/template-render as a
  pre-deploy check, `helm upgrade --install` with environment-specific values files, gated by
  the same approval/test steps as any other deploy
- Managing secrets with Helm - values files aren't a secrets store; use a Kubernetes Secret
  populated by an external mechanism (Vault, External Secrets Operator, sealed-secrets) rather
  than plaintext values committed to Git
- Can 20 microservices be deployed as one Helm release? → technically yes (one umbrella chart
  with sub-charts), but it couples unrelated services' release lifecycles together; the
  stronger default is one chart per service, each independently releasable, with a shared
  library chart for common templates if there's real duplication

## Adjacent-stack gap: Java build tooling (flag before using if it's not your stack)

<!-- These appeared in one collected question set (Maven/pom.xml, JAR vs WAR, Tomcat
deployment). If Java/Maven/JVM tooling isn't your stack, use the acknowledge-bridge pattern
explicitly rather than improvising an answer that sounds more confident than your real
experience - e.g. "I haven't run a Maven build pipeline myself; my CI/CD experience is
Python/Node-style packaging and Docker builds, but the lifecycle model (dependency resolution →
compile → test → package → deploy) is the same shape I've automated for other ecosystems, and
I'd expect to be productive with `pom.xml` quickly." Never claim a specific Maven/Tomcat project
that isn't real. -->

- Explain the Maven build process end-to-end; how do you execute a `pom.xml`
- Difference between a JAR and a WAR file, and how each gets deployed to a Tomcat server

## 11. Distributed Systems: Data Consistency & Event-Driven Architecture

<!-- This is system-design/database-architecture judgment, one layer up from pure
infra/DevOps - a legitimate senior/staff system-design probe, but schema and application-level
ledger design itself sits outside the core DevOps remit if your actual role is infrastructure,
not the transformation/business logic. If you've built a real CDC (Change Data Capture)
pipeline, the Outbox/CDC half of this section has a genuinely strong real anchor in that
experience - reliably streaming committed changes out of a database. The lock-contention/
ledger-schema half is sound reasoning to have ready even if you haven't personally implemented
it - frame it as "here's how I'd reason about it," not as a shipped project, the same discipline
as the EKS upgrade runbook flag elsewhere in this file. -->

- 8 microservices run concurrent CRUD workers against one shared database (e.g. `UPDATE accounts SET balance = balance - X`) - what breaks at real production scale, and why does it look fine in staging at low request rates? → concurrent row-level updates take exclusive write locks; two workers each holding a lock the other needs produces a deadlock, lock-wait timeouts saturate the connection pool, failed workers retry with backoff, and the retry storm becomes a thundering herd that pins DB CPU on lock arbitration rather than real work - throughput can go to near-zero under contention even though no single query is slow in isolation
- Why does an append-only, immutable event log (e.g. a double-entry ledger - `INSERT` DEBIT/CREDIT journal rows, never `UPDATE` a balance column) scale better under concurrency than mutating a balance row in place? → inserts don't require an exclusive lock on an existing row the way updates do, which removes the row-level lock contention entirely; the "balance" becomes a derived/materialized value computed from the journal rather than a mutable field multiple workers fight over
- Is a Dead Letter Queue enough as the reliability safety net for async workers, or does it just look like one? → a DLQ is a transport mechanism, not a system of record, and it fails in several specific ways worth naming: a broker node can crash before a message is durably replicated, so the message is gone before it ever reaches the DLQ; a worker can write to the DB successfully but the ACK to the broker gets lost in a network partition, causing redelivery and duplicate processing (a real double-spend risk in a financial context); messages can silently expire under severe consumer lag (TTL); and a DLQ gives you zero auditability - you cannot reconstruct an accounting proof from an error queue
- How do you guarantee an event is reliably published if and only if the business transaction that produced it actually committed? → the **Transactional Outbox Pattern**: write the business row and an outbox/event row in the same local ACID transaction (so they either both commit or both roll back - no dual-write inconsistency), then a separate asynchronous process (commonly Change Data Capture, e.g. Debezium reading the database's write-ahead log/binlog) streams the outbox rows out to the message broker. Because the source of truth is the committed database transaction, not the broker, the broker can be down for hours and zero transactions are lost - it just catches up once it recovers. **If you've built any kind of CDC pipeline, this is the same underlying mechanism** - reliably streaming committed database changes to a downstream consumer without depending on that consumer being up at write time. Tie this answer to your own real CDC/event-pipeline experience if you have it; if not, the reasoning stands on its own as system-design judgment

## 12. Live-coding & Manifest-Writing Practicals

Distinct skill from conceptual troubleshooting above - these expect an actual written artifact
(code, YAML, HCL, a Dockerfile), typed live or on a shared screen. Unlike the rest of this
file, each item here includes a real worked example, since "write it" questions need to be
practiced by actually writing it, not just discussed.

### Basic coding (Python/Bash warm-ups - not LeetCode-style DSA, but expect one trivial one)
- FizzBuzz-style warm-up: print numbers 1-100, "Fizz" for multiples of 3, "Buzz" for 5, "FizzBuzz" for both
  ```python
  for i in range(1, 101):
      print("FizzBuzz" if i % 15 == 0 else "Fizz" if i % 3 == 0 else "Buzz" if i % 5 == 0 else i)
  ```
- Check a list for duplicates / find the second-largest number - classic warm-ups, know a clean one-liner (`len(lst) != len(set(lst))`; `sorted(set(lst))[-2]`)
- Parse a JSON/YAML config and validate required keys exist before using it (a realistic DevOps warm-up, not abstract DSA):
  ```python
  import json
  def load_config(path, required_keys):
      with open(path) as f:
          cfg = json.load(f)
      missing = [k for k in required_keys if k not in cfg]
      if missing:
          raise ValueError(f"Missing required config keys: {missing}")
      return cfg
  ```
- Write an idempotent function (safe to call twice with the same result) - e.g. "ensure a directory exists" (`os.makedirs(path, exist_ok=True)`) as the canonical example of idempotency in infra code, tying directly to why Terraform/Ansible are built around the same principle

### Linux command-line practicals (beyond the conceptual troubleshooting in section 1)
- File permissions: explain `chmod 750`, when to use SUID/SGID/sticky bit, and a real scenario each solves (sticky bit on `/tmp` so users can't delete each other's files)
- Process management: `kill` (SIGTERM) vs `kill -9` (SIGKILL) - why you always try SIGTERM first (lets the process clean up/flush); zombie vs. orphan process (zombie = terminated but unreaped by parent; orphan = parent died but child still running, gets reparented to init/PID 1)
- A cron job works when run manually but fails/does nothing when run by cron - why? → cron runs with a minimal environment (no interactive shell's `PATH`, no sourced `.bashrc`), so always use absolute paths and explicitly set/source any environment the script needs inside the cron job itself, not rely on the shell environment
- `find`/`xargs` one-liners: find and delete files older than 7 days (`find /var/log -mtime +7 -name "*.log" -delete`); find all files containing a string across a directory tree (`grep -rl "pattern" /path`)
- `awk`/`sed` beyond section 1's basics: sum a column (`awk '{sum+=$3} END {print sum}'`), in-place replace across multiple files (`sed -i 's/old/new/g' *.conf`)

### Terraform manifest-writing (beyond the S3-bucket exercise in section 9)
- Write an EC2 instance with an attached security group:
  ```hcl
  resource "aws_security_group" "web" {
    name_prefix = "web-sg-"
    vpc_id      = var.vpc_id
    ingress {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  resource "aws_instance" "web" {
    ami                    = var.ami_id
    instance_type          = "t3.small"
    subnet_id              = var.subnet_id
    vpc_security_group_ids = [aws_security_group.web.id]
    tags = { Name = "web", Environment = var.environment }
  }
  ```
- Write a least-privilege IAM role + policy for a specific action (e.g. read-only S3 access):
  ```hcl
  resource "aws_iam_role" "app_role" {
    name = "app-s3-readonly"
    assume_role_policy = jsonencode({
      Version = "2012-10-17"
      Statement = [{
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "ec2.amazonaws.com" }
      }]
    })
  }

  resource "aws_iam_role_policy" "s3_read" {
    role = aws_iam_role.app_role.id
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [{
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
      }]
    })
  }
  ```
- Write a `for_each` loop creating multiple similar resources from a map (the modern alternative to `count` for named resources):
  ```hcl
  variable "buckets" {
    default = { logs = "app-logs", backups = "app-backups" }
  }
  resource "aws_s3_bucket" "this" {
    for_each = var.buckets
    bucket   = each.value
    tags     = { Purpose = each.key }
  }
  ```
- Write a module call with variables/outputs (shows understanding of module composition, not just resource blocks):
  ```hcl
  module "vpc" {
    source   = "./modules/vpc"
    cidr_block = "10.0.0.0/16"
    environment = var.environment
  }
  output "vpc_id" {
    value = module.vpc.vpc_id
  }
  ```

### Dockerfile-writing practicals
- Write a multi-stage Dockerfile for a Python app - separate build and runtime stages to keep the final image small:
  ```dockerfile
  FROM python:3.11-slim AS builder
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install --user --no-cache-dir -r requirements.txt

  FROM python:3.11-slim
  WORKDIR /app
  COPY --from=builder /root/.local /root/.local
  COPY . .
  ENV PATH=/root/.local/bin:$PATH
  RUN useradd -m appuser && chown -R appuser /app
  USER appuser
  ENTRYPOINT ["python", "app.py"]
  ```
  Points to narrate: multi-stage keeps build-only dependencies out of the final image; a
  non-root `USER` is a security baseline, not optional; pinning the base image tag (not
  `:latest`) makes builds reproducible.
- Given a "bad" Dockerfile (single stage, runs as root, `:latest` base, no `.dockerignore`,
  copies the whole build context including `.git`/secrets), identify and fix each issue -
  expect this as a live "spot the problems" exercise as much as a from-scratch write

### Kubernetes manifest-writing practicals
- Write a Deployment with resource requests/limits and both probes:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata: {name: web, labels: {app: web}}
  spec:
    replicas: 3
    selector: {matchLabels: {app: web}}
    template:
      metadata: {labels: {app: web}}
      spec:
        containers:
        - name: web
          image: myregistry/web:1.4.2
          ports: [{containerPort: 8080}]
          resources:
            requests: {cpu: "250m", memory: "256Mi"}
            limits: {cpu: "500m", memory: "512Mi"}
          readinessProbe:
            httpGet: {path: /healthz, port: 8080}
            initialDelaySeconds: 5
          livenessProbe:
            httpGet: {path: /healthz, port: 8080}
            initialDelaySeconds: 15
  ```
- Write a matching Service (selector must match the Deployment's pod labels, not its own):
  ```yaml
  apiVersion: v1
  kind: Service
  metadata: {name: web}
  spec:
    selector: {app: web}
    ports: [{port: 80, targetPort: 8080}]
    type: ClusterIP
  ```
- Write an Ingress with host/path routing:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata: {name: web-ingress}
  spec:
    rules:
    - host: app.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend: {service: {name: web, port: {number: 80}}}
  ```
- Write a ConfigMap and mount it two ways (env var vs. volume file) - know both patterns and
  when each fits (env vars for a few simple values, a mounted volume for a full config file the
  app reads at startup):
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata: {name: app-config}
  data:
    LOG_LEVEL: "info"
    config.yaml: |
      feature_flags:
        new_ui: true
  ```
- Write a PodDisruptionBudget and a NetworkPolicy (both covered conceptually in sections 2 and 7
  above - the "write it" version):
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata: {name: web-pdb}
  spec: {minAvailable: 2, selector: {matchLabels: {app: web}}}
  ---
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata: {name: web-allow-frontend}
  spec:
    podSelector: {matchLabels: {app: web}}
    ingress:
    - from: [{podSelector: {matchLabels: {app: frontend}}}]
      ports: [{protocol: TCP, port: 8080}]
  ```

## Stretch scenarios (above current documented experience - use acknowledge-bridge, don't invent)

### Amazon-style flash-sale scale round `(stretch)`
- Design auto-scaling for a 40x traffic spike in the first 10 minutes of a sale without over-provisioning the other 364 days
- Checkout pods healthy, but order-confirmation emails silently back up 20 minutes - first 3 diagnostic steps
- Use **Istio traffic shifting** to canary a payment-service update with automatic rollback on error rate > 0.5% `(stretch - most candidates won't have hands-on service-mesh experience; bridge via your Kubernetes rollout/canary reasoning and any API-gateway work you've done, the closest adjacent precedent)`
- Pod anti-affinity so a single-AZ failure doesn't take out checkout during peak
- Detect an HPA scaling on stale metrics before it silently under-provisions during a spike
- Cart abandonment spikes 15% for one payment method, one region, during peak - CPU/memory/pods all green - triage path
- Inventory-sync **Kafka** topic lags 90s during a flash sale; producers healthy; consumers show 0% CPU - what's actually happening `(if you have real Kafka experience, use it - but regardless, this specific "0% CPU with growing lag" symptom points at a stuck/rebalancing consumer group or a poison-pill message, worth reasoning through explicitly rather than guessing scale-specific causes)`
- Redis-backed session store shows tail-latency spikes only during checkout, not browsing - isolate the bottleneck in under 10 minutes
- Get a business team to treat "checkout latency" as seriously as "site is up or down"
- Ship multi-region failover for the next sale in 10 days with **no DNS-based routing allowed** `(stretch)`
- Chaos-test a payment pipeline without risking a single real transaction `(stretch - most candidates won't have formal chaos-engineering exposure; bridge via any zero-downtime migration discipline you have - staged validation before cutover is the same underlying skill)`

### Hyperscaler-style service-mesh & multi-cloud round `(stretch - most candidates won't have hands-on experience here; bridge, don't invent)`
- Fine-grained service discovery across 1,000+ microservices using Envoy or Istio `(stretch - bridge via your Kubernetes Service/DNS discovery reasoning and any API-gateway work you've done, the closest adjacent precedent)`
- Using eBPF + Cilium to enforce network security policy at runtime, vs. a traditional CNI `(stretch - no documented eBPF exposure; bridge via your Kubernetes NetworkPolicy/RBAC/least-privilege reasoning - same goal, different enforcement layer)`
- Cross-cloud routing, IAM, and secret-syncing in a multi-cloud (not just multi-account) setup `(stretch - AWS-only track record; be direct about this one rather than stretching it)`
- Vetting a custom AMI at the kernel/runtime level before it reaches production `(plausible bridge if you've done any AMI-level work - e.g. an architecture migration involving custom node-group AMIs; kernel-level vetting specifically is a step up from that, so be honest about where your real experience stops)`
- RCA for an Envoy config rollout that silently broke mTLS between edge and mesh `(stretch - bridge via your general RCA discipline and any TLS/IAM-authentication work you've done on databases/caches)`
- A sidecar-based caching layer causes a tail-latency spike - debug path `(stretch - bridge via any cache-latency and API-gateway reasoning you have)`
- 504s on a service despite a healthy load balancer and healthy mesh sidecars - triage `(builds on the "pods healthy but 502/503" reasoning above - same isolate-the-layer discipline, just with a mesh hop added to the chain)`

### Principal/Staff-level org design `(stretch - good practice if you're aiming for a lead/staff step-up; not a realistic bar for a standard Senior IC round)`
- Architect a platform for 5,000+ engineers, 2,000+ production workloads, reliability/scale/DR/observability
- Availability fell from 99.99% to 99.5% as complexity grew - restore it using SLOs/error budgets/observability/capacity planning
- Standardize CI/CD across hundreds of inconsistent pipelines - IaC, GitOps, policy, self-service platform
- Migrate hundreds of apps to Kubernetes across multiple clouds - secure, scalable platform design
- Design the incident-management framework end to end: detection, alerting, escalation, ownership, comms, automated remediation, postmortems
- A first 100-day roadmap inheriting a troubled global DevOps/SRE org (outages, weak observability, high cost, fragmented CI/CD, K8s complexity, toil, low deploy confidence) - sometimes framed as a 12-month transformation roadmap instead; same answer, just paced across more milestones
- How do you prove the ROI of infrastructure modernization work to non-technical executives? (translate technical wins into the language they track - cost reduction in dollars, incident/downtime reduction in revenue-at-risk terms, deployment velocity in time-to-market - your own concrete cost/reliability numbers are the ready-made answer here)

## Personal-reflection warm-up questions (use your own real history, not a rehearsed script)
- What was the last production outage you handled, and what was the root cause?
- What deployment failure did you debug recently?
- Have you ever rolled back a release? Why?
- What was the last Terraform issue you fixed?
- How did you reduce infrastructure cost in your project?
- What monitoring alert did you personally configure?
- Which CI/CD failure took you the longest to debug?
- What scaling issue did you face in Kubernetes?
- Tell me one mistake you made in production and what you learned
- "If I join your project today, what would you improve first in the infrastructure?"
- Tell me about a time you proactively identified a problem or opportunity and took the initiative to address it without being asked - what was the outcome? (a distinct classic behavioral prompt from the "what would you improve" one above - this one wants something you *actually did* unprompted, not a hypothetical)
- Tell me about a production incident you owned where the root cause was **not** in the first place anyone looked (DNS, MTU, clock skew, a hot loop, a noisy neighbor) - what was it, and how did you actually find it? (this is specifically testing for a real, non-obvious war story - a surface-level answer here, e.g. "it was a typo," reads as junior depth under a senior title)

## Generic intro / resume-walkthrough questions
- Tell me about yourself and your experience
- How many years of Kubernetes experience do you have?
- What is your role in your current project? Is your work ticket/Jira-based? Do you build pipelines or mainly support deployments?
- What infrastructure have you built with Terraform, which AWS resources, have you used modules and why
- What is Infrastructure as Code and its benefits; purpose of the Terraform state file; what are Terraform providers; what happens after `terraform init`; can you use multiple providers; what are meta-arguments; what is "indexing" in Terraform (`count.index`/`each.key`)
