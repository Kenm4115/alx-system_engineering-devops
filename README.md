# Web Infrastructure Design — www.foobar.com

This README contains **four whiteboard-ready infrastructure designs** for the website `www.foobar.com`. Each task includes:

- An ASCII whiteboard diagram you can copy/draw.
- Short, interview-ready explanations of components and request flow.
- Answers to the explicit questions required by the task.
- Important limitations and mitigations (SPOF, security, scaling, monitoring).

> Notes for reviewers: diagrams are intentionally simple so they are quick to redraw on a physical whiteboard. Keep explanations short and focused — the examiner will ask follow-ups if they want more detail.

---

## Table of Contents

- [Task 1 — Single-server architecture (one server, Nginx, app server, MySQL)](#task-1---single-server-architecture-one-server-nginx-app-server-mysql)  
- [Task 2 — Three-server architecture (HAProxy, 2 servers, MySQL)](#task-2---three-server-architecture-haproxy-2-servers-mysql)  
- [Task 3 — Secured & monitored three-server architecture (firewalls, SSL, monitoring)](#task-3---secured--monitored-three-server-architecture-firewalls-ssl-monitoring)  
- [Task 4 — Clustered LBs + separated roles (LB cluster, separate web/app/db servers)](#task-4---clustered-lbs--separated-roles-lb-cluster-separate-webappdb-servers)  
- [Common quick-scripts for presenting each whiteboard](#common-quick-scripts-for-presenting-each-whiteboard)

---

## Task 1 — Single-server architecture (one server, Nginx, app server, MySQL)


### Request flow (brief)
1. User types `https://www.foobar.com`.  
2. DNS resolves `www` to `8.8.8.8` (A record).  
3. Browser connects to server on port 443 (HTTPS) or 80 (HTTP).  
4. Nginx accepts request, serves static files or proxies to the app server.  
5. App server runs code and queries MySQL as needed.  
6. Response flows back via Nginx to user's browser.

### Required answers (concise)
- **What is a server?**  
  A machine (physical or virtual) that runs services and responds to network requests (CPU, memory, storage, network interfaces).
- **Role of the domain name?**  
  Human-friendly name that DNS maps to an IP address so clients can locate the server.
- **DNS record type for `www`:**  
  **A record** (maps hostname to IPv4 address `8.8.8.8`).
- **Role of the web server (Nginx):**  
  Accepts HTTP(S) connections, terminates TLS (if configured), serves static files, reverse-proxies dynamic requests to the app server.
- **Role of the application server:**  
  Runs the application code (business logic), renders templates, returns dynamic responses, and queries the DB.
- **Role of the database (MySQL):**  
  Persistent storage of structured data (users, posts, transactions).
- **How server communicates with user's computer:**  
  Over the Internet using TCP/IP; HTTP/HTTPS on port 80/443 (HTTPS uses TLS on top of TCP).

### Issues / limitations (short)
- **Single Point of Failure (SPOF):** server crash or network outage brings whole site down.  
- **Downtime for maintenance:** restarts or upgrades can cause downtime unless careful zero-downtime techniques are used.  
- **Cannot scale:** CPU, RAM, and DB I/O limited by single machine resources.

### Short mitigations
- Use graceful restarts, process managers (systemd, supervisor), and backups. For real scale, split roles and add more servers (see Tasks 2–4).

---

## Task 2 — Three-server architecture (HAProxy, 2 servers, MySQL)

### Requirements recap
- Add 2 servers (total of two backend servers serving app/web)
- HAProxy as load balancer
- Codebase on app servers
- MySQL (Primary + Replica concept explained)



### For every additional element — why added
- **Second App Server:** redundancy + horizontal capacity (if one fails, other serves).  
- **HAProxy (Load Balancer):** distributes traffic, performs health checks, central entry point.  
- **Optional Replica DB:** scale reads and provide redundancy for failover.

### Load balancer distribution algorithm (example)
- **Round-Robin:** HAProxy forwards requests sequentially among healthy backends: Req1→A, Req2→B, Req3→A, ...  
  - Works well for similar-capacity servers.
  - Alternatives: **leastconn** (sends to server with fewest active connections), **source** (sticky by client IP), **weighted** round-robin.

### Active-Active vs Active-Passive
- **Active-Active:** both load-balancers or both app servers serve traffic simultaneously. Higher throughput and graceful failover.  
- **Active-Passive:** one node actively handles traffic; passive node stands by and takes over only on failure. Simpler but wastes resources.
- **This design:** App servers are **Active-Active** behind HAProxy.

### Primary–Replica DB cluster (how it works)
- **Primary** accepts writes (INSERT/UPDATE/DELETE) and optionally reads.  
- **Replica(s)** replicate the Primary’s binary log and apply changes to keep a copy; typically serve read traffic and backups.
- **Replication type:** asynchronous (common) — replica may lag; synchronous replication exists but is slower.

### Primary vs Replica for the application
- **Writes** must go to Primary; **reads** can be directed to Replica to reduce Primary load. The app must be aware or use a proxy/router for read/write splitting.

### Issues
- **SPOF(s):** Single MySQL Primary (write SPOF). Single HAProxy (if only one) is an LB SPOF unless clustered.  
- **Security:** no firewall, no HTTPS by default.  
- **No monitoring:** operational visibility missing.

---

## Task 3 — Secured & monitored three-server architecture (firewalls, SSL, monitoring)

### Requirements recap
- Add 3 firewalls (host-based or network boundary)
- 1 SSL certificate to serve `www.foobar.com` over HTTPS
- 3 monitoring clients (one per server) to push logs/metrics to an external collector


### Why each added element exists (concise)
- **Firewalls (3):** block/allow traffic per-role. Example rules:
  - Firewall #1 (edge): allow incoming 80/443 → LB only.
  - Firewall #2/3 (app hosts): allow only LB IPs to app ports, admin IPs for SSH.
  - DB firewall: allow only app servers to port 3306.
- **SSL cert:** encrypts traffic for confidentiality and integrity (TLS). Use a certificate for `www.foobar.com` (Let’s Encrypt or commercial CA).
- **Monitoring clients:** lightweight agents (Sumo Logic collector, DataDog agent, Prometheus node_exporter + pushgateway, etc.) collect sysmetrics, app metrics, and logs and forward them securely to a SaaS or central collector.

### What monitoring is used for
- Detect outages, measure performance (latency, QPS, error rate), collect logs for debugging, alerting on thresholds, long-term capacity planning, and security event detection.

### How the monitoring tool collects data
- **Agent-based model (push):** agent installed on each server collects metrics & tail logs; forwards them over TLS to the monitoring endpoint.  
- **Pull model (Prometheus):** central Prometheus server scrapes `/metrics` endpoints from each host (requires network access).  
- Logging can be forwarded in real time (e.g., filebeat → central store) or aggregated by the agent.

### How to monitor web server QPS (requests per second)
- **Metrics endpoint:** enable Nginx/HAProxy exporter and scrape `nginx_http_requests_total` and compute `rate(nginx_http_requests_total[1m])`.  
- **Log-based:** ingest access logs and compute count per time window (e.g., sum of access log lines per second/minute).  
- Prefer metrics counters (low-latency, accurate, less parsing).

### Issues to explain
- **SSL termination at LB is an issue because:**  
  - Traffic from LB → backend may be unencrypted unless re-encrypted; this breaks end-to-end encryption and can be a compliance/security risk in untrusted network environments.  
  - Client identity (TLS-level client certs) won't be passed through unless configured.
  - Mitigation: re-encrypt connection to backend or use TLS passthrough if you need true E2E encryption.
- **Only one MySQL server accepts writes:** write SPOF — when it fails, writes stop until failover and promotion happen. Mitigate with automated failover tools, replicas, or clustered DB solutions.
- **Servers with all same components (web, app, DB on each host):**  
  - Resource contention (DB I/O vs app CPU) causes unpredictable performance.  
  - Security: compromise of host gives attacker multiple services.  
  - Operational complexity: backups, replication and scaling harder.

---

## Task 4 — Clustered HAProxy & split components (LB cluster + separate web/app/db servers)

### Requirements recap
- 1 (additional) server to build a cluster with HAProxy
- Two HAProxy instances configured as a cluster (redundant LBs)
- Split components: separate servers for web, app, and DB


### Why each element is added
- **Second HAProxy (LB2):** avoids LB single point of failure; can be Active-Active or Active-Passive. Use VRRP/keepalived for virtual IP or DNS-based failover.  
- **Dedicated Web servers:** handle static content and act as a buffer to app servers.  
- **Dedicated App servers:** run code and scale horizontally.  
- **Dedicated DB server(s):** optimized for I/O/consistency (Primary) and replicas for reads or failover.  
- **Separation benefits:** independent scaling, easier security hardening, isolation of failures, and better performance tuning per role.

### Short notes on configuration choices
- **LB cluster mode:** Active-Active (both LBs accept traffic) with a virtual IP (keepalived) or DNS round-robin plus health checks.  
- **Read/write DB strategy:** Direct writes to Primary, route reads to replicas (via app or proxy).

### Issues & mitigations (brief)
- **SPOFs** reduced but not eliminated: need redundant DB primaries or automated promotion; make sure LB cluster has reliable failover (VRRP or DNS).  
- **Operational overhead:** more machines require config management (Ansible/Chef/Puppet) and monitoring.  
- **Network segmentation & security:** use internal networks/VLANs for backend traffic, firewall rules, and internal TLS where required.

---

## Common quick-scripts for presenting each whiteboard

Use these short scripts when presenting each diagram in a live whiteboard session (30–90s each):

### Task 1 script (30s)
“User types `www.foobar.com`. DNS resolves `www` to `8.8.8.8` (A record). Browser connects to Nginx on the server; Nginx serves static files or proxies to an app server running our code. The app reads/writes to MySQL on the same machine. This is simple and cheap but has a single point of failure and cannot scale well; maintenance or crashes take the whole site down.”

### Task 2 script (45s)
“This extends Task 1 to two app servers behind HAProxy. HAProxy does round-robin and health checks; app servers host Nginx+app and talk to a MySQL primary (with optional replica for reads). We add redundancy and read scaling. Downsides: primary DB is still a write SPOF, and if HAProxy isn’t redundant it’s a SPOF too. Also we need to add security (firewall) and monitoring.”

### Task 3 script (60s)
“Here we add security and observability: three firewalls (edge, app, DB), an SSL certificate on the LB for HTTPS, and a monitoring agent on each server that forwards metrics and logs to a collector. Monitoring gives us QPS, error rates, and alerts; use Nginx/HAProxy metrics or access logs to compute QPS. Watch out: terminating SSL at the LB leaves backend traffic unencrypted unless you re-encrypt; and a single MySQL write node remains a risk.”

### Task 4 script (60s)
“Finally we add a second HAProxy and split roles: dedicated web servers, app servers, and DB servers. LBs run as a cluster (Active-Active or Active-Passive with VRRP). Split roles allow independent scaling and reduce resource contention. This reduces SPOFs at the LB layer, but DB writes still depend on a primary unless you introduce automated failover or multi-primary DBs.”

---