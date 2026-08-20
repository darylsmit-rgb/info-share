# Palette / VerteX + AWS — split-horizon DNS preflight

**Purpose:** a set of questions to answer *together* before we sit down for the
engagement, so we can pick the right long-term DNS pattern for reaching your
self-hosted Palette / VerteX management plane from tenant / workload clusters.
The wrong answer here doesn't cause install-time failures — it causes tenant
clusters to become silently unreachable weeks later when a load-balancer IP
rotates, a CASB signature updates, or a network team makes a routine change.

Everything in this folder assumes:

- The mgmt-plane is **self-hosted** (not SaaS) inside an AWS account.
- Tenant clusters live in AWS accounts that need to reach it — the mgmt-plane
  agent on each tenant cluster ("`cluster-management-agent`" / CMA) makes
  outbound HTTPS to the mgmt-plane's rootDomain to phone home.
- Something about the network path is not "public DNS + open Internet" — either
  the rootDomain isn't in public DNS at all, or a security appliance sits
  between tenant egress and the mgmt-plane IP.

## Two artifacts alongside this README

Both are drop-in samples — they contain no environment-specific values. Fill in
the placeholders (marked `<...>`) for your environment.

| File | What it is |
|---|---|
| `coredns-phonehome.yaml` | A single-file "Add Manifest" layer for any Palette cluster profile. It patches `kube-system/coredns` on the tenant cluster to add a `hosts { }` block resolving the mgmt-plane rootDomain to a specific IP. Idempotent, sentinel-bracketed, refuses to run if placeholders weren't filled in. |
| `example-palette-cluster-profile.tf` | A minimal Terraform example showing how to authenticate to the Palette API and author a cluster profile that includes the CoreDNS phone-home layer conditionally (only when `mgmt_target_ip` is set). |

## The three failure shapes we're solving for

Any of these produces the same customer-visible symptom — tenant clusters
provision successfully in the UI, then never transition to `Running` because
CMA can't reach the mgmt-plane. Root cause differs though:

1. **`.local` (or any non-routable) rootDomain.** Sandbox-style installs
   sometimes use a rootDomain that isn't in any DNS zone the tenant cluster
   can see. Symptom: CMA logs `no such host` on the rootDomain.

2. **CASB / SASE / SSL-inspection interception.** A corporate proxy
   (Netskope, Zscaler, Palo Alto Prisma Access, iBoss, Cisco Umbrella,
   Forcepoint, Menlo, ...) intercepts outbound HTTPS to the mgmt-plane's
   public ELB IP and returns its own HTML/redirects to the CMA. Symptom:
   HTTP 303 or 200 with `content-type: text/html`, `Server:` or `Via:`
   headers naming the vendor. This is the failure mode we most often see
   with enterprise customers.

3. **Mgmt-plane sits behind a private-only network path.** The rootDomain
   resolves via public DNS to a public ELB IP, but the tenant VPC has no
   Internet egress (or has egress restricted from the destination IP). The
   real answer here is different — you need private-network reachability,
   not just DNS.

`coredns-phonehome.yaml` is a **workaround** for shapes 1 and 2. It patches
tenant CoreDNS so the rootDomain resolves to whatever IP you specify (public
ELB, NAT EIP, private endpoint, ...). It does not solve shape 3 — no amount of
CoreDNS surgery routes traffic that has nowhere to go.

## Questions to answer BEFORE the meeting

We want to leave the meeting with a chosen pattern, not with "we'll figure out
DNS later." Please come with answers (or "unsure — let's decide together")
for as many of these as you can.

### A. What does the mgmt-plane rootDomain look like today?

1. What's the exact hostname? Public FQDN, corporate-internal FQDN,
   `.local`, IP-in-a-hostname?
2. Is it registered in **any** public DNS zone? (`dig +short` from outside your
   corporate network — no answer means "no public DNS".)
3. Which DNS zone hosts it *internally* today? (Route53 private hosted zone,
   AWS-managed AD DNS, on-prem Windows DNS forwarded to AWS, other?)
4. Does the mgmt-plane sit on a **public** ELB/NLB/ALB, a **private**
   internal load balancer, or a direct EC2/instance IP? (Both is possible
   with two listeners — we care about which one tenants are meant to hit.)

### B. Where do the tenant clusters live relative to the mgmt-plane?

5. Same AWS account, same VPC as the mgmt-plane?
6. Same AWS account, **different** VPC?
7. **Different** AWS account (peered? Transit Gateway? PrivateLink?
   Internet-only?)
8. On-prem or another cloud, reaching AWS via VPN / Direct Connect / SD-WAN?

### C. What's in the network path?

9. Do tenant clusters have unrestricted egress to the Internet? Or does
   traffic exit through a NAT + firewall / proxy stack?
10. Do you run a CASB or SSL-inspection layer (Netskope, Zscaler, Prisma
    Access, iBoss, Cisco Umbrella, Forcepoint, Menlo, other)? If yes:
    which vendor, and can specific hostnames be added to a bypass /
    "do-not-decrypt" list?
11. Is there a corporate DNS layer (Infoblox, BIND, Windows DNS)
    that tenant clusters resolve against, either directly or via Route53
    Outbound Resolver?

