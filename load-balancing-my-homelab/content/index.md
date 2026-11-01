---
title: "Load Balancing My Homelab"
date: 2026-11-01
slug: load-balancing-my-homelab
author: "spleenftw"
status: published
tags:
  - "homelab"
  - "haproxy"
  - "anubis"
  - "cloudflare"
  - "headscale"
  - "tailscale"
  - "ansible"
  - "semaphore"
  - "docker"
  - "monitoring"
  - "self-hosting"
description: "How I deployed two HAProxy and Anubis edges with Cloudflare load balancing, Headscale and Semaphore: configuration, failover tests and direct media access."
category: "network"
---


<h2>Another perfectly reasonable weekend project</h2>

<p>The public entrance to the homelab has changed again. In <a href="https://blog.interlope.xyz/my-homelab-setup-2">WIMH #2</a>, I described the VPS that forwards traffic into the lab over the overlay, without opening ports on the home connection. That arrangement is still there, but the entrance was mixed in with the application machines: Nginx, Anubis and the services they exposed shared those VPS hosts. This time I moved the public entry onto two dedicated machines, in different OVH locations, and put Cloudflare Load Balancing in front of them. Which sounds like a fairly contained change until you start following a request all the way through it.</p>

<p>There is now HAProxy on each edge, an Anubis instance per application, private routes back to the origins, a fallback through the other edge, certificates to renew, and Semaphore deploying the whole thing from Git. The applications stayed where they were. The interesting work was getting the new entrances to reach them, deciding what should happen when something breaks, and then actually breaking the appropriate thing to check. Also, the media clients needed their own treatment. Asking a video player to solve a browser challenge was never going to be a good viewing experience.</p>

<p>Same homelab, more plumbing. I'll call the entrances <code>edge-a</code> and <code>edge-b</code>, the application VPS machines <code>origin-a</code> and <code>origin-b</code>, and the home subnet router <code>home-router</code>. These are aliases. The addresses and hostnames in the examples are placeholders, the screenshot legends have been replaced, and I'm keeping the actual service catalogue and access policy private. You can follow the routing without getting a map of everything currently online.</p>

<p>Here is the short version before we get into the deployment. Web requests go through Cloudflare to either edge, then HAProxy and Anubis, and finally to the application over the private network. For services at home, the last part crosses the subnet router and firewall to Nginx Proxy Manager. NPM still does the forwarding inside the lab; the new edges haven't replaced it.</p>

<p><img src="../img/architecture.png" alt="Anonymized overview of two public edges, private application origins and direct media access" /></p>

<p>And here is the less short version. The diagram also shows the deployment and monitoring paths, which aren't extra stops for a visitor's request. Headscale manages the Tailscale clients and their access policy; HTTP traffic doesn't pass through Headscale itself. That is another change since the older WIMH post, which described the official Tailscale control plane. The red panel is a separate explanation of what the redundancy covers and where it stops.</p>

<figure>
<a href="../img/infrastructure-overview.png"><img src="../img/infrastructure-overview.png" alt="Detailed anonymized architecture: Cloudflare proxy and DNS-only access, two HAProxy and Anubis edges, private service failover, Tailscale origins, Semaphore deployment, Headscale policy, certificates and Prometheus monitoring" loading="lazy" /></a>
<figcaption>Open the image at full size to follow the paths. The separate red panel explains failure handling and redundancy limits; it is not an infrastructure component or a traffic path. Cloudflare checks the public entrance; each HAProxy checks its application paths.</figcaption>
</figure>

<p>The <a href="../files/infrastructure-mermaid.txt">Mermaid source</a> is available if you want to follow the connections at full size. The application groups in it are generic.</p>

<h2>First, make the applications reachable</h2>

<p>Before putting a second entrance anywhere near the public DNS, I had to check the origins. One application was listening only on loopback, which made perfect sense while its reverse proxy ran on the same machine. From a new edge, that listener might as well not exist. I added a binding on its overlay address while keeping loopback for the old entrance, then checked the other applications' listeners too. This let the migration happen alongside the working setup rather than replacing its access path before the new one had been tested.</p>

