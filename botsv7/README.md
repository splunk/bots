# Boss of the SOC (BOTS) Version 7: Pre-indexed Dataset Recovery

BOTS v7 consists of **three separate scenario datasets**. Each scenario is distributed as its own pre-indexed bucket archive and is restored into its own set of dedicated indexes on Splunk Enterprise 10.x. The archives contain pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archives and check `splunkd.log` for errors.

## Scenarios

Each scenario has its own README with download links, index definitions, and restore steps. Scenarios are independent: restore any one, or all three.

| Scenario | Title | Index prefix | Restore guide |
| --- | --- | --- | --- |
| ESS | The Enterprise Security Scenario | `botsv7-ess-` | Not yet available |
| APT | The Advanced Persistent Threat Scenario | `botsv7-apt-` | Not yet available |
| SIM | The Splunk Infrastructure Migration Scenario | `botsv7-sim-` | Not yet available |

## Index naming

Every scenario is restored into indexes named `botsv7-<scenario>-<index>`, such as `botsv7-ess-main` or `botsv7-ess-notable`. **This restoration intentionally uses new logical index names** to avoid interfering with an existing installation's indexes, and to keep scenarios separate from one another. Some scenarios include several additional indexes; see each scenario's README.

To search across everything you have restored:

```spl
| tstats count WHERE index=botsv7-* earliest=0 BY index
| sort index
```

KV Store, original configurations, apps, and add-ons are not included in any archive.

## Licensing

These archives contain **pre-indexed Splunk data**. Copying existing buckets does not itself re-ingest the original raw events. Splunk Enterprise licensing and any app/add-on licensing remain subject to their respective terms. Re-indexing exported raw events would be a different workflow.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.

## Scope

These guides document recovery of the original event buckets into dedicated destination indexes. They do not include the original KV Store, configuration files, saved searches, dashboards, data models, or required apps/add-ons. Those components may be needed to reproduce the complete original BOTS v7 experience.

## Authors

Written in 2022 by Tom Smit, Simon O'Brien, John Stoner, Drew Church, Shannon Davis, and James Young.
