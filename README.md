# blog-bind-views-acls

Companion code for the blog post
[BIND Zone Security: Views, ACLs, and Query Restrictions](https://blog.bsdserver.nl/bind-zone-security-views-acls-and-query-restrictions/).

It builds the post's "Complete Secure Configuration Example" on one server:

- ACLs `trusted` and `secondary-ns`;
- options with recursion off by default, transfers and updates denied by default, version,
  hostname and server-id hidden, `minimal-responses` and `minimal-any`, response rate
  limiting (10/s, window 15, slip 2) and DNSSEC validation;
- view `internal` (clients in `trusted`): recursion for trusted clients and the internal
  `example.com` with `db` and `ldap`;
- view `external` (everyone else): no recursion and the public `example.com` without them;
- zone transfers in both views only with the TSIG key `transfer-key`;
- `named.log` and `security.log` in `/var/log/named/` (security, xfer-in and xfer-out).

The zone files are the post's internal and external `example.com` files.

## Differences from the post

- The post's example is written for RHEL (`/etc/named.conf`, `directory "/var/named"`,
  `named.ca`). On Debian/Ubuntu the role writes the same statements into
  `/etc/bind/named.conf.options` and `named.conf.local`, uses `/usr/share/dns/root.hints` for
  the hint zone and keeps zone files in `/var/lib/bind/zones/{internal,external}/`
  (RHEL: `/var/named/zones/...`).
- The key is in its own file (`/etc/bind/keys/transfer-key.key`, 0640 root:bind) and is
  included, instead of sitting in the main config.
- The distribution's default zones (localhost, reverse zones) are not included: with views,
  every zone has to be inside a view.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | One Ubuntu 24.04 VM on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/bind` | Installs BIND and writes `named.conf` with includes |
| `ansible/roles/bind_views` | ACLs, options, views, zone files, transfer key, logging |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM)
- ansible-core >= 2.15 (tested with 2.20.1)
- Target host: Ubuntu 24.04 (tested, BIND 9.18) or RHEL/Rocky 9 (handled, not lab-tested)

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible                                   # or: cp inventory.example.yml inventory.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# secret from `tsig-keygen -a hmac-sha256 transfer-key`, then:
ansible-vault encrypt group_vars/all/vault.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass

dig @192.168.1.10 +short db.example.com        # from an internal client: 192.168.1.200
dig @192.168.1.10 +short www.example.com       # from outside trusted: 203.0.113.100
```

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in `roles/bind_views/defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `bind_views_trusted` | localhost, localnets, 192.168.0.0/16, 10.0.0.0/8 | `acl "trusted"`, matched by the internal view |
| `bind_views_secondary_ns` | 192.168.1.10, 192.168.1.11 | `acl "secondary-ns"` |
| `bind_views_transfer_key` / `bind_views_transfer_secret` | `transfer-key` / vault | TSIG key for transfers |
| `vault_bind_views_transfer_secret` | none | The secret, in `group_vars/all/vault.yml` |
| `bind_views_rrl_*` | 10 / 15 / 2 | Rate limit: responses per second, window, slip |
| `bind_views_zone` / `bind_views_zone_serial` | `example.com` / `2024010101` | Zone served in both views |
| `bind_views_log_dir` | `/var/log/named` | Log directory |

Edit the zone templates in `roles/bind_views/templates/zones/` for your own records and bump
the serial.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (BIND 9.18.39); the second
run reports `changed=0`. Checks afterwards, from an internal address and from an address
outside `trusted`: internal clients see `www` as 192.168.1.100 plus `db` and `ldap` and get
recursion; external clients see 203.0.113.100, NXDOMAIN for `db`/`ldap`, and REFUSED for
recursion. `version.bind`, `hostname.bind` and `id.server` return "not disclosed",
`ANY` over UDP returns a single record, AXFR without the key fails, and with the key each
view transfers its own version of the zone (logged in `security.log`). ansible-lint
(production profile), yamllint, `terraform validate` and tflint are clean. The Terraform code
has not been applied against a real Proxmox cluster.

## License

MIT