<p>At home, the subnet router already provided the route to NPM. The firewall between that router and the gateway permits HTTPS to NPM, so the edges use that specific path rather than getting access to the entire LAN. They join Headscale with <code>tag:edge</code>, accept the approved private routes, and keep their ordinary Internet egress. They don't select an exit node. That last detail becomes relevant when Certbot starts talking to Cloudflare from the VPS's own public address.</p>

```bash
sudo tailscale up \
  --login-server=https://control.example.net \
  --hostname=edge-a \
  --advertise-tags=tag:edge \
  --accept-routes
```

<p>The second edge uses its own hostname, followed by registration and approval on the control plane. Then come the checks: the right tag, the right route, and an actual response from the origin. Being present in the mesh only gets us part of the way. During preparation, a request between the application VPS machines timed out even though both nodes were online. Their role didn't have the destination access intended for the new edges. Opening that access just to make an arbitrary test machine happy would have changed the policy for the wrong reason; the useful test was from a node carrying <code>tag:edge</code>.</p>

<p>For an HTTPS origin behind NPM, I tested the application hostname against the private address directly. This keeps the Host header and TLS name correct while avoiding public DNS resolution. The certificate still has to validate:</p>

```bash
curl --noproxy '*' --connect-timeout 5 --max-time 15 \
  --resolve app.example.net:443:NPM_PRIVATE_IP \
  -I https://app.example.net/
```

<p>That private resolution is carried into the deployment too. Anubis targets the application hostname with a private address mapping, and HAProxy's origin probe uses the proper SNI and verifies the origin certificate. Once this part works from each edge, there is something worth deploying in front of it. Until then, a beautiful proxy configuration would just produce a more elaborate error page.</p>

<h2>Two edges, two different decisions</h2>

<p>The part that needed untangling was the word "failover", because there are two separate decisions in this setup. Cloudflare chooses the public entrance. Both edges are active, with equal weights and Random steering inside the pool; session affinity then keeps a browser on its selected edge while that endpoint stays healthy. This won't alternate every request between the machines. One session can sit on one edge and transfer much more data than several visits on the other. I'll come back to that when we reach the graphs.</p>

<p>After the request lands, HAProxy chooses a path for that particular application. Normally it uses the local Anubis. If that path becomes unhealthy, it can use a private gateway on the other edge, which forwards to the peer's local Anubis and then to the origin. So a visitor can still enter through edge-a while one application's request crosses to edge-b. The other healthy applications on edge-a keep their local paths. We aren't moving the entire entrance because one container stopped.</p>

```text
Normal:
visitor → Cloudflare → edge-a → local Anubis → origin

Local Anubis unavailable:
visitor → Cloudflare → edge-a → private gateway on edge-b
                              → edge-b Anubis → origin

edge-a unavailable:
visitor → Cloudflare → edge-b → local Anubis → origin
```

<p>Those private gateways bind to the overlay addresses only. Headscale restricts which nodes may reach them, and HAProxy checks the source edge and the expected Host as well. Each gateway has one destination: its own local Anubis. It cannot choose the peer again, so there is no opportunity for the two edges to pass the same request back and forth. The public access restrictions run before HAProxy chooses either path.</p>

<p>Here is the backend on edge-a for an example application. The server called <code>anubis</code> actually points at the local private gateway, because that's where the complete local path gets checked. The second server is the peer, marked <code>backup</code>. Replace the address tokens before trying to use this anywhere:</p>

```haproxy
backend be_app
    cookie EDGE_ROUTE_app insert indirect nocache httponly secure
    option httpchk
    http-check send meth GET uri /_edge/health ver HTTP/1.1 hdr Host app.example.net
    http-check expect status 200

    server anubis EDGE_A_MESH_IP:18923 source EDGE_A_MESH_IP cookie edge-a check inter 5s fall 2 rise 2 init-state fully-down
    server peer EDGE_B_MESH_IP:18923 source EDGE_A_MESH_IP cookie edge-b backup check inter 5s fall 2 rise 2 init-state fully-down
```

