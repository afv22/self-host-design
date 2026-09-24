# How I Self-Host [WIP]

## Design Principles

### Everything is declared

#### The Situation

My servers were built up incrementally. Services were onboarded, tools were installed, and secrets were stashed wherever was convenient at the time. With nothing tracking my changes, the running state of each machine was the only record, and it could only be read by logging in and looking. Upgrading a program or rotating a key meant visiting every host and making the change by hand.

This ad-hoc management inevitably caused issues. Logging pipelines had inconsistent formats and the same CLI tool expected different arguments depending on where I ran it.

Worst of all, nothing described how to rebuild a server. Setting everything up again was dependant on me remembering everything to install.

I tried to solve this all with a custom server management system, which ended up being a better lesson in the complexities of multi-host management than it was a reliable tool. I needed instead to find a platform that allowed me to focus on designing the servers rather than maintaining the platform.

#### The Solution

A single Ansible project defines configurations for every service and host I run. Each logical service, whether a single Docker container or a multi-role log pipeline, links its dependencies, configurations, and secrets, allowing for a holistic definition of what is run and how each component fits together.

Secrets are tracked alongside the rest of the configuration to ensure that they are imported properly and backed up. To ensure that they aren't exposed, SOPS is used to encrypt secret values before they are committed.

Being able to back up the entire configuration, secrets and all, means that I can redeploy changes from anywhere, as long as I have access to my SOPS decryption key. With service state also backed up (see [Assume it will fail](#assume-it-will-fail)), onboarding a new or rebuilt host is straightforward.

```
# Sample Service - Actual Budget

## Service Files

inventory.yaml                  # which hosts will run it
playbooks/actual.yaml           # links the hosts to the role
roles/actual/
  defaults/main.yaml            # image version, ports, paths
  tasks/main.yaml               # container, backups, tailnet publish
  tasks/backup.yaml
  templates/actual-backup.*.j2  # systemd timer + export script
group_vars/actual/
  main.yaml                     # which budgets to back up
  secrets.sops.yaml             # encrypted server password

## roles/actual/tasks/main.yaml

- name: Create actual data dir
  become: true
  ansible.builtin.file:
    path: "{{ actual_data_dir }}"
    ...

- name: Start actual container
  community.docker.docker_container:
    name: actual
    volumes:
      - "{{ actual_data_dir }}:/actual"
    state: started
    ...

- name: Configure backups
  ansible.builtin.import_tasks: backup.yaml

- name: Publish as a tailscale service
  ansible.builtin.import_role:
    name: tailscale_service

## group_vars/actual/secrets.sops.yaml

actual_backup_password: ENC[AES256_GCM,data:UQfd...,type:str]

```

#### The Tradeoffs

Manual changes are either overwritten or unnoticed. Requires operator discipline to apply changes via the config rather than directly on the server.

State data is not included. Storage and backup strategies can be defined and deployed with Ansible, but the data must live elsewhere.

Initial machine setup must be handled separately. Setting up the OS, creating the user, and authenticating to the Tailnet must be done manually or with an image builder (e.g. Packer).

The SOPS decryption key is a key dependency. Per-host keys would limit a leak's blast radius to a single host's secrets, but at this scale that's not a meaningful security gain over disciplined single-key hygiene.

### Nothing is exposed

#### The Situation

I need access to my services and data while away from my home network in a way that ensures privacy and security.

My ISP gives me a dynamic address behind CGNAT, so inbound connections to home servers aren't reliably available. I'm also the only operator, so anything that requires routine upkeep (certs pipelines, etc.) will likely get neglected.

Previous experiences running everything privately on GCP and AWS were complex and expensive enough to be unsustainable. Commercial VPNs that support device-to-device access exist, but can have questionable data privacy practices.

#### The Solution

All devices are connected to a single Tailnet and assigned to one of three trust tiers:

- Personal devices (mobile, laptop): allowed to SSH into all runners and access all services.
- Runners (servers, VMs): limited to accessing core services (e.g. log sink) to enable standard operation while restricting abitrary traversal.
- Edge proxy nodes (ephemeral cloud VMs): restricted from accessing any internal resources as they are externally controlled and don't require any sensitive access.

Each service listens only on loopback. Reaching one from another device goes through the Tailnet; Tailscale terminates the connection on the host and forwards it to the local port. This route means every connection is encrypted, explicitly allowed by ACL rules, and addressed by a name that Tailscale resolves. No host listens on a public interface.

![Tailnet trust tiers](assets/tailnet_trust_tiers.svg)

#### The Tradeoffs

Tailscale is a hard dependency, though its mesh topology means authenticated devices can continue to securely communicate with each other while the central server is unavailable.

Zero public exposure means that I can't use webhooks for real-time triggers or alerts. This has been mitigated by configuring my services to use soon-enough polling, balancing latency against request volume and power draw.

Sharing a service with another person requires onboarding them to the Tailnet.

A compromised personal device can access everything.

### Assume it will fail

#### The Situation

#### The Solution

#### The Tradeoffs
