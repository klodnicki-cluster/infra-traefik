# Cloudflare-Protected Ingress Plan

## Goals

- Require Google Workspace Cluster Admins membership for regional cluster
  hostnames.
- Require Workspace membership, but not Cluster Admins membership, for selected
  private applications such as `photos.klodnicki.com`.
- Keep other applications public.
- Load balance selected public applications across regions.
- Keep Traefik as the Kubernetes ingress controller.
- Keep public traffic from bypassing Cloudflare Access by reaching Traefik
  directly.

## Request path

```text
Browser
  -> Cloudflare DNS and proxy
  -> Access policy for protected hostnames
  -> Cloudflare Load Balancer for multi-region applications
  -> regional Cloudflare Tunnel
  -> regional Traefik
  -> Kubernetes Service
```

Access is applied to the hostname requested by the client. A public hostname
such as `www.klodnicki.com` is outside the regional cluster Access hostnames and
can remain public. If account-wide default-deny Access is enabled, explicitly
make intended public hostnames public.

## DNS and Cloudflare

- Keep the domain registrar; delegate authoritative DNS to Cloudflare by
  changing the domain's nameservers there.
- Copy and verify existing DNS records before changing nameservers, including
  mail-related MX and TXT records.
- Proxy all application hostnames through Cloudflare. Do not leave public
  routes that bypass Cloudflare and reach Traefik directly.
- Cloudflare Load Balancing is an account add-on. Current billing documentation
  shows a $5 Basic Load Balancing subscription, with separate usage-based
  items; the first 500,000 Load Balancing DNS queries are included. Confirm
  current pricing and usage in the account before enabling it.

## Regional cluster access

Create a distinct Cloudflare Tunnel for each region:

- zack region
- mh region
- josh region, when ready

Run tunnel connectors in their corresponding region and route them to that
region's Traefik ingress. Restrict direct access to Traefik's node ports and
load-balancer address so Cloudflare Access cannot be bypassed.

Protect these hostname patterns with a Cloudflare Access application and a
Google Workspace policy requiring the Cluster Admins group:

- `*.cluster.zack.klodnicki.com`
- `*.cluster.mh.klodnicki.com`
- `*.cluster.josh.klodnicki.com` when that region is added

For example, a regional app hostname under the original naming scheme is
`www.cluster.zack.klodnicki.com`. The order matters:
`www.zack.cluster.klodnicki.com` is a different hostname and does not match
`*.cluster.zack.klodnicki.com`.

Use a separate Workspace-membership policy for applications such as
`photos.klodnicki.com`. Public single-region applications do not require an
Access policy.

Google group membership and active Access sessions may not reflect removals
immediately. Set an appropriate session duration and document how to revoke a
user's active Access session for urgent offboarding.

## Multi-region public applications

For `www.klodnicki.com`, configure one Cloudflare Load Balancer with the
regional tunnel endpoints in a pool. Each endpoint uses its tunnel UUID as the
endpoint address and a per-endpoint `Host` header override matching that
region's published application route. The regional route sends traffic to its
local Traefik ingress.

The regional published route hostnames may differ. They do not need to be the
same hostname in every tunnel. Use separate tunnel UUIDs for regions; a pool
cannot reference the same tunnel UUID more than once.

Attach an HTTPS health monitor that checks the real application health path.
For tunnel endpoints, a fixed `HTTP_STATUS` route in `cloudflared` only proves
that the connector is reachable; it does not test the upstream application.

Cloudflare Load Balancing has an account-level base subscription and additional
usage billing. Create a Load Balancer for each public hostname that needs
independent multi-region routing, and monitor account usage.

## Health checks and failover

There are two distinct failure paths:

1. **Active health monitoring:** Cloudflare periodically probes each origin.
   Configure the probe to reach the app's real readiness/health endpoint and
   expect `200`. If the app is unavailable and Traefik returns `503`, the
   monitor can mark that endpoint unhealthy after its configured failure
   threshold. This is proactive but waits for monitor intervals and retries.
