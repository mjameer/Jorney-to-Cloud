# Journey to Cloud: A Tale of Legacy Apps, Containers, and "Why Is This So Slow?"

In the fast-evolving world of tech, "modernize or perish" isn't just a slogan — it's a Monday morning standup topic. This is the story of how we took multiple Java applications that were happily living in the cloud, dragged them through Mavenization, Liberty migration, containerization, and finally dropped them into Azure Kubernetes Service (AKS)... only to discover the journey had a few plot twists we didn't see coming.

If I had to summarize six months of work in one sentence: **we Googled a lot of YAML errors.** Seriously — if you're planning a move to containers and Kubernetes, buckle up. It's a ride.

---

## The Typical Cloud Migration Phases

Most cloud migration strategies follow a well-known progression — think of it as the five stages of grief, but for your infrastructure:

1. **Assess** — "How bad is it, really?"
2. **Mobilize** — "Okay, it's bad. Let's plan."
3. **Migrate** — "Why did we agree to this?"
4. **Modernize** — "Oh wait, this is actually better."
5. **Optimize** — "We saved HOW much money?"

Here's how our journey mapped to these phases — with all the detours included.

---

## Phase 1: Mavenization — Taming the Build Chaos

Every great journey starts with a single `pom.xml`.

Our legacy applications had build processes that could charitably be described as "artisanal." Custom scripts, manual dependency management, and the kind of tribal knowledge that lives exclusively in one senior developer's head (and nowhere else).

**Mavenization** was our first move — standardizing every project to Apache Maven. This gave us:

- **Consistent project structure** across all applications (no more "where's the source folder?" archaeology)
- **Dependency management** via a central `pom.xml` instead of random JARs in a `/lib` folder
- **Reproducible builds** — because "it works on my machine" doesn't scale to production

Was it glamorous? No. Was it absolutely necessary before anything else could happen? 100%.

---

## Phase 2: Migration to Liberty — Breaking Up with Heavyweight App Servers

Next up: migrating from our legacy application server to **Open Liberty**.

Why Liberty? Think of it as going from a bulky SUV to a nimble sports car:

- **Lightweight runtime** — Liberty starts in seconds, not minutes. Your coffee won't even be ready before the server is up.
- **Microservices-friendly** — feature-based configuration means you only load what you actually use. No more dragging along the entire Java EE stack just to serve a REST endpoint.
- **Cloud-ready architecture** — Liberty was designed with containers in mind, making the next phase much smoother.

The migration involved rewriting `server.xml` configurations, updating deployment descriptors, and testing every endpoint. Not trivial — but the performance gains were absolutely worth it.

---

## Phase 3: Containerization — Docker Enters the Chat

With Liberty in place, it was time to **Dockerize everything**.