<p>There are cookies at both layers. Cloudflare's affinity cookie chooses the public edge, while <code>EDGE_ROUTE_app</code> keeps an application's chosen local or peer path consistent through its challenge and subsequent requests. The cookie values identify the same edge on both machines. A session that moved to the peer can stay there after the local path recovers; a fresh session prefers the healthy local path. That explains why reloading one existing browser session after recovery doesn't tell you where all new traffic is going.</p>

<p>The checks also have their own timing. The gateway checks above run every five seconds with two failures or successes needed for a state change, but they depend on the separate Anubis and origin checks, which have their own intervals and timeouts. Multiplying five by two doesn't give us a reliable total failover time. Existing connections are a separate matter again: a WebSocket or video stream doesn't move between processes. The client reconnects, and this HAProxy configuration adds no policy to replay a POST after receiving an application response.</p>

<h2>Getting a useful green light out of Cloudflare</h2>

<p>For Cloudflare, I settled on one shared pool containing the two endpoints, reused by the application load balancers. This stayed within the endpoint allocation shown by my subscription instead of repeating a two-endpoint pool for each application. The tradeoff is that all those load balancers share the pool's health decision. A monitor aimed at one application would let that application's failure affect the other domains too. A monitor requiring every application to be healthy could remove both edges because one common origin had failed. Neither outcome would fix the application.</p>

<figure>
<img src="../img/pool.png" alt="Cloudflare pool with two healthy anonymized endpoints, each weighted 1, and Random steering" loading="lazy" />
<figcaption>The shared pool after deployment: both endpoints healthy and enabled, with equal weights. Names and IP addresses have been replaced.</figcaption>
</figure>

<p>The common monitor therefore checks the public HTTPS entrance. It requests <code>/_edge/entry-health</code> on a dedicated Host and expects HTTP 200 with <code>edge-ready</code>. HAProxy produces that response after checking the Host, the actual network source and the monitor token. The settings are HTTPS on port 443, GET, a 60-second interval, five-second timeout and two retries. Certificate verification stays enabled. Meanwhile, HAProxy checks each application's Anubis and origin separately: the private gateway is ready only when both local components are healthy, and the public per-service <code>/_edge/health</code> route can accept a healthy local or peer path.</p>

<figure>
<img src="../img/monitor-settings.png" alt="Cloudflare HTTPS monitor configured with a 60-second interval, five-second timeout, two retries, status 200 and body edge-ready" loading="lazy" />
<figcaption>The actual common monitor settings. Its request headers are configured separately; no token value is included in this screenshot.</figcaption>
</figure>

<p>This means the common Cloudflare probe can stay green while an application is unavailable. That's the intended division of work, but it has a limit worth spelling out once: if an edge loses both paths for a service while the peer can still serve that service to visitors arriving directly, the shared ingress monitor won't redirect only that domain. Separate service pools could make that choice. Here, the private fallback handles the per-service routing, and Prometheus exposes the detailed path states. <a href="https://developers.cloudflare.com/load-balancing/monitors/">Cloudflare's monitor documentation</a> explains the pool health side.</p>

<p>Getting the monitor green also produced one of the more concrete mistakes in the deployment. Both endpoints reported a TLS name mismatch, while the local HTTPS test with the intended hostname worked. The fix was in the request headers: the hostname belongs in the value of <code>Host</code>, and the secret belongs in a second header. Two fields, two different jobs:</p>

```text
Host: edge-health.example.net
X-Edge-Health-Token: <monitor secret>
```

<p>Cloudflare uses that Host for the monitor's TLS SNI. The endpoints' Host overrides stay empty so they don't take precedence over it. I left certificate verification on and corrected the name instead; <a href="https://developers.cloudflare.com/load-balancing/troubleshooting/common-error-codes/">the error reference</a> covers this case. For a local check, loopback is exempt from the external monitor-token requirement:</p>

```bash
curl --noproxy '*' --silent --show-error --max-time 10 \
  --resolve edge-health.example.net:443:127.0.0.1 \
  https://edge-health.example.net/_edge/entry-health
```

<p>That should return <code>edge-ready</code> with a verified certificate. It proves the local listener works. The external Cloudflare monitor still has to pass its own network, source and header checks, which is exactly where the mismatch showed up.</p>

<h2>The browser challenge meets the actual clients</h2>