### D. AWS-native DNS solutions we should evaluate first

The CoreDNS phone-home YAML is a workaround. Where possible, we prefer to
solve the problem at the AWS layer instead — it's less magic, doesn't require
a `kube-system` mutation on every tenant cluster, and doesn't rot when a load
balancer's IP changes. Which of these are viable in your environment?

12. **Route53 private hosted zone associated with tenant VPCs.** Create a
    private hosted zone for the mgmt-plane rootDomain, put an ALIAS/CNAME
    to the mgmt-plane's internal LB, and associate the zone with every
    VPC that will host a tenant cluster. Tenant CoreDNS then resolves
    through VPC-provided DNS naturally. Requires:
    - Willingness to run the mgmt-plane behind an INTERNAL load balancer
      (or dual-listener), and
    - Cross-account zone association if tenants are in different accounts
      (`aws route53 create-vpc-association-authorization` on the zone's
      account, `associate-vpc-with-hosted-zone` from the tenant's).
13. **VPC peering / Transit Gateway + `enableDnsSupport` + `enableDnsHostnames`.**
    If tenant VPCs already peer to the mgmt-plane VPC (or share a Transit
    Gateway), VPC-provided DNS can resolve the internal LB name across
    the peering. Cheaper than PrivateLink for a small number of tenants.
14. **AWS PrivateLink (VPC Endpoint Service).** The mgmt-plane exposes an
    NLB behind an Endpoint Service; tenant VPCs create VPC Endpoints for
    it. DNS is via a private hosted zone AWS creates for you. Best when
    tenants and mgmt-plane are in different accounts and you don't want
    to expose an internal LB across peering.
15. **Route53 Resolver — Outbound endpoint + forwarding rule.** Tenant VPCs
    forward specific domains (the mgmt-plane rootDomain) to an on-prem or
    corporate resolver that already knows the answer. Useful when your
    company DNS is already the source of truth.
16. **Route53 Resolver — Inbound endpoint.** Reverse of the above — your
    on-prem resolvers forward the mgmt-plane rootDomain to an Inbound
    Endpoint in the mgmt-plane VPC. Useful for hybrid tenants (on-prem
    workers registering to a cloud mgmt-plane).

### E. Deployment logistics

17. Is the mgmt-plane already installed, or are we standing it up as part of
    this engagement? (If already installed, changing the rootDomain post-hoc
    is expensive — we may be locked into a workaround.)
18. Who owns Route53 in your org — the same team we're engaging with, or a
    separate central platform / network team? (Determines lead time on
    creating zones or associations.)
19. Is a **change management window** required to modify tenant VPC DNS
    settings, or can we make changes ad-hoc during the engagement?
20. If we end up needing the CoreDNS workaround, does the tenant cluster
    profile go through GitOps / a review process, or can we apply it
    directly through the Palette UI?

## Decision matrix (short version)

| If your environment... | Best pattern |
|---|---|
| Tenant VPCs already peer to mgmt-plane VPC | Route53 private hosted zone associated with tenant VPCs → internal LB (#12 + #13) |
| Tenants are in different AWS accounts, no peering yet | AWS PrivateLink (#14) — cleanest cross-account story |
| Corporate DNS is the source of truth (`corp.example.com`), rootDomain lives there | Route53 Resolver Outbound endpoint + forwarding rule (#15) |
| Sandbox / demo with public ELB + `.local` rootDomain | CoreDNS phone-home layer (`coredns-phonehome.yaml`) — acceptable *for a sandbox*, not production |
| Production install and a CASB is intercepting | CASB bypass rule for the rootDomain + rely on public DNS + real cert — CoreDNS phone-home layer is the fallback if that can't be arranged |
| Mgmt-plane has no private network path from tenants | Stop — this is a network-architecture question, not DNS. Solve reachability first. |

## Why we prefer AWS-native over the CoreDNS workaround

1. **It survives infrastructure changes.** Route53 records automatically follow
   a load balancer's DNS name; CoreDNS `hosts { }` blocks pin an IP that
   rotates without warning.
2. **It doesn't mutate `kube-system` on every tenant cluster.** Anything in
   `kube-system` is fragile — cluster upgrades, add-on reconciliations, and
   security policies (Kyverno, VAP) all have opinions about it.
3. **It composes with existing corporate DNS.** Most enterprises already have
   a story for split-horizon DNS; Route53 fits into that story. The CoreDNS
   hack does not.
4. **It works for non-Palette workloads too.** Any pod on any tenant cluster
   that needs to reach the mgmt-plane rootDomain will resolve it correctly,
   not just CMA.

The CoreDNS workaround is documented here because sometimes the AWS-native
answer isn't available on the timeline you have. When that's the case, use it
deliberately — knowing what you're trading off.

## What to send back to us before the meeting

A short reply with your answers to as many of A–E as you can, plus **an
architecture drawing** (whiteboard photo is fine) showing:

- Mgmt-plane VPC, its LB(s), and the rootDomain.
- Two representative tenant VPCs (same-account + cross-account if that's
  your topology).
- Any peering / Transit Gateway / PrivateLink / VPN links between them.
- Where your CASB / proxy sits, if any.

That's enough for us to arrive with a proposed pattern and validate it in
real time, instead of leaving with an action item.
