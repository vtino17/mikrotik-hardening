# Rollback runbook

Keep this runbook and recovery artifacts available without relying on the
router being reachable.

1. While still in Safe Mode, undo the last change or terminate the Safe Mode
   session abnormally to trigger automatic rollback.
2. If remote access is lost, switch to the documented out-of-band or local
   console path. Do not improvise a factory reset during an incident.
3. Prefer reverting the specific reviewed change. Use a text export only after
   checking it for device-specific interface and identity differences.
4. Restore a binary backup only to the intended device and a compatible
   RouterOS version. A binary backup can also restore device-specific values.
5. Re-run the acceptance checks and preserve logs describing the failure and
   recovery.

Never publish a backup or export in an issue, chat, or public repository.