<p>With the routes in place, <a href="https://github.com/TecharoHQ/anubis">Anubis</a> sits in front of each application on each edge. Its listeners and metrics remain on loopback, and each instance gets its own target and policy directory. Browser-facing routes receive the proof-of-work challenge. The complications start with API clients, notification streams and media playback, because those requests still need to work without a browser solving anything. The policies therefore follow how the application is used, rather than applying the same challenge to every request.</p>

<p>The media policy is a good example. Its web interface and bare root receive a challenge, including mixed-case variants of the web path. The remaining routes pass through to the application's own authentication, so native clients, API calls, media requests and sockets can work. The specific rule has to come before the catch-all allowance:</p>

```yaml
bots:
  - name: media-browser
    path_regex: '(?i)^/(?:web(?:/|$)|$)'
    action: CHALLENGE
    challenge:
      algorithm: fast
      difficulty: 5

  - name: media-native-routes
    path_regex: '^/'
    action: ALLOW
```

<p>An <code>ALLOW</code> here skips Anubis's challenge; the application still checks the account or token. I checked that distinction anonymously on both edges: the tested private routes for users, sessions, libraries, server settings and API keys returned 401. The server-information route remained public, the public user-list route returned an empty list, and HAProxy blocked <code>/health</code> and <code>/metrics</code> with 404. The protected edge readiness route returned 403 without its secret. Those checks cover the tested routes, not every permission or plugin, but they establish why blocking the entire API would break the clients rather than solve the access question.</p>

<p>Other clients need different exceptions. Notification delivery and streaming use their API routes; a request application's web UI can depend on its API as well as external integrations. Changing a policy changes its content hash in the container labels, which causes a recreation. The policies are mounted as directories so replacing a file atomically doesn't leave a container reading an old inode. HAProxy also normalizes the relevant application paths and rejects ambiguous encodings before applying restrictions. Otherwise, a rule for <code>/admin</code> could miss an equivalent encoded request before Anubis even enters the discussion.</p>

<p>The media entrance needed another change: it uses a DNS-only Cloudflare load balancer. Cloudflare selects healthy addresses through DNS, then the client connects directly to an edge over HTTPS. The video doesn't cross Cloudflare's reverse proxy. The other web entrances remain proxied.</p>

```text
DNS:    media hostname → healthy edge address
HTTPS:  client → edge-a or edge-b → Anubis → home NPM → the media service
```

<p>That required adjusting the service's source restriction before switching its DNS. The proxied vhosts accept Cloudflare sources and explicit administration exceptions; a direct media client needs to be accepted on its own vhost. Just changing the cloud colour would have left ordinary users with a 403. I checked the direct route from the administration connection first, then from a phone outside that exception. The logs showed the external client reaching the second edge directly, and login and playback worked. Testing only from the allowlisted connection would have missed a broken public rule.</p>

<p>DNS caching still applies, and an open stream doesn't migrate when an edge fails. That's part of the <a href="https://developers.cloudflare.com/reference-architecture/architectures/load-balancing/">DNS-only load-balancing model</a>. The configuration gives new connections another available entrance after detection and resolution; it doesn't pick up a running video connection and carry it across.</p>

<figure>
<img src="../img/application-lb.png" alt="Anonymized Cloudflare application load balancer showing a healthy shared pool and two endpoints" loading="lazy" />
<figcaption>An application load balancer after migration, with its identifying labels replaced: one healthy pool with both endpoints. This row shows pool health, not application login behavior.</figcaption>
</figure>

<h2>Teaching Semaphore what an application looks like</h2>

<p>Once the routing existed, the next question was maintenance. Adding another application shouldn't mean finding every list of ports and services in several templates and hoping I remembered all of them. The configuration now starts with <code>docker/haproxy/vars/services.yml</code>: one entry for the domain, origin, permanent ID, policy profile and any application-specific health or access rules. This is an illustrative entry, with the origin and domain replaced:</p>

```yaml
haproxy_service_catalog:
  example_app:
    id: 1
    domain: app.example.net
    origin: {node: origin-a, port: 8080}
    profile: browser
    admin:
      paths: [/admin]
    blocked_paths:
      - {path: /api, status: 403}
    dns: manual
```