2. **Zero-downtime failover:** Enable Cloudflare's zero-downtime failover under
   Adaptive Routing. It can retry a request once against another healthy
   endpoint in the same pool when Cloudflare detects a `521`, `522`, `523`,
   `525`, or `526` connection/TLS failure. Cross-pool failover is a separate
   setting if the design uses multiple pools.

Do not configure Traefik to return `521`, `522`, `523`, `525`, or `526` when an
application is unavailable. These are Cloudflare-generated failure codes, not
application status codes; arbitrary origin responses do not trigger
zero-downtime failover. Traefik normally returns `503` when a router matches but
has no ready servers, and `502` for errors contacting an upstream. Use those
real health outcomes with the active monitor, not a fabricated Cloudflare
status.

For faster application-readiness response than monitor polling, a future
Kubernetes controller could watch region-specific Service `EndpointSlice`s and
disable the corresponding Cloudflare pool endpoint when no ready backends
remain. It would re-enable the endpoint after recovery. This would be custom
integration: Cloudflare exposes endpoint enable/disable, not a writable health
status. Keep external health monitoring as a fallback.

## Kubernetes availability constraints

All regions currently participate in one Kubernetes cluster whose control
plane resides in one region. If that region is unavailable, existing workloads
may continue serving, but Kubernetes cannot reliably schedule or reconcile new
work. A controller that depends on the Kubernetes API may stop receiving
EndpointSlice changes, so endpoint-driven Cloudflare updates cannot be the only
failover mechanism.

Keep Cloudflare health monitoring independent of Kubernetes. Test the actual
control-plane-region failure scenario. Cloudflare routing does not provide
multi-region database, storage, or session replication; those dependencies need
their own availability design.

## Automation approach

Start with manually managed Cloudflare tunnels, Access applications/policies,
health monitors, and Load Balancers. A single multi-region public hostname does
not require a Kubernetes controller. At the time of writing, no established
drop-in controller has been identified for the complete workflow of translating
existing Traefik Ingress annotations, regional Service readiness, Cloudflare
Access policy, and Load Balancer state.

If automatic endpoint status updates become necessary, prototype a narrowly
scoped controller that:

- Maps each regional application Service to one Cloudflare pool endpoint.
- Watches only the relevant region-specific ready endpoints.
- Uses hysteresis to prevent endpoint flapping.
- Re-enables recovered origins and reports Cloudflare API errors.
- Runs with least-privilege Cloudflare API credentials.
- Leaves Cloudflare's independent health monitor enabled as a fallback.

## Verification checklist

- Non-members of Cluster Admins cannot access any regional cluster hostname.
- Cluster Admins can access the regional hostnames through Access.
- Workspace users outside Cluster Admins can access the Workspace-only app.
- Public apps remain public only on their explicitly intended hostnames.
- Direct public access to Traefik is blocked.
- Each regional tunnel reaches its regional Traefik.
- The public Load Balancer's endpoint Host overrides select the expected
  regional route.
- An application health failure causes the monitor to mark that region
  unhealthy after its configured threshold.
- A simulated Cloudflare connection/TLS failure retries another healthy pool
  endpoint with zero-downtime failover enabled.
- The cluster continues serving current workloads, and Cloudflare's independent
  monitor behaves as expected, during a control-plane-region outage.

## References

- [Cloudflare public load balancers for Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/public-load-balancers/)
- [Cloudflare endpoint Host header overrides](https://developers.cloudflare.com/load-balancing/additional-options/override-http-host-headers/)
- [Cloudflare adaptive routing](https://developers.cloudflare.com/load-balancing/understand-basics/adaptive-routing/)
- [Cloudflare zero-downtime failover reference](https://developers.cloudflare.com/reference-architecture/architectures/load-balancing/)
- [Cloudflare Load Balancing billing](https://developers.cloudflare.com/billing/understand/how-billing-works/)
- [Cloudflare Load Balancing query metering](https://developers.cloudflare.com/billing/understand/how-charges-accrue/)
- [Traefik FAQ: 502 and 503 responses](https://doc.traefik.io/traefik/getting-started/faq/)
