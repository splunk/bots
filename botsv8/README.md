# Boss of the SOC (BOTS) Version 8: Pre-indexed Dataset Recovery

BOTS v8 has **three scenarios that share one dataset**. Unlike v5, v6, and v7, there is no separate archive per scenario. A single pre-indexed bucket archive supports all three scenarios, and it is restored into dedicated indexes on Splunk Enterprise 10.x. The archive contains pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Scenarios

The three BOTS v8 scenarios all run against the same restored data. Restore the dataset once, then use the questions for the scenario you want.

| Scenario | Title |
| --- | --- |
| APT | The Advanced Persistent Threat Scenario |
| ESS | The Enterprise Security Scenario |
| WAA | The Web Applications Scenario |

The Q&A zip contains a separate `.xls` file for each scenario.

## Download

The pre-indexed BOTS v8 dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v8 | Pre-indexed buckets shared by the APT, ESS, and WAA scenarios, restored as `botsv8-main`, `botsv8-battleship`, `botsv8-mc_aux_incidents`, `botsv8-notable`, `botsv8-risk` | `.tgz` archive | [Download BOTS v8 dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv8/botsv8.tgz) |
| BOTS v8 Q&A | Questions and answers for all BOTS v8 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v8 Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv8/botsv8-qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv8.tgz`. The archive is about 2.4 GB (about 3.5 GB extracted). Its SHA-256 checksum is `996508571793349744026154c957a1543091333798bff9bb1a0ea952ff56a1bf`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes. Because all three scenarios share one dataset, the indexes are named `botsv8-<index>` with no scenario in the name.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv8-main` | `$SPLUNK_DB/botsv8-main/` | 3.5 GB | 5 |
| `battleship/` | `botsv8-battleship` | `$SPLUNK_DB/botsv8-battleship/` | 3.3 MB | 6 |
| `mc_aux_incidents/` | `botsv8-mc_aux_incidents` | `$SPLUNK_DB/botsv8-mc_aux_incidents/` | 1.6 MB | 3 |
| `notabledb/` | `botsv8-notable` | `$SPLUNK_DB/botsv8-notable/` | 60 KB | 8 |
| `riskdb/` | `botsv8-risk` | `$SPLUNK_DB/botsv8-risk/` | 30 KB | 3 |

The `defaultdb/` archive directory is restored as `botsv8-main`, `notabledb/` as `botsv8-notable`, and `riskdb/` as `botsv8-risk`.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `battleship/`, `defaultdb/`, and `mc_aux_incidents/`) contain acceleration summaries and are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 3.5 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv8.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv8.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep -E '^[^/]+/db/db_[^/]+/$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `battleship/db/db_...`, `mc_aux_incidents/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`. The archive should contain 25 bucket directories. Its journals are zstd-compressed (`rawdata/journal.zst`) rather than `journal.gz`. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 996508571793349744026154c957a1543091333798bff9bb1a0ea952ff56a1bf
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv8-main]
homePath = $SPLUNK_DB/botsv8-main/db
coldPath = $SPLUNK_DB/botsv8-main/colddb
thawedPath = $SPLUNK_DB/botsv8-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv8-battleship]
homePath = $SPLUNK_DB/botsv8-battleship/db
coldPath = $SPLUNK_DB/botsv8-battleship/colddb
thawedPath = $SPLUNK_DB/botsv8-battleship/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv8-mc_aux_incidents]
homePath = $SPLUNK_DB/botsv8-mc_aux_incidents/db
coldPath = $SPLUNK_DB/botsv8-mc_aux_incidents/colddb
thawedPath = $SPLUNK_DB/botsv8-mc_aux_incidents/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv8-notable]
homePath = $SPLUNK_DB/botsv8-notable/db
coldPath = $SPLUNK_DB/botsv8-notable/colddb
thawedPath = $SPLUNK_DB/botsv8-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv8-risk]
homePath = $SPLUNK_DB/botsv8-risk/db
coldPath = $SPLUNK_DB/botsv8-risk/colddb
thawedPath = $SPLUNK_DB/botsv8-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv8-main botsv8-battleship botsv8-mc_aux_incidents botsv8-notable botsv8-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv8-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv8-stage
sudo find /tmp/botsv8-stage -type d -name 'db_*' | wc -l   # expect 25
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv8-main' 'battleship:botsv8-battleship' 'mc_aux_incidents:botsv8-mc_aux_incidents' 'notabledb:botsv8-notable' 'riskdb:botsv8-risk'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv8-stage/$srcname/$tier"
    dst="$SPLUNK_DB/$dstname/$tier"
    [ -d "$src" ] || continue

    sudo mkdir -p "$dst"
    if [ -n "$(sudo find "$dst" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
      echo "ERROR: destination is not empty: $dst" >&2
      exit 1
    fi
    sudo cp -a "$src/." "$dst/"
  done

  sudo mkdir -p "$SPLUNK_DB/$dstname/thaweddb"
done
```

Ensure the copied directories and files are owned by the Splunk service account. For example, **if Splunk runs as `splunk`**:

```bash
for idx in botsv8-main botsv8-battleship botsv8-mc_aux_incidents botsv8-notable botsv8-risk; do
  sudo chown -R splunk:splunk "$SPLUNK_DB/$idx"
done
```

The archive's `datamodel_summary/` directories are intentionally not copied. Do not import them as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv8-main earliest=0 BY sourcetype
| sort -count
```

```spl
| tstats count WHERE index=botsv8-* earliest=0 BY index
| sort index
```

Inspect registered buckets:

```spl
| dbinspect index=botsv8-*
| stats count AS buckets sum(eventCount) AS events BY index
```

If data does not appear, check logs before attempting another import:

```bash
grep -Ei 'bucket|tsidx|journal|corrupt|incompatible|freeze|frozen' \
  "$SPLUNK_HOME/var/log/splunk/splunkd.log" | tail -100
```

Do not delete the original archive or repeatedly restart against the only copy of the buckets. If Splunk 10.x rejects the files, an intermediate compatible Splunk version or another supported migration method may be necessary.

## 6. Verify retention

```bash
for idx in botsv8-main botsv8-battleship botsv8-mc_aux_incidents botsv8-notable botsv8-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. The same data serves all three scenarios. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv8-main`)

