# BOTS v6 FIT: Pre-indexed Dataset Recovery

This guide restores the BOTS v6 FIT scenario into 7 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v6 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v6 FIT dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v6 FIT | Pre-indexed buckets, restored as `botsv6-fit-main`, `botsv6-fit-audit`, `botsv6-fit-dtex`, `botsv6-fit-frothly_events`, `botsv6-fit-notable`, `botsv6-fit-risk`, `botsv6-fit-ueba` | `.tgz` archive | [Download BOTS v6 FIT dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv6/botsv6-fit.tgz) |
| BOTS v6 Q&A | Questions and answers for all BOTS v6 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v6 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv6/BOTSv6qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv6-fit.tgz`. The archive is about 2.1 GB (about 3.8 GB extracted). Its SHA-256 checksum is `328d524e240ba464b5c7e511c22e2ae756db2ab5a23f0d73a62b26f62eba29ae`.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv6-fit-main` | `$SPLUNK_DB/botsv6-fit-main/` | 2.6 GB | 5 |
| `audit/` | `botsv6-fit-audit` | `$SPLUNK_DB/botsv6-fit-audit/` | 1.2 GB | 79 |
| `dtex/` | `botsv6-fit-dtex` | `$SPLUNK_DB/botsv6-fit-dtex/` | 20 MB | 2 |
| `frothly_events/` | `botsv6-fit-frothly_events` | `$SPLUNK_DB/botsv6-fit-frothly_events/` | 0.7 MB | 1 |
| `oldnotabledb/` | `botsv6-fit-notable` | `$SPLUNK_DB/botsv6-fit-notable/` | 160 KB | 2 |
| `oldriskdb/` | `botsv6-fit-risk` | `$SPLUNK_DB/botsv6-fit-risk/` | 280 KB | 3 |
| `uebadb/` | `botsv6-fit-ueba` | `$SPLUNK_DB/botsv6-fit-ueba/` | 30 KB | 4 |

The `defaultdb/` archive directory is restored as `botsv6-fit-main`. The `old` prefix is dropped from `oldnotabledb/` and `oldriskdb/`, and the `db` suffix from `uebadb/`, so they are restored as `botsv6-fit-notable`, `botsv6-fit-risk`, and `botsv6-fit-ueba`.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `audit/`, `defaultdb/`, `dtex/`, and `frothly_events/`) contain acceleration summaries, and its `summary/` directories contain summary data. Neither is imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 3.8 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv6-fit.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv6-fit.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `audit/db/db_...`, `dtex/db/db_...`, `frothly_events/db/db_...`, `oldnotabledb/db/db_...`, `oldriskdb/db/db_...`, and `uebadb/db/db_...`. The archive should contain 96 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 328d524e240ba464b5c7e511c22e2ae756db2ab5a23f0d73a62b26f62eba29ae
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv6-fit-main]
homePath = $SPLUNK_DB/botsv6-fit-main/db
coldPath = $SPLUNK_DB/botsv6-fit-main/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-audit]
homePath = $SPLUNK_DB/botsv6-fit-audit/db
coldPath = $SPLUNK_DB/botsv6-fit-audit/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-audit/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-dtex]
homePath = $SPLUNK_DB/botsv6-fit-dtex/db
coldPath = $SPLUNK_DB/botsv6-fit-dtex/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-dtex/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-frothly_events]
homePath = $SPLUNK_DB/botsv6-fit-frothly_events/db
coldPath = $SPLUNK_DB/botsv6-fit-frothly_events/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-frothly_events/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-notable]
homePath = $SPLUNK_DB/botsv6-fit-notable/db
coldPath = $SPLUNK_DB/botsv6-fit-notable/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-risk]
homePath = $SPLUNK_DB/botsv6-fit-risk/db
coldPath = $SPLUNK_DB/botsv6-fit-risk/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-fit-ueba]
homePath = $SPLUNK_DB/botsv6-fit-ueba/db
coldPath = $SPLUNK_DB/botsv6-fit-ueba/colddb
thawedPath = $SPLUNK_DB/botsv6-fit-ueba/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv6-fit-main botsv6-fit-audit botsv6-fit-dtex botsv6-fit-frothly_events botsv6-fit-notable botsv6-fit-risk botsv6-fit-ueba; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv6-fit-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv6-fit-stage
sudo find /tmp/botsv6-fit-stage -name journal.gz -type f | wc -l   # expect 96
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv6-fit-main' 'audit:botsv6-fit-audit' 'dtex:botsv6-fit-dtex' 'frothly_events:botsv6-fit-frothly_events' 'oldnotabledb:botsv6-fit-notable' 'oldriskdb:botsv6-fit-risk' 'uebadb:botsv6-fit-ueba'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv6-fit-stage/$srcname/$tier"
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
for idx in botsv6-fit-main botsv6-fit-audit botsv6-fit-dtex botsv6-fit-frothly_events botsv6-fit-notable botsv6-fit-risk botsv6-fit-ueba; do
  sudo chown -R splunk:splunk "$SPLUNK_DB/$idx"
done
```

The archive's `datamodel_summary/` and `summary/` directories are intentionally not copied. Do not import them as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv6-fit-* earliest=0 BY index
| sort index
```

```spl
| tstats count WHERE index=botsv6-fit-* earliest=0 BY index, sourcetype
| sort index, -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv6-fit-*
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
for idx in botsv6-fit-main botsv6-fit-audit botsv6-fit-dtex botsv6-fit-frothly_events botsv6-fit-notable botsv6-fit-risk botsv6-fit-ueba; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv6-fit-main`)