<p>The ID stays put even if the YAML entries move around. It gives us Anubis port <code>8922 + ID</code>, local metrics port <code>9200 + ID</code>, and private gateway port <code>18922 + ID</code>. For ID 1, those are 8923, 9201 and 18923. Deleting a different entry won't renumber that application. The catalogue compiler checks the IDs, profiles, defined origins and derived port collisions, then feeds generic templates for the routing, containers, policies, probes and managed Headscale permissions.</p>

<p>There is a separate Semaphore plan task with a local inventory and no deployment secrets. It validates the declarations and writes a service summary and ACL fragment on the runner. It makes no VPS or DNS changes, and doesn't test the network:</p>

```bash
ansible-playbook -i localhost, docker/haproxy/plan_services.yml
```

<p>For an application behind home NPM, the practical sequence is to deploy it privately, configure its Proxy Host and certificate, enable WebSocket support if needed, and check the private HTTPS route from the edges. Then add its exposure entry. Here is an eighth-service example, not another application currently deployed:</p>

```yaml
  example_app:
    id: 8
    domain: app.example.net
    origin:
      node: npm
      port: 443
      scheme: https
    profile: browser
    health:
      path: /
      status: '200-399'
    dns: manual
```

<p>The catalogue owns the exposure configuration. The application's database, persistent data and backups belong to its own deployment. A client that cannot solve a browser challenge can use <code>profile: native</code>, with an explicit health route returning 200; behavior outside the supported profiles still needs a code change. Similarly, disabling an entry removes its active routes and containers from this project and allows the managed ACLs to be withdrawn, but doesn't delete the application data or automatically remove its public DNS.</p>

<p>The actual deployment task runs <code>docker/haproxy/deploy_haproxy.yml</code> serially across the edges. It checks the target, mesh membership, routes and origin responses; obtains the certificate; pulls the selected images; and validates the candidate configuration with the downloaded HAProxy image before installing it. Then it checks the local TLS routes, challenges, redirects and restrictions, followed by the local and peer application paths. The images deliberately remain <code>haproxy:lts</code> and Anubis <code>latest</code>, so a deployment can also update an image. So the job's changes can include an image update as well as my configuration.</p>

<p>Semaphore supplies four separate secrets: the DNS API token, shared Anubis signing key, monitor token and metrics token. They go in secret extra variables, with environment-variable fallbacks supported by the playbook. The SSH account and sudo settings have to work on the edges and on the control host receiving delegated Headscale tasks. That delegation reads the active policy, replaces only the rules owned by this deployment, validates a candidate, backs up the old policy and reloads when necessary. An adjacent tracking file remembers the managed rules so permissions can be removed later without replacing the rest of my policy.</p>

<p>Bootstrap needs a small exception. The first edge cannot validate a peer stack that hasn't been installed yet, so peer readiness isn't required for the initial rollout. After both stacks are running, I enable the blocking check and deploy again:</p>

```yaml
haproxy_edge_redundancy_enabled: true
haproxy_edge_require_peer_ready: true
```

<p>That stays enabled afterward. A healthy peer mustn't hide a broken local installation during deployment, and a healthy local installation mustn't let a broken backup slip through. The normal target selection is simply the edge inventory group:</p>

```json
{"target_hosts": "vps_haproxy"}
```

<p>The Cloudflare application load balancers are still managed separately. The playbook handles dedicated DNS-only edge records and offers a simple publication mode toward a single primary edge, but that single-primary mode doesn't give us the public two-edge arrangement. The migrated applications keep <code>dns: manual</code>. The catalogue and a green Semaphore run aren't a hidden instruction to switch all the public names.</p>

<h2>And then the deployment had opinions</h2>

<p>The first edge took several runs to get right. There were runner dependency and proxy issues, followed by SSH account and sudo settings to align with the working Semaphore templates. The successful execution skips Galaxy installation because the collections are already available on the runner; skipping the step doesn't make a missing collection appear. Once those prerequisites passed, Certbot reached Cloudflare and ran into the token's source-address restriction. The API request came from the edge's public egress, because Certbot runs there. The token needed to allow that address, and eventually the second edge's, for issuance and future renewals.</p>