**57 sourcetypes**, 6,600,199 events.

| Sourcetype | Events |
| --- | ---: |
| `linux:audit` | 2,083,019 |
| `XmlWinEventLog` | 1,444,638 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 792,884 |
| `aws:cloudwatch` | 576,531 |
| `aws:cloudtrail` | 492,387 |
| `WindowsUpdateLog` | 318,378 |
| `stream:dns` | 267,931 |
| `stream:udp` | 128,837 |
| `stream:ip` | 101,730 |
| `linux:mail` | 98,382 |
| `stream:tcp` | 74,876 |
| `aws:elb:accesslogs` | 66,991 |
| `statistics` | 56,846 |
| `stream:http` | 23,286 |
| `postfix_syslog` | 11,185 |
| `linux_secure` | 9,636 |
| `Script:ListeningPorts` | 8,151 |
| `syslog` | 7,755 |
| `sssd` | 7,388 |
| `azure:aad:signin` | 5,115 |
| `sfdc:logfile` | 4,191 |
| `cron-3` | 3,005 |
| `powershell` | 2,574 |
| `Powershell:ScriptExecutionSummary` | 1,842 |
| `measurements-too_small` | 1,625 |
| `statistics-too_small` | 1,581 |
| `slack:messages` | 1,142 |
| `Script:InstalledApps` | 1,091 |
| `ms:iis:splunk` | 778 |
| `cloud-init-3` | 736 |
| `sfdc:contentversion` | 716 |
| `OktaIM2:log` | 710 |
| `sfdc-streaming-api-events` | 708 |
| `tracking-too_small` | 502 |
| `MSAD:NT6:Health` | 477 |
| `Powershell:ScriptExecutionErrorRecord` | 444 |
| `measurements-2` | 401 |
| `statistics-2` | 400 |
| `slack:audit` | 245 |
| `Script:TimesyncConfiguration` | 202 |
| `Script:TimesyncStatus` | 202 |
| `linux_bootlog` | 175 |
| `azure:aad:user` | 144 |
| `bash_history` | 66 |
| `ms:o365:email` | 66 |
| `Script:NetworkConfiguration` | 60 |
| `github:cloud:audit` | 48 |
| `amazon-ssm-agent` | 34 |
| `Script:GetEndpointInfo` | 29 |
| `dnf` | 29 |
| `hibernate` | 10 |
| `dnf.librepo` | 7 |
| `azure:aad:audit` | 4 |
| `ActiveDirectory` | 3 |
| `dnf.rpm` | 2 |
| `hawkey-too_small` | 2 |
| `sssd_implicit_files-too_small` | 2 |

### `index=battleship` (restored as `index=botsv8-battleship`)

**2 sourcetypes**, 99 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 52 |
| `comms` | 47 |

### `index=mc_aux_incidents` (restored as `index=botsv8-mc_aux_incidents`)

**1 sourcetype**, 51 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 51 |

### `index=notable` (restored as `index=botsv8-notable`)

**1 sourcetype**, 25 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 25 |

### `index=risk` (restored as `index=botsv8-risk`)

**1 sourcetype**, 15 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 15 |

## Licensing

This archive contains **pre-indexed Splunk data**. Copying existing buckets does not itself re-ingest the original raw events. Splunk Enterprise licensing and any app/add-on licensing remain subject to their respective terms. Re-indexing exported raw events would be a different workflow.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.

## Scope of this guide

This README documents recovery of the original event buckets into dedicated destination indexes. It does not include the original KV Store, configuration files, saved searches, dashboards, data models, or required apps/add-ons. Those components may be needed to reproduce the complete original BOTS v8 experience.

## Authors

Written in 2023 by Tom Smit, Shannon Davis, James Young, David Gamer, Simon O'Brien, Tim Hard, and Matthew Moore.