Docker containers gave us what every ops team dreams about: *consistency*. The same container image that runs in dev runs in staging runs in production. No more environment-specific surprises. (Well, fewer surprises. Let's be honest.)

**What this looked like in practice:**

```dockerfile
FROM open-liberty:kernel-slim-java17
COPY --chown=1001:0 server.xml /config/
COPY --chown=1001:0 target/*.war /config/apps/
RUN configure.sh
```

**Key challenges we tackled:**

- **Base image selection** — choosing the right Liberty image variant (full vs. kernel-slim) for each app's needs
- **Layer optimization** — ordering Dockerfile instructions to maximize build cache hits
- **Image scanning** — integrating vulnerability scanning (e.g., Trivy, Aqua) into the build pipeline to catch CVEs before they hit production
- **Developer buy-in** — convincing the team that learning Docker was an investment, not a tax

Pro tip: Your first `docker build` will fail. Your second one will too. By the fifteenth, you'll be a pro.

---

## Phase 4: Kubernetes Orchestration — Herding Containers at Scale

Docker gives you containers. **Kubernetes** gives you a way to manage hundreds of them without losing your mind.

We chose Kubernetes (specifically Azure Kubernetes Service — AKS) for orchestration because:

- **Horizontal scaling** — need more instances? Kubernetes spins them up. Traffic drops? It scales back down.
- **Self-healing** — if a pod crashes, K8s restarts it automatically. It's like having a tireless SRE who never sleeps.
- **Declarative configuration** — you describe the *desired state* in YAML, and Kubernetes figures out how to get there.
- **Rich ecosystem** — Helm charts, Ingress controllers, service meshes — the tooling ecosystem is massive.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-legacy-app-thats-now-cool
spec:
  replicas: 3
  selector:
    matchLabels:
      app: legacy-but-modern
  template:
    metadata:
      labels:
        app: legacy-but-modern
    spec:
      containers:
      - name: app
        image: myregistry.azurecr.io/my-app:latest
        ports:
        - containerPort: 9080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 9080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 9080
          initialDelaySeconds: 30
          periodSeconds: 10
```

Yes, that's a lot of YAML. Yes, you'll get used to it. No, you'll never love it.

A few things worth noting in the manifest above: the `resources` block prevents any single pod from eating your entire node's CPU and memory (ask me how we learned that one). The readiness and liveness probes ensure Kubernetes knows when your app is actually ready to serve traffic vs. when it's stuck in an infinite loop contemplating the meaning of `ClassNotFoundException`.

We managed our K8s manifests using **Helm charts**, which let us template configurations across environments (dev, staging, prod) without maintaining three separate sets of YAML files. For deployments, we adopted a **rolling update strategy** with `maxSurge: 1` and `maxUnavailable: 0` — ensuring zero-downtime deployments. No user should ever see a 503 because you pushed a config change.

---

## Phase 5: Externalizing Configurations — No More Hardcoded Secrets

Kubernetes brought one massive quality-of-life improvement: **externalized configuration** via ConfigMaps and Secrets.

Before K8s, application configurations (including database credentials, API keys, and service URLs) were often baked into property files inside the application. Changing a config meant rebuilding and redeploying.

With Kubernetes:

- **ConfigMaps** hold non-sensitive configuration data, mounted as files or environment variables
- **Secrets** store sensitive credentials with base64 encoding (and ideally backed by a proper secrets manager like Azure Key Vault or HashiCorp Vault)
- **Config changes don't require rebuilds** — update the ConfigMap, restart the pod, done
- **RBAC policies** control who can read/write secrets — because not every developer needs access to production database credentials

This was a game-changer for our security posture and deployment velocity.

> **A note on K8s Secrets:** Base64 encoding is **not encryption**. If you're storing anything truly sensitive, back your Secrets with an external secrets manager and enable encryption at rest in etcd. We learned this the hard way when a security audit politely pointed out that base64-decoding our "encrypted" credentials took exactly one terminal command.

---

## Phase 6: Database Migration — From MSSQL to PostgreSQL (a.k.a. "Saving Real Money")

While we were modernizing the application layer, we also took a hard look at the database tier.

**MSSQL** was costing us approximately **~$800/month in licensing** (SQL Server Standard, core-based licensing on a multi-core VM — your mileage will vary depending on edition, core count, and whether you have Software Assurance). PostgreSQL? **Zero license cost.** It's released under the PostgreSQL License — a permissive open-source license similar to BSD/MIT — meaning no per-core fees, no CALs, no surprise true-ups during audits.

**Important caveat:** "Zero license cost" doesn't mean "zero cost." If you use a managed service like Azure Database for PostgreSQL, you're still paying for compute, storage, backups, and HA configuration. The savings come from eliminating the software licensing layer entirely. In our case, the net savings were still significant even after factoring in the managed service costs.

The migration itself involved:

- Schema translation (T-SQL → PL/pgSQL — mostly straightforward, occasionally painful)
- Data migration using dedicated tooling with checksum validation to ensure data integrity
- Application-level query adjustments (goodbye `TOP`, hello `LIMIT`; farewell `ISNULL`, hello `COALESCE`)
- Performance benchmarking to ensure parity — PostgreSQL's query planner behaves differently from SQL Server's, so we tuned indexes and ran `EXPLAIN ANALYZE` until our eyes glazed over

The savings were real and immediate. But then... things got interesting.

---

## The Plot Twist: Latency Strikes Back 🎭

Here's where the "fun" really began.

Our applications were originally designed for **intranet access** — they lived on-premises and talked to on-premises databases. Fast, snappy, zero complaints.

After migrating to AKS (Azure cloud), these same applications now had to traverse the network boundary between the **corporate intranet** and **Azure's public cloud** for every request. The result? **Noticeable latency.** Users started complaining. What used to feel instant now had a perceptible delay — and when you're accessing an application dozens of times a day, even small latency adds up to a miserable experience.

The irony wasn't lost on us: we'd modernized the application stack beautifully, only to be bottlenecked by network physics. No amount of code optimization can fix the speed of light across a WAN hop.

---

## The Pivot: Internal OpenShift to the Rescue

Rather than stubbornly optimizing around the latency problem, the team made a **pragmatic mid-course correction**: we adopted **Red Hat OpenShift** running on internal infrastructure.

**Why OpenShift made sense here:**

- **On-premises deployment** — eliminated the intranet-to-cloud network hop entirely, putting the apps back where the users (and the data) lived
- **Kubernetes-compatible** — our existing K8s manifests, Helm charts, and CI/CD pipelines transferred with minimal changes. OpenShift is built on Kubernetes, so the migration was more of a "lift and shift" than a rewrite
- **Smoother operator ecosystem** — while tools like Venafi's cert-manager work across any Kubernetes distribution (including AKS and EKS), OpenShift's **OperatorHub** provided a more integrated, first-class experience for automatic certificate lifecycle management. The installation and configuration was noticeably simpler — install the cert-manager operator from OperatorHub, configure the Venafi issuer, and certificates auto-renew without manual intervention
- **Enterprise support** — Red Hat's backing gave leadership the confidence to approve the pivot

This wasn't part of the original plan. But that's the thing about real-world migrations — **the plan is a starting point, not a contract.** The team demonstrated adaptability by prioritizing user experience over architectural purity.

---

## The Stuff Nobody Talks About: CI/CD, Observability, and Security

Here's the thing about cloud migration blog posts — they usually cover the glamorous parts (look at our shiny Kubernetes cluster!) and skip the operational plumbing that actually keeps things running in production. Let's not make that mistake.

### CI/CD Pipeline

You can't do Kubernetes at scale without a solid CI/CD pipeline. Ours looked something like this:

```
Code Push → Build (Maven) → Unit Tests → Docker Build → Image Scan
    → Push to Registry → Helm Deploy → Integration Tests → Promote to Prod
```

Every pull request triggered a build, ran tests, built the Docker image, and deployed to a dev namespace in K8s. Promotion to staging and production required manual approval gates — because nobody wants to auto-deploy a broken WAR file to prod on a Friday afternoon.

### Observability — Because You Can't Fix What You Can't See

Running containers without observability is like driving blindfolded. We implemented the "three pillars":

- **Metrics** — Prometheus scraping Liberty's MicroProfile Metrics endpoints, visualized in Grafana dashboards. We tracked JVM heap usage, request latency percentiles (p50, p95, p99), error rates, and pod resource utilization.
- **Logging** — Centralized log aggregation using **Grafana Loki** with **Promtail** as the log shipper. Loki's label-based indexing (instead of full-text indexing) kept storage costs low while still giving us fast, filterable log queries right inside our existing Grafana dashboards. No more `kubectl logs` across 30 pods to find one stack trace — just filter by namespace, pod, or app label and you're there.
- **Alerting** — Integration with alerting tools for critical alerts (pod crash loops, high error rates, disk pressure). Non-critical alerts went to a team channel. The rule of thumb: if it pages someone at 3 AM, it better be worth waking up for.

### Security Hardening

Enterprise Kubernetes without security hardening is a liability waiting to happen. We implemented:

- **RBAC** — Role-Based Access Control to limit who can do what in which namespace. Developers got read access to prod; only the deployment pipeline could write.
- **Network Policies** — pod-to-pod traffic was deny-by-default. Each service explicitly declared which other services it needed to talk to. This caught a surprising number of "wait, why is Service A calling Service C directly?" situations.
- **Pod Security Standards** — no running as root, no privilege escalation, read-only root filesystems where possible.
- **Image provenance** — only images from our private registry (after vulnerability scanning) were allowed to run. No pulling `random-image:latest` from Docker Hub in production.

---

## Balancing Cost and Performance

This experience highlights the delicate balance organizations must strike between cost considerations and performance optimization during complex migrations. While cost-effective solutions like PostgreSQL can be extremely advantageous (and we'd make the same choice again), addressing the subsequent challenges in network performance is equally crucial to ensure a seamless user experience.

The total cost picture for any cloud migration includes:

- **Infrastructure costs** — compute, storage, networking (egress charges add up fast)
- **Licensing costs** — the MSSQL → PostgreSQL move saved us here
- **Operational costs** — the people and tooling needed to run K8s (don't underestimate this)
- **Opportunity costs** — the months of engineering time spent on migration instead of new features

If you only optimize for one of these and ignore the others, you'll end up with a "successful" migration that nobody is happy with.

---

## Key Lessons Learned

After six months of migration work, here's what we'd tell anyone embarking on a similar journey:

**Do your POCs early.** Our biggest regret? Not running end-to-end proof-of-concept evaluations at the very beginning. We learned how the pieces fit together by doing — which works, but running proper POCs upfront would have surfaced the latency issue *before* we committed to AKS. The decision-making would have been clearer, faster, and less stressful.

**Cost optimization ≠ performance optimization.** Saving on database licensing is great. But if your migration introduces significant latency, your users won't care about the savings. Always evaluate the full picture — license costs, infrastructure costs, operational overhead, and end-user experience.

**Flexibility is a feature, not a failure.** Pivoting from AKS to OpenShift mid-migration felt uncomfortable at first. In retrospect, it was the best technical decision we made. Don't let sunk-cost fallacy trap you in a bad architecture.

**Invest in observability from day one.** Don't wait until things break in production to set up monitoring. Instrument your apps, centralize your logs, and set up alerting before you migrate the first workload. You'll thank yourself when (not if) something goes sideways.

**Containerization is a journey, not a destination.** Getting apps into containers is step one. Operating them reliably at scale — with proper observability, security, CI/CD, and configuration management — is where the real work begins.

**Security is not a phase — it's a constant.** Bake security into every phase: image scanning in CI, RBAC in K8s, network policies between services, secrets management with proper encryption. A security audit at the end is too late; build it in from the start.

---

## The TL;DR Migration Flow

```
Legacy Java Apps
      │
      ▼
 Mavenization (standardize builds)
      │
      ▼
 Liberty Migration (lightweight runtime)
      │
      ▼
 Dockerization (containerize everything)
      │
      ▼
 Kubernetes / AKS (orchestration + Helm + CI/CD)
      │
      ▼
 Externalize Configs (ConfigMaps + Secrets + Vault)
      │
      ▼
 DB Migration: MSSQL → PostgreSQL (save $$$)
      │
      ▼
 Observability (Prometheus + Grafana + EFK)
      │
      ▼
 Latency Issues (intranet ↔ cloud network hop)
      │
      ▼
 Pivot to Internal OpenShift (problem solved)
      │
      ▼
 Security Hardening (RBAC + Network Policies + Image Scanning)
      │
      ▼
Production-ready, performant, cost-effective
```

---

## Final Thoughts

Cloud migration isn't a straight line. It's more like a GPS route that keeps "recalculating" every time you think you've figured it out. But every detour teaches you something valuable, and the destination — modern, containerized, scalable applications — is absolutely worth the trip.

