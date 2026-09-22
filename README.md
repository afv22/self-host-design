# How I Self-Host [WIP]

## Design Principles

### Everything is declared

#### The Constraint

#### The Decision

#### The Tradeoff

### Nothing is exposed

#### The Constraint

As privacy was a key motivator for self-hosting, I needed a cohesive solution for secure networking. Personal devices had to access services hosted at home and in the cloud while ensuring that those services were not exposed to the open internet. At the host level, no ports should be published without both an authorization mechanism and encryption.

#### The Decision

All devices are connected to a single Tailnet and assigned to a trust tier: personal devices (phone, laptop) which can SSH into each device and access each service, runners (servers, VMs) which are restricted to accessing critical services like logging, and edge nodes (ephemeral cloud VMs) which have no internal access and can only be used as exit nodes.

All services are bound to loopback addresses and exposed as Tailscale Services within the private network, which enables ACL, encryption, and DNS resolution. No host listens on a public interface.

#### The Tradeoff

Strict refusal to publish any external interfaces means that I can't use webhooks for real-time triggers or alerts. This was mitigated by configuring my services to use soon-enough polling instead, balancing acceptable lag time with noisy requests and power usage.

<div style="text-align: center;">
  <img
    src="assets/tailnet_trust_tiers.svg"
    alt="Tailnet trust tiers"
    style="max-width: 600px;"
  />
</div>

### Assume it will fail

#### The Constraint

#### The Decision

#### The Tradeoff