<p>Each edge obtains its certificate through DNS-01 and renews it locally. The renewal hook assembles the certificate and private key, checks that they match, validates HAProxy and signals the master to reload. If validation or signalling fails, it restores the previous PEM on disk. That safeguard covers this local step; it doesn't turn the whole playbook into a transaction. The final successful deployment still needed one retry of the per-domain HTTPS check on each edge. Both passed on the next attempt.</p>

<p>By the end, neither edge had an unreachable host or failed task. The checks covered the application TLS routes and browser challenges, the blog-alias redirects, restricted and encoded paths, local readiness and every configured peer route. The Headscale candidate was valid, and its reload was skipped because the active policy hadn't changed. Temporary candidate files still counted as changes in the job. The application publication task was skipped, the Cloudflare pool settings stayed untouched, and the deployment notification arrived. The full log stays private; publishing it would also publish the inventory embedded in its task results.</p>

<p>There is one other boundary in the request handling that I don't want buried in the diagrams. HAProxy only accepts <code>CF-Connecting-IP</code> as the visitor address when the connection actually comes from a configured Cloudflare range. Other connections use their real source address. It rebuilds the forwarding headers before passing requests downstream, while monitor and metrics access rules still check the real network peer. An arbitrary client mustn't get the administration exception by supplying a convenient header. The trusted ranges are an explicit configuration snapshot, so they need periodic review. The logs also leave out request URLs, since some clients carry credentials in query strings.</p>

<h2>Time to stop something</h2>

<p>I tested the new entrances before changing the application DNS. Local resolution overrides let the browser keep the real hostname and certificate while connecting to the new edge, with other visitors still using the old entrance. For command-line checks, <code>curl --resolve</code> does the same thing; the address below is a documentation address:</p>

```bash
curl --noproxy '*' --silent --show-error \
  --connect-timeout 5 --max-time 15 \
  --resolve app.example.net:443:192.0.2.10 \
  --output /dev/null \
  --write-out 'HTTP %{http_code}, TLS %{ssl_verify_result}\n' \
  https://app.example.net/
```

<p>A zero TLS verification result confirms the certificate check, but HTTP 200 can still be the challenge page. I had to go through the application logins, watch live updates, receive notifications and test playback. The readiness and metrics routes were also checked from an unauthenticated workstation and returned 403. Those are different checks from using the applications, and both mattered before the public move.</p>

<p>For the first controlled failure, I stopped one application's Anubis container on edge-a while leaving its common origin running. Requests still entering edge-a selected the peer backend, and a WebSocket opened through edge-b. That exercised the private fallback without deliberately taking the shared application down. The public migration then started with one status service: after enabling its Cloudflare load balancer, I stopped HAProxy on edge-a. Cloudflare detected the failed entrance, the other edge served the application, and I restored edge-a and waited for health to settle.</p>

<p>These tests stop different containers for different reasons: an application Anubis has a name such as <code>haproxy-prod-anubis-example_app</code>, while <code>haproxy-prod</code> is HAProxy itself. Restoring the stopped container and checking recovery is part of the test. The first case proves that one service can use its peer path; the second proves that the public entrance can move. Neither test implies that an open socket teleports to another machine.</p>

<p>The remaining proxied names followed gradually, with logins and notifications checked after switching. The root and <code>www</code> aliases were pointed at the blog and their redirects checked. The media entrance moved separately through its DNS-only load balancer, followed by the external phone test. Only after those paths worked did I disable the old application vhosts, validate and reload Nginx, check the retained services, and stop the redundant old Anubis containers.</p>

<p>The origin application containers stayed running, because the new edges still use them. Nginx and Certbot also stayed where other active services needed them, including the Headscale entrance. Retaining the previous application CNAME records helped with DNS rollback during migration, but once a vhost is disabled, reverting DNS alone is no longer enough: that old vhost has to be restored too. The older Nginx deployment playbook belongs to that previous entrance, so rerunning it during cleanup would risk recreating what I had just retired.</p>

<h2>The graphs, now with traffic in them</h2>

