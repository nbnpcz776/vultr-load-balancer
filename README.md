# Vultr Load Balancer: What Does It Cost, How Do You Configure One, and What Can't It Do? (Same-Region Rules, VPC Firewalls, and the SSL Question)

Vultr Load Balancer starts at **$10.00 per subscription per month**, and the exact number depends on the data center you deploy into. Bandwidth isn't billed on the balancer at all; Vultr charges traffic on the instances behind it. Two constraints matter more than the price: the balancer and its backend instances must be in the same region, and Vultr's own FAQ states that the service does not support SSL termination.

What follows is what the $10 covers, where the rest of the bill actually comes from, the limits that only appear in the documentation, and the console sequence for a working setup with a private VPC and two layers of firewall.

> **Quick verdict:** if you run two or more instances in one region and want health checks plus automatic failover without operating HAProxy yourself, the single subscription is the entire product. There are no tiers to compare. 👉 [See the current Load Balancer price for your region](https://www.vultr.com/products/load-balancers?ref=9738262-9J)

## The whole product, on one line of the price list

Vultr doesn't publish a ladder of load balancer tiers the way it does for Cloud Compute. There is one subscription, priced from $10.00 per unit per month, and Vultr notes that pricing varies by data center location.

| Offering | What's included | Price | Billing | Get started |
| --- | --- | --- | --- | --- |
| Vultr Load Balancer (single subscription) | Fully managed Layer 4 TCP/UDP balancing; TCP, HTTP and HTTPS forwarding rules; configurable health checks and automatic failover; Round Robin or Least Connections; sticky sessions; Proxy Protocol; integrated firewall; optional non-public VPC network | From **$10.00/unit/month**, varies by region | Per subscription | [Check the price in your data center](https://www.vultr.com/products/load-balancers?ref=9738262-9J) |

Everything else is a configuration option rather than a purchase decision: health check protocol, port, path and interval; the algorithm; the number of nodes; forwarding rules; firewall rules; and an SSL section. Vultr manages the underlying infrastructure, scaling and monitoring, which is what "fully managed" refers to on the product page.

## Where the real cost comes from

Bandwidth neutral is the phrase Vultr uses: "we only charge on bandwidth on the instances attached to the load balancer." Put plainly, a terabyte pushed through the balancer costs the same as a terabyte pushed to a single instance directly. The balancer adds $10 and nothing else on the traffic side.

The bill does grow in two other directions.

The first is pool size. Attached instances bill at normal Cloud Compute rates, so a three-server pool costs three instances plus the $10 subscription. That's not a hidden fee, but it's the number people forget when they compare a load-balanced setup against a single bigger instance.

The second is geography. Because a balancer can only serve instances in its own region, running a multi-region deployment means one balancer per region. Three regions is three subscriptions, each priced at that location's rate. Vultr's product page markets "Global Load Balancers" that route traffic to the nearest server worldwide, and the provisioning doc lets you select one or more deployment locations. Neither of those overrides the regional binding: you're creating several balancers, not one balancer spanning continents.

👉 [Compare the subscription cost against the instance pricing in your region](https://www.vultr.com/products/load-balancers?ref=9738262-9J)

## Six limits that decide whether it fits

These come from the FAQ, the provisioning doc and the VPC tutorial. None of them appear on a comparison page, and each one can invalidate a design after you've already built it.

**Same region, no exceptions.** Vultr's FAQ answers "can I use my load balancer in one region with instances in a different region?" with a flat no. Instances and balancer must match.

**Node count has to be odd.** The provisioning form takes a Number of Nodes value that must be an odd number, so 3, 5, 7 rather than 2 or 4.

**The balancer eats three VPC IPs.** In the VPC tutorial, Vultr's own Networking tab reports that the balancer consumes three IP addresses from the VPC range for itself. Size your subnet with that overhead in mind, particularly on a small /24 you're also carving up for instances.

**Proxy Protocol needs a matching backend.** Turning it on forwards the client's original IP to backend nodes, and Vultr's doc warns you to configure the nodes to accept Proxy Protocol headers. If you skip that step, health checks and access logs start behaving strangely because the backend is parsing a header it wasn't told to expect.

**HTTP/3 depends on HTTP/2.** The provisioning doc requires at least one HTTPS forwarding rule before you enable HTTP/2 or HTTP/3, and HTTP/3 only works if an HTTP/2 rule already exists.

**No SSL termination, per the FAQ.** More on this below, because the documentation isn't unanimous.

There's also a cap on forwarding rules. Vultr's docs point to a separate feature reference for the limit rather than stating a number inline, so check it before you plan on dozens of port mappings.

## TCP, HTTP, HTTPS, and the SSL question

Vultr's FAQ describes the service twice in slightly different language: "support TCP, HTTP, and HTTPS" in one answer, "Layer 4 (TCP/UDP) load balancers, which operate at the transport layer" in another. The practical reading is a Layer 4 balancer that forwards on IP and port, with TCP, HTTP and HTTPS as the available forwarding-rule protocols. Don't buy it expecting path-based routing, host-header rules or content-aware decisions; the docs don't list them.

SSL is the part to verify yourself before committing. Three official sources say three different things:

- The product page advertises "SSL configuration options by allowing you to bring your own SSL certificates."
- The FAQ states plainly that load balancers "do not support SSL termination" and that SSL has to be managed at the application or server level.
- The 2024 provisioning doc includes an SSL section and a Force HTTP to HTTPS toggle, which requires at least one HTTPS forwarding rule and a valid certificate to work.

The conservative reading that satisfies all three: you can upload a certificate and use HTTPS forwarding rules, including the HTTP-to-HTTPS redirect, but TLS is not terminated at the balancer in the way a Layer 7 appliance would do it. Plan on terminating TLS on your instances or at a CDN sitting in front. Before you order, open the console in your target region and confirm what the SSL section offers there; the documentation hasn't been updated since September 2024.

👉 [Read the forwarding and SSL options on the product page](https://www.vultr.com/products/load-balancers?ref=9738262-9J)

## Setting one up: the fields you'll actually fill in

The path is **Products → Load Balancers → Create Load Balancer**. The form groups into four areas.

**Basics.** Label, Algorithm (Round Robin is the default, Least Connections is the alternative), Timeout in seconds, and Number of Nodes as an odd number.

**The three toggles.** Force HTTP to HTTPS redirects all HTTP traffic; it needs an HTTPS forwarding rule and a valid certificate. Proxy Protocol passes the client's real IP through. Sticky Sessions pins a client to one backend server.

**Health Checks.** Protocol, Port, Path, and Health Check Intervals. This is what drives automatic failover: traffic leaves an unhealthy instance and lands on a healthy one without you touching DNS.

**Load Balancer Configuration.** Expand the sections you need: Non-Public VPC Network, Forwarding Rules, Firewall Rules, SSL.

Click create, then attach servers. Open the balancer, click **Add Instance**, pick an instance from the dropdown, click **Attach Instance**, and repeat for each server. Adding or removing instances later rebalances traffic automatically, which is also how you'd run blue-green or canary releases: shift the backend pool, let health checks confirm the new version, then remove the old nodes.

Automation paths exist for all of it. Vultr documents provisioning through the Console, API, CLI and Terraform, and the product page notes integration with Terraform or Ansible workflows.

## Two firewalls, one reason

The VPC tutorial's architecture is worth copying because it closes the usual gap: web servers listening on the VPC only, SSH on the public interface, and no HTTP exposure to the open internet.

On the balancer, add a firewall rule for port 80 with the source set to `cloudflare`, so only your CDN can reach the balancer. On the instances, create a separate Vultr Firewall Group that allows SSH from a limited set of addresses and HTTP only from the balancer. Vultr's docs cover using a Load Balancer as an IP source in the Vultr Firewall, which is what makes the second rule possible.

Then point DNS at the balancer's IP. The tutorial uses Cloudflare: the CDN accelerates static assets, the balancer provides high availability, and the instances never accept a direct public connection. That's more moving parts than a single server, and it's also the configuration that survives one instance dying at 3am.

## When Vultr Load Balancer is the right call, and when it isn't

**Good fit:** two or more instances in one region that need health-checked failover; horizontal scaling where you add and remove backends by hand or by API; teams already using Terraform or Ansible who want the balancer in the same version-controlled workflow; microservice or API backends that speak plain TCP; gaming servers and database endpoints, since Vultr states compatibility with any TCP or UDP application.

**Poor fit:** a single balancer in front of instances spread across regions (build one per region instead); TLS termination at the balancer as an architectural requirement; Layer 7 routing on paths or headers, which the documentation doesn't list; anyone who needs published concurrency or throughput ceilings before buying, because Vultr doesn't publish those figures for this product.

## The $300 credit, and what to confirm before you order

Vultr's coupons page advertises several new-account credit tiers along with a deposit match of up to $100. Users on r/Vultr report that the promotional balance expires roughly 30 days after it's added, that it's limited to new accounts, and that it can't be combined with other offers. Treat it as a trial window rather than a discount: check Billing → Credits right after signup, note the expiry date, and only deploy what you'll actually evaluate in that period. The credit also applies to select products, so confirm the load balancer subscription is covered before you assume it's free for a month.

Two things to confirm in the console before you pay: the actual monthly rate for the data center you're deploying into, since the $10 is a starting figure, and whether the SSL section in that region supports what your architecture needs.

If both check out, the decision is simple. One region, multiple instances, health-checked failover, $10 on top of the compute you're already paying for.

👉 [Start with a Load Balancer subscription and see your regional quote](https://www.vultr.com/products/load-balancers?ref=9738262-9J)
