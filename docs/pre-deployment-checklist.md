# Pre-deployment checklist

Complete this checklist before changing a production router.

## Scope and access

- Record the device model, RouterOS version, serial number, and maintenance
  window.
- Record every management path and the source subnets that must retain access.
- Open a second authenticated session and confirm out-of-band or local-console
  access where available.
- Identify routing, bridging, VLAN, DHCP, DNS, VPN, CAPsMAN, and monitoring
  dependencies that must remain available.

## Recovery artifacts

- Create a password-protected binary backup and store it outside the router.
- Create a text export for review and version comparison.
- Treat both files as secrets; exports can contain sensitive configuration even
  when credentials are hidden.
- Record the RouterOS version used to create the binary backup.

## Change review

- Enter Safe Mode before changing remote-access, interface, route, or firewall
  settings.
- Restrict management services to explicit administrative source networks.
- Disable only services confirmed as unused.
- Review firewall rules in order; document the purpose of each accept rule and
  ensure the final policy matches the deployment design.
- Confirm NTP, DNS, remote logging, and monitoring destinations are reachable.
- Plan firmware and RouterOS updates separately from access-control changes.

## Acceptance checks

- A new administrative session succeeds from every approved management path.
- Management access fails from an unapproved network.
- Required client traffic, routing, DNS, DHCP, VPN, and monitoring still work.
- Logs have correct timestamps and reach the expected collector.
- The saved export matches the reviewed post-change configuration.