**95 sourcetypes**, 5,682,715 events.

| Sourcetype | Events |
| --- | ---: |
| `cp_log` | 2,942,654 |
| `XmlWinEventLog` | 1,614,149 |
| `osquery:info` | 178,054 |
| `sfdc:logfile` | 103,728 |
| `osquery:results` | 74,163 |
| `sfdc-streaming-api-events` | 69,289 |
| `gapps:admin:directory:users` | 49,560 |
| `WindowsUpdateLog` | 44,520 |
| `gmail:audit:ta:output` | 38,727 |
| `stream:ip` | 35,810 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 34,651 |
| `XmlWinEventLog:Microsoft-Windows-Powershell/Operational` | 30,710 |
| `Script:ListeningPorts` | 29,448 |
| `osquery:warning` | 27,560 |
| `osquery:error` | 26,497 |
| `gapps:report:chat:modular_input_result` | 21,243 |
| `sfdc:loginhistory` | 19,922 |
| `gapps:usage:user:api` | 18,911 |
| `gapps:usage:customer:api` | 16,046 |
| `gapps:report:access_transparency:modular_input_result` | 14,162 |
| `gapps:report:admin:modular_input_result` | 14,162 |
| `gapps:report:calendar:modular_input_result` | 14,162 |
| `gapps:report:drive:modular_input_result` | 14,162 |
| `gapps:report:gcp:modular_input_result` | 14,162 |
| `gapps:report:gplus:modular_input_result` | 14,162 |
| `gapps:report:groups:modular_input_result` | 14,162 |
| `gapps:report:groups_enterprise:modular_input_result` | 14,162 |
| `gapps:report:jamboard:modular_input_result` | 14,162 |
| `gapps:report:login:modular_input_result` | 14,162 |
| `gapps:report:meet:modular_input_result` | 14,162 |
| `gapps:report:mobile:modular_input_result` | 14,162 |
| `gapps:report:rules:modular_input_result` | 14,162 |
| `gapps:report:saml:modular_input_result` | 14,162 |
| `gapps:report:token:modular_input_result` | 14,162 |
| `gapps:report:user_accounts:modular_input_result` | 14,162 |
| `stream:dns` | 11,776 |
| `gmail:enforce:audit:ta:output` | 8,646 |
| `gapps:chrome:api` | 7,081 |
| `illumio:pce` | 6,740 |
| `Powershell:ScriptExecutionSummary` | 5,976 |
| `stream:tcp` | 5,769 |
| `MSAD:NT6:Health` | 5,035 |
| `stream:udp` | 4,616 |
| `stream:http` | 2,592 |
| `Script:InstalledApps` | 2,090 |
| `Unix:Service` | 2,074 |
| `ps` | 1,949 |
| `MSAD:NT6:DNS-Zone-Information` | 1,912 |
| `st_frothly_events` | 1,592 |
| `ActiveDirectory` | 1,298 |
| `GSuiteForSplunk:error` | 1,088 |
| `interfaces` | 975 |
| `netstat` | 975 |
| `protocol` | 975 |
| `top` | 975 |
| `Powershell:ScriptExecutionErrorRecord` | 779 |
| `osquery:snapshots` | 747 |
| `MSAD:NT6:SiteInfo` | 742 |
| `Script:TimesyncConfiguration` | 698 |
| `Script:TimesyncStatus` | 698 |
| `Unix:UserAccounts` | 595 |
| `MSAD:NT6:DNS-Health` | 239 |
| `XmlWinEventLog:Microsoft-Windows-PrintService/Operational` | 238 |
| `df` | 194 |
| `lastlog` | 194 |
| `openPorts` | 194 |
| `gmail:audit:headers` | 192 |
| `WinPrintMon` | 183 |
| `Unix:ListeningPorts` | 151 |
| `lsof` | 97 |
| `Script:NetworkConfiguration` | 87 |
| `gapps:usage:user` | 55 |
| `gapps:report:drive` | 36 |
| `config_file` | 33 |
| `Script:GetEndpointInfo` | 30 |
| `who` | 30 |
| `Unix:SSHDConfig` | 17 |
| `illumio:pce:metadata` | 17 |
| `package` | 17 |
| `usersWithLoginPrivs` | 17 |
| `gsuite:directory:user` | 16 |
| `gapps:report:mobile` | 13 |
| `gapps:report:token` | 8 |
| `gapps:usage:customer` | 8 |
| `alternatives.log-too_small` | 3 |
| `gapps:report:calendar` | 3 |
| `script::checkpcsversions.sh` | 3 |
| `sfdc:account` | 3 |
| `gapps:report:login` | 2 |
| `hardware` | 2 |
| `sfdc:user` | 2 |
| `Unix:Uptime` | 1 |
| `Unix:Version` | 1 |
| `XmlWinEventLog:Microsoft-Windows-PrintService/Admin` | 1 |
| `gapps:report:user_accounts` | 1 |

### `index=dtex` (restored as `index=botsv6-fit-dtex`)

**2 sourcetypes**, 16,596 events.

| Sourcetype | Events |
| --- | ---: |
| `dtex_st_activities` | 16,093 |
| `dtex_st_alerts` | 503 |

### `index=notable` (restored as `index=botsv6-fit-notable`)

**1 sourcetype**, 63 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 63 |

### `index=risk` (restored as `index=botsv6-fit-risk`)

**1 sourcetype**, 128 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 128 |

### `index=ueba` (restored as `index=botsv6-fit-ueba`)

**1 sourcetype**, 14 events.

| Sourcetype | Events |
| --- | ---: |
| `ueba` | 14 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