<p>The existing Prometheus stack scrapes the edges over public HTTPS, using a dedicated DNS-only name for each machine. It doesn't join the mesh. The metrics endpoint requires both the authorized public source and a separate Bearer token, while the unauthenticated exporter remains on loopback. Giving Prometheus an application load-balancer name would defeat the point: two scrapes could reach the same selected edge while looking like two independent observations. Here is the job structure, with generic targets and a placeholder for the monitoring stack's existing outbound proxy:</p>

```yaml
scrape_configs:
  - job_name: haproxy-edges
    scrape_interval: 15s
    scrape_timeout: 10s
    scheme: https
    proxy_url: http://monitor-proxy.example.net:3128
    metrics_path: /_edge/metrics
    authorization:
      type: Bearer
      credentials_file: /etc/prometheus/secrets/edge-metrics-token
    tls_config:
      insecure_skip_verify: false
    static_configs:
      - targets: ['edge-a.example.net:443']
        labels:
          edge: a
      - targets: ['edge-b.example.net:443']
        labels:
          edge: b
```

<p>The token file has to be readable inside the Prometheus container, and the source allowlist has to match the actual egress of that connection. After that, <code>up{job="haproxy-edges"}</code> tells us whether the scrape succeeded. The application states come from the other metrics. Grafana can then put traffic, sessions, backend errors, origin probes and the available service paths next to each other, instead of collapsing everything into one green square.</p>

<figure>
<img src="../img/traffic.png" alt="Grafana scrape availability, traffic share, per-service response rates and response bytes for the two anonymized edges" loading="lazy" />
<figcaption>A real observation window after migration. Both scrape series are at one; traffic reaching Anubis is uneven across the two edges. Legend labels have been anonymized.</figcaption>
</figure>

<p>This window has both scrape series at one and an uneven traffic split. That's entirely compatible with equal configured weights: sessions have different sizes, a media stream moves far more bytes than a short page visit, and requests satisfied by Cloudflare's cache never reach these backends. I read this as traffic arriving at the edges, not as a count of unique visitors or proof that each machine got exactly half the users.</p>

<figure>
<img src="../img/health.png" alt="Grafana service-path and origin health, active backend sessions and backend 5xx responses" loading="lazy" />
<figcaption>The sampled health window contains brief drops and a few backend errors, followed by healthy values. The screenshot alone does not identify their cause.</figcaption>
</figure>

<p>There are brief health drops and backend error peaks in the sample. Their cause would need the matching logs and time-aligned events; the capture alone doesn't establish how long a visitor lost access. There is also a dashboard naming trap here: "Anubis health" displays <code>be_*</code> series, which in the redundant configuration represent service paths through the gateways. I read that panel with the <code>probe_*</code> origin series rather than treating its title as a measurement of an isolated Anubis process.</p>

<figure>
<img src="../img/latency.png" alt="Grafana rolling backend response times and health-check failure rates" loading="lazy" />
<figcaption>Rolling response time and check failures. The health-check legend containing infrastructure labels has been removed; the plotted samples are unchanged.</figcaption>
</figure>

<p>One rolling response-time series reaches roughly 850 ms and stays there in the displayed window. It's an average, not a p95, and a quiet service can retain a value without much new traffic. That gives me something to inspect, not a latency diagnosis by itself. Likewise, these metrics don't contain Cloudflare's actual endpoint verdict; that lives in its monitor and event view. The reference alert rules need to be loaded and connected to notification delivery separately. Importing the dashboard doesn't install the alerts.</p>

<h2>Where this leaves the lab</h2>

<p>The public entrance is now redundant, with a private fallback for each application's path and a catalogue for maintaining the exposure configuration. The application origins are still shared. If a single application instance fails, both edges lose it; if the home connection, gateway or NPM fails, several private paths can fail together. Both VPS instances also remain with one provider, and the proxied entrances depend on Cloudflare. Those are the boundaries shown in the separate red panel of the diagram.</p>

<p>The deployment, access checks and controlled stops are done, and the old application entrances have been retired without deleting the origins underneath them. For the next application, there is a private deployment to prepare, a catalogue entry to review and an edge rollout to run. Replicating the application and its data is still its own project. This weekend's one already had enough moving parts.</p>

<p>Thanks for reading me,<br>spleenftw</p>
