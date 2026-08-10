# MikroTik Hardening

A review-first hardening guide for MikroTik RouterOS 7 deployments.

## Repository status

This repository currently contains operational guidance, not an executable
one-size-fits-all router configuration. Router interface names, management
subnets, routing, VPNs, and service dependencies differ between deployments;
applying an unreviewed generic script can lock out administrators or interrupt
traffic.

## Safe workflow

1. Inventory the device model, RouterOS version, interfaces, VLANs, routes,
   management sources, and required services.
2. Follow the [pre-deployment checklist](docs/pre-deployment-checklist.md).
3. Export the current configuration and create an encrypted backup.
4. Use RouterOS Safe Mode while applying changes interactively.
5. Validate management access from a second session before leaving Safe Mode.
6. Keep the [rollback runbook](docs/rollback.md) available offline.

## Scope

The checklist covers management-plane exposure, credentials, service
minimization, time and logging, update planning, firewall review, backup
handling, and rollback readiness. It deliberately avoids undocumented commands
and does not claim certification for a router it has not inspected.

## References

- [RouterOS configuration management and Safe Mode](https://help.mikrotik.com/docs/spaces/ROS/pages/328155/Configuration%2BManagement)
- [RouterOS backup guidance](https://help.mikrotik.com/docs/spaces/ROS/pages/40992852/Backup)

## License

MIT
