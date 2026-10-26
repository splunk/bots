# Boss of the SOC (BOTS) Version 5: Pre-indexed Dataset Recovery

BOTS v5 consists of **six separate scenario datasets**. Each scenario is distributed as its own pre-indexed bucket archive and is restored into its own set of dedicated indexes on Splunk Enterprise 10.x. The archives contain pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archives and check `splunkd.log` for errors. **TODO: confirm the Splunk version that produced the v5 buckets.**

## Scenarios

Each scenario has its own README with download links, index definitions, restore steps, and sourcetype inventory. Scenarios are independent: restore any one, or all six.

| Scenario | Summary | Index prefix | Restore guide |
| --- | --- | --- | --- |
| GCP | TODO: scenario title and summary | `botsv5-gcp-` | [gcp/README.md](gcp/README.md) |
| K8S | TODO: scenario title and summary | `botsv5-k8s-` | [k8s/README.md](k8s/README.md) |
| APT | TODO: scenario title and summary | `botsv5-apt-` | [apt/README.md](apt/README.md) |
| ESS | TODO: scenario title and summary | `botsv5-ess-` | [ess/README.md](ess/README.md) |
| RWI | TODO: scenario title and summary | `botsv5-rwi-` | [rwi/README.md](rwi/README.md) |
| AWS | TODO: scenario title and summary | `botsv5-aws-` | [aws/README.md](aws/README.md) |

## Index naming

Every scenario is restored into indexes named `botsv5-<scenario>-<index>`, such as `botsv5-gcp-main`, `botsv5-gcp-notable`, and `botsv5-gcp-risk`. **This restoration intentionally uses new logical index names** to avoid interfering with an existing installation's `main`, `notable`, or `risk` indexes, and to keep scenarios separate from one another. The AWS scenario includes additional indexes; see its README.

To search across everything you have restored:

```spl
| tstats count WHERE index=botsv5-* earliest=0 BY index
| sort index
```

KV Store, original configurations, apps, and add-ons are not included in any archive.

## Licensing

These archives contain **pre-indexed Splunk data**. Copying existing buckets does not itself re-ingest the original raw events. Splunk Enterprise licensing and any app/add-on licensing remain subject to their respective terms. Re-indexing exported raw events would be a different workflow.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.

## Scope

These guides document recovery of the original event buckets into dedicated destination indexes. They do not include the original KV Store, configuration files, saved searches, dashboards, data models, or required apps/add-ons. Those components may be needed to reproduce the complete original BOTS v5 experience.

## Authors

Written by Dave Herrald, Ryan Kovar, James Brodsky, John Stoner, Tom Smit, Tim Frazier, Lily "LLZ" Lee, Simon O'Brien, Ian Forrest, Michael Natkin, and Shannon Davis.
