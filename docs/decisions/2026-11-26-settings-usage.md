# SETTINGS-001 Usage balances, Settings pagination and preset ownership

**Date** 2026-11-26. **Status** Pushed to API master in `d427409`. The MySQL-enabled API suite passed locally with 1,341 tests passing and one live voice-preview test skipped. Web integration and production verification are pending.

**Sources and origin** Requested in [issues 3, 6, 9 and 10](../../issues.md). The user explicitly chose API and handoff delivery and confirmed tenants see their own presets plus the 16 built-ins. This narrows the visibility of platform custom presets in the [AGENT-014 catalog](2026-11-22-shared-agent-configuration.md#agent-014-creation-time-call-presets-and-campaign-authored-instructions).

**Technical choices and consequences** Return signed, period-based AI and operator balances in seconds alongside usage. Add scoped database paging, literal search and stable sorting for Twilio inventory and industry classifiers. Keep the array response for existing picker requests without table parameters. Platform administrators can filter presets by exact tenant owner. Ownerless custom presets stay visible to platform administrators. Existing campaign snapshots retain their content. New campaign-created Collections scripts belong to the campaign tenant. No schema or data migration is needed.

The [API handoff](../../apps/api/docs/api/settings-usage-web-handoff.md) defines the client requests, result fields and rollout scope.
