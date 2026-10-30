# BOTS v6 ESS: Pre-indexed Dataset Recovery

This guide restores the BOTS v6 ESS scenario into 7 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v6 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v6 ESS dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v6 ESS | Pre-indexed buckets, restored as `botsv6-ess-apt`, `botsv6-ess-apt_dtex`, `botsv6-ess-frothly_events`, `botsv6-ess-insider`, `botsv6-ess-notable`, `botsv6-ess-risk`, `botsv6-ess-toads` | `.tgz` archive | [Download BOTS v6 ESS dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv6/botsv6-ess.tgz) |
| BOTS v6 ESS Q&A | Questions and answers for the BOTS v6 ESS exercise | `.zip` archive | [Download BOTS v6 ESS Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv6/botsv6-ess-qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv6-ess.tgz`. The archive is about 6.1 GB (about 10.6 GB extracted). Its SHA-256 checksum is `9d70218d2681e8a3fb4b4949e4ff707fe9baa92225e4f82584b94bc82e85c742`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `apt/` | `botsv6-ess-apt` | `$SPLUNK_DB/botsv6-ess-apt/` | 2.4 GB | 5 |
| `apt_dtex/` | `botsv6-ess-apt_dtex` | `$SPLUNK_DB/botsv6-ess-apt_dtex/` | 41 MB | 4 |
| `frothly_events/` | `botsv6-ess-frothly_events` | `$SPLUNK_DB/botsv6-ess-frothly_events/` | 0.2 MB | 1 |
| `insider/` | `botsv6-ess-insider` | `$SPLUNK_DB/botsv6-ess-insider/` | 2.3 GB | 1 |
| `notabledb/` | `botsv6-ess-notable` | `$SPLUNK_DB/botsv6-ess-notable/` | 0.3 MB | 13 |
| `riskdb/` | `botsv6-ess-risk` | `$SPLUNK_DB/botsv6-ess-risk/` | 0.4 MB | 10 |
| `toads/` | `botsv6-ess-toads` | `$SPLUNK_DB/botsv6-ess-toads/` | 5.9 GB | 1 |

The `notabledb/` and `riskdb/` archive directories are restored as `botsv6-ess-notable` and `botsv6-ess-risk`. The archive does not contain a `defaultdb/` (`main`) index.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `apt/`, `apt_dtex/`, `frothly_events/`, `insider/`, and `toads/`) contain acceleration summaries and are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 10.6 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv6-ess.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv6-ess.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `apt/db/db_...`, `apt_dtex/db/db_...`, `frothly_events/db/db_...`, `insider/db/db_...`, `notabledb/db/db_...`, `riskdb/db/db_...`, and `toads/db/db_...`. The archive should contain 35 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 9d70218d2681e8a3fb4b4949e4ff707fe9baa92225e4f82584b94bc82e85c742
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv6-ess-apt]
homePath = $SPLUNK_DB/botsv6-ess-apt/db
coldPath = $SPLUNK_DB/botsv6-ess-apt/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-apt/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-apt_dtex]
homePath = $SPLUNK_DB/botsv6-ess-apt_dtex/db
coldPath = $SPLUNK_DB/botsv6-ess-apt_dtex/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-apt_dtex/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-frothly_events]
homePath = $SPLUNK_DB/botsv6-ess-frothly_events/db
coldPath = $SPLUNK_DB/botsv6-ess-frothly_events/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-frothly_events/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-insider]
homePath = $SPLUNK_DB/botsv6-ess-insider/db
coldPath = $SPLUNK_DB/botsv6-ess-insider/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-insider/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-notable]
homePath = $SPLUNK_DB/botsv6-ess-notable/db
coldPath = $SPLUNK_DB/botsv6-ess-notable/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-risk]
homePath = $SPLUNK_DB/botsv6-ess-risk/db
coldPath = $SPLUNK_DB/botsv6-ess-risk/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-ess-toads]
homePath = $SPLUNK_DB/botsv6-ess-toads/db
coldPath = $SPLUNK_DB/botsv6-ess-toads/colddb
thawedPath = $SPLUNK_DB/botsv6-ess-toads/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv6-ess-apt botsv6-ess-apt_dtex botsv6-ess-frothly_events botsv6-ess-insider botsv6-ess-notable botsv6-ess-risk botsv6-ess-toads; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv6-ess-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv6-ess-stage
sudo find /tmp/botsv6-ess-stage -name journal.gz -type f | wc -l   # expect 35
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'apt:botsv6-ess-apt' 'apt_dtex:botsv6-ess-apt_dtex' 'frothly_events:botsv6-ess-frothly_events' 'insider:botsv6-ess-insider' 'notabledb:botsv6-ess-notable' 'riskdb:botsv6-ess-risk' 'toads:botsv6-ess-toads'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv6-ess-stage/$srcname/$tier"
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
for idx in botsv6-ess-apt botsv6-ess-apt_dtex botsv6-ess-frothly_events botsv6-ess-insider botsv6-ess-notable botsv6-ess-risk botsv6-ess-toads; do
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
| tstats count WHERE index=botsv6-ess-* earliest=0 BY index
| sort index
```

```spl
| tstats count WHERE index=botsv6-ess-* earliest=0 BY index, sourcetype
| sort index, -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv6-ess-*
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
for idx in botsv6-ess-apt botsv6-ess-apt_dtex botsv6-ess-frothly_events botsv6-ess-insider botsv6-ess-notable botsv6-ess-risk botsv6-ess-toads; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=apt` (restored as `index=botsv6-ess-apt`)

**33 sourcetypes**, 1,017,664 events.

| Sourcetype | Events |
| --- | ---: |
| `stream:smb` | 460,582 |
| `XmlWinEventLog` | 256,807 |
| `bro:dns:json` | 116,594 |
| `stream:dns` | 75,586 |
| `stream:ip` | 31,179 |
| `bro:conn:json` | 22,926 |
| `stream:udp` | 18,093 |
| `stream:tcp` | 10,657 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 8,920 |
| `bro:ssl:json` | 7,857 |
| `bro:rdp:json` | 3,156 |
| `o365:graph:api` | 1,770 |
| `stream:http` | 1,478 |
| `Script:ListeningPorts` | 909 |
| `o365:management:activity` | 311 |
| `bro:http:json` | 174 |
| `bro:files:json` | 171 |
| `WindowsUpdateLog` | 147 |
| `azure:aad:signin` | 88 |
| `Script:InstalledApps` | 48 |
| `ms:iis:splunk` | 33 |
| `Script:TimesyncConfiguration` | 29 |
| `Script:TimesyncStatus` | 29 |
| `azure:aad:audit` | 26 |
| `o365:service:status` | 25 |
| `bro:smb_mapping:json` | 24 |
| `ActiveDirectory` | 18 |
| `bro:smb_files:json` | 9 |
| `o365:service:message` | 6 |
| `Script:GetEndpointInfo` | 5 |
| `Script:NetworkConfiguration` | 3 |
| `bro:x509:json` | 3 |
| `bro:software:json` | 1 |

### `index=apt_dtex` (restored as `index=botsv6-ess-apt_dtex`)

**1 sourcetype**, 8,061 events.

| Sourcetype | Events |
| --- | ---: |
| `dtex_st_activities` | 8,061 |

### `index=frothly_events` (restored as `index=botsv6-ess-frothly_events`)

**1 sourcetype**, 1,586 events.

| Sourcetype | Events |
| --- | ---: |
| `st_frothly_events` | 1,586 |

### `index=insider` (restored as `index=botsv6-ess-insider`)

**63 sourcetypes**, 5,122,137 events.

| Sourcetype | Events |
| --- | ---: |
| `cp_log` | 2,942,654 |
| `XmlWinEventLog` | 1,193,859 |
| `osquery:info` | 178,054 |
| `sfdc:logfile` | 103,728 |
| `osquery:results` | 74,163 |
| `sfdc-streaming-api-events` | 69,289 |
| `gapps:admin:directory:users` | 49,560 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 34,651 |
| `XmlWinEventLog:Microsoft-Windows-Powershell/Operational` | 30,710 |
| `Script:ListeningPorts` | 28,472 |
| `osquery:warning` | 27,560 |
| `osquery:error` | 26,497 |
| `gmail:audit:ta:output` | 24,032 |
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
| `gmail:enforce:audit:ta:output` | 8,499 |
| `gapps:chrome:api` | 7,081 |
| `Powershell:ScriptExecutionSummary` | 5,976 |
| `MSAD:NT6:Health` | 5,035 |
| `MSAD:NT6:DNS-Zone-Information` | 1,912 |
| `Script:InstalledApps` | 1,885 |
| `ActiveDirectory` | 1,298 |
| `Powershell:ScriptExecutionErrorRecord` | 779 |
| `osquery:snapshots` | 747 |
| `MSAD:NT6:SiteInfo` | 742 |
| `Script:TimesyncConfiguration` | 658 |
| `Script:TimesyncStatus` | 658 |
| `MSAD:NT6:DNS-Health` | 239 |
| `XmlWinEventLog:Microsoft-Windows-PrintService/Operational` | 179 |
| `GSuiteForSplunk:error` | 110 |
| `gmail:audit:headers` | 87 |
| `WinPrintMon` | 82 |
| `Script:NetworkConfiguration` | 74 |
| `gapps:usage:user` | 55 |
| `gapps:report:drive` | 36 |
| `Script:GetEndpointInfo` | 21 |
| `gapps:report:mobile` | 13 |
| `gapps:report:token` | 8 |
| `gapps:usage:customer` | 8 |
| `gapps:report:calendar` | 3 |
| `sfdc:account` | 3 |
| `gapps:report:login` | 2 |
| `sfdc:user` | 2 |
| `XmlWinEventLog:Microsoft-Windows-PrintService/Admin` | 1 |
| `gapps:report:user_accounts` | 1 |

### `index=notable` (restored as `index=botsv6-ess-notable`)

**1 sourcetype**, 85 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 85 |

### `index=risk` (restored as `index=botsv6-ess-risk`)

**1 sourcetype**, 118 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 118 |

### `index=toads` (restored as `index=botsv6-ess-toads`)

**57 sourcetypes**, 21,377,568 events.

| Sourcetype | Events |
| --- | ---: |
| `XmlWinEventLog` | 10,288,688 |
| `amdl:nsg:flowlogs` | 4,248,658 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 3,206,665 |
| `google:gcp:pubsub:message` | 1,287,512 |
| `WindowsUpdateLog` | 568,303 |
| `aws:cloudwatchlogs:vpcflow` | 479,681 |
| `aws:cloudwatchlogs` | 447,967 |
| `aws:cloudtrail` | 96,135 |
| `azure:vnet:ip:public` | 95,221 |
| `azure:compute:vm` | 81,666 |
| `syslog` | 73,633 |
| `azure:vnet:nsg` | 68,020 |
| `azure:compute:snapshot` | 54,448 |
| `WinEventLog:Microsoft-Windows-PowerShell/Operational` | 49,344 |
| `linux_audit` | 45,476 |
| `auth-too_small` | 42,370 |
| `azure:vnet:nic` | 40,815 |
| `azure:compute:disk` | 27,230 |
| `azure:compute:image` | 27,228 |
| `azure:compute:vm:instanceView` | 27,220 |
| `Script:ListeningPorts` | 20,923 |
| `daemon-too_small` | 19,191 |
| `azure:aad:device` | 16,350 |
| `azure:aad:user` | 14,781 |
| `azure:aad:signin` | 14,640 |
| `azure:vnet` | 13,606 |
| `access-too_small` | 6,850 |
| `google:gcp:buckets:jsondata` | 4,482 |
| `o365:management:activity` | 2,778 |
| `access_combined` | 2,206 |
| `ActiveDirectory` | 1,954 |
| `Script:InstalledApps` | 1,019 |
| `XmlWinEventLog:Microsoft-Windows-Windows Defender/Operational` | 588 |
| `Script:TimesyncConfiguration` | 457 |
| `Script:TimesyncStatus` | 457 |
| `ms:o365:email` | 305 |
| `user-too_small` | 174 |
| `apache_error` | 95 |
| `error` | 90 |
| `debug-too_small` | 77 |
| `unattended-upgrades-too_small` | 47 |
| `Script:NetworkConfiguration` | 46 |
| `dpkg-too_small` | 43 |
| `azure:aad:audit` | 40 |
| `azure:resource:group` | 36 |
| `Script:GetEndpointInfo` | 19 |
| `azure:subscriptions` | 9 |
| `php7.4-fpm-too_small` | 6 |
| `google:gcp:pubsub:audit:auth` | 3 |
| `term-too_small` | 3 |
| `unattended-upgrades-dpkg-too_small` | 3 |
| `alternatives-too_small` | 2 |
| `google:gsuite:pubsub:audit:auth` | 2 |
| `history-too_small` | 2 |
| `powershell` | 2 |
| `google:gcp:buckets:metadata` | 1 |
| `web` | 1 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
