# BOTS v5 ESS: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 ESS scenario into 4 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 ESS dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 ESS | Pre-indexed buckets, restored as `botsv5-ess-main`, `botsv5-ess-ar`, `botsv5-ess-notable`, `botsv5-ess-risk` | `.tgz` archive | [Download BOTS v5 ESS dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv5/botsv5-ess.tgz) |
| BOTS v5 Q&A | Questions and answers for all BOTS v5 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v5 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv5/BOTSv5qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-ess.tgz`. The archive is about 3.4 GB (about 5.8 GB extracted). Its SHA-256 checksum is `051569b944564b78bed3500fe82d65c77ce33fcabfb96b18399ebfb9cbcda224`.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `defaultdb/` | `botsv5-ess-main` | `$SPLUNK_DB/botsv5-ess-main/` | 34 |
| `botsv_ar/` | `botsv5-ess-ar` | `$SPLUNK_DB/botsv5-ess-ar/` | 5 |
| `notabledb/` | `botsv5-ess-notable` | `$SPLUNK_DB/botsv5-ess-notable/` | 4 |
| `riskdb/` | `botsv5-ess-risk` | `$SPLUNK_DB/botsv5-ess-risk/` | 3 |

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `defaultdb/` and `botsv_ar/`) contain acceleration summaries and are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 5.8 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv5-ess.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-ess.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep -E '^[^/]+/db/db_[^/]+/$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `botsv_ar/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`. The archive should contain 46 bucket directories. Of these, 45 hold a `rawdata/journal.gz` file. The other one (in `defaultdb`) is a small bucket whose raw data is stored as `rawdata/0` instead. It is a valid bucket and is restored with the rest. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 051569b944564b78bed3500fe82d65c77ce33fcabfb96b18399ebfb9cbcda224
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-ess-main]
homePath = $SPLUNK_DB/botsv5-ess-main/db
coldPath = $SPLUNK_DB/botsv5-ess-main/colddb
thawedPath = $SPLUNK_DB/botsv5-ess-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-ess-ar]
homePath = $SPLUNK_DB/botsv5-ess-ar/db
coldPath = $SPLUNK_DB/botsv5-ess-ar/colddb
thawedPath = $SPLUNK_DB/botsv5-ess-ar/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-ess-notable]
homePath = $SPLUNK_DB/botsv5-ess-notable/db
coldPath = $SPLUNK_DB/botsv5-ess-notable/colddb
thawedPath = $SPLUNK_DB/botsv5-ess-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-ess-risk]
homePath = $SPLUNK_DB/botsv5-ess-risk/db
coldPath = $SPLUNK_DB/botsv5-ess-risk/colddb
thawedPath = $SPLUNK_DB/botsv5-ess-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-ess-main botsv5-ess-ar botsv5-ess-notable botsv5-ess-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-ess-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-ess-stage
sudo find /tmp/botsv5-ess-stage -type d -name 'db_*' | wc -l   # expect 46
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-ess-main' 'botsv_ar:botsv5-ess-ar' 'notabledb:botsv5-ess-notable' 'riskdb:botsv5-ess-risk'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-ess-stage/$srcname/$tier"
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
for idx in botsv5-ess-main botsv5-ess-ar botsv5-ess-notable botsv5-ess-risk; do
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
| tstats count WHERE index=botsv5-ess-main earliest=0 BY sourcetype
| sort -count
```

```spl
| tstats count WHERE index=botsv5-ess-* earliest=0 BY index
| sort index
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-ess-main
| table bucketId state eventCount path
```

If data does not appear, check logs before attempting another import:

```bash
grep -Ei 'bucket|tsidx|journal|corrupt|incompatible|freeze|frozen' \
  "$SPLUNK_HOME/var/log/splunk/splunkd.log" | tail -100
```

Do not delete the original archive or repeatedly restart against the only copy of the buckets. If Splunk 10.x rejects the files, an intermediate compatible Splunk version or another supported migration method may be necessary.

## 6. Verify retention

```bash
for idx in botsv5-ess-main botsv5-ess-ar botsv5-ess-notable botsv5-ess-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes included in the archive. Each is restored into the matching `botsv5-ess-*` index. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv5-ess-main`)

**121 sourcetypes**, 11,693,645 events.

| Sourcetype | Events |
| --- | ---: |
| `stream:ip` | 3,620,360 |
| `stream:dns` | 2,768,923 |
| `stream:tcp` | 2,202,686 |
| `stream:http` | 552,503 |
| `juniper:junos:firewall` | 535,439 |
| `XmlWinEventLog` | 513,211 |
| `stream:smb` | 429,174 |
| `stream:udp` | 270,839 |
| `stream:sip` | 165,580 |
| `linux_audit` | 131,779 |
| `aws:cloudwatchlogs:eks` | 94,470 |
| `access-too_small` | 48,856 |
| `stream:ldap` | 48,202 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 47,887 |
| `syslog` | 47,104 |
| `freepbx-2` | 33,245 |
| `juniper:ive` | 21,224 |
| `stream:icmp` | 16,719 |
| `Script:ListeningPorts` | 11,214 |
| `osquery:info` | 11,151 |
| `fluentd:monitor-agent` | 11,086 |
| `aws:elb:accesslogs` | 9,981 |
| `kube:container:webfrontend` | 9,022 |
| `kube:container:adminfrontend` | 8,757 |
| `XmlWinEventLog:Microsoft-Windows-Powershell/Operational` | 7,877 |
| `gapps:admin:directory:users` | 7,614 |
| `Unix:Service` | 6,011 |
| `kube:objects:service_accounts` | 5,632 |
| `kube:objects:pods` | 5,006 |
| `kube:container:splunk-fluentd-k8s-logs` | 4,924 |
| `osquery:results` | 4,062 |
| `stream:mysql` | 2,972 |
| `kube:objects:replica_sets` | 2,817 |
| `Perfmon:Network Interface` | 2,508 |
| `Perfmon:CPU Load` | 2,435 |
| `kube:container:kubernetes-dashboard` | 2,341 |
| `Script:InstalledApps` | 2,084 |
| `kube:objects:deployments` | 2,051 |
| `kube:objects:config_maps` | 1,905 |
| `gapps:chrome:api` | 1,874 |
| `kube:objects:services` | 1,778 |
| `Powershell:ScriptExecutionSummary` | 1,571 |
| `MSAD:NT6:Health` | 1,315 |
| `Perfmon:Available Memory` | 1,254 |
| `Unix:ListeningPorts` | 1,166 |
| `kube:container:dashboard-metrics-scraper` | 1,157 |
| `redis` | 1,152 |
| `o365:management:activity` | 968 |
| `o365:service:status` | 924 |
| `kube:objects:namespaces` | 889 |
| `kube:objects:daemon_sets` | 663 |
| `kube:objects:events` | 634 |
| `stream:dhcp` | 632 |
| `kube:objects:nodes` | 508 |
| `postfix_syslog` | 486 |
| `MSAD:NT6:DNS-Zone-Information` | 481 |
| `kube:container:cartservice` | 426 |
| `apache_error` | 386 |
| `azure:eventhub` | 382 |
| `kube:objects:component_statuses` | 381 |
| `kube` | 344 |
| `kube:container:productservice` | 308 |
| `Script:TimesyncConfiguration` | 292 |
| `Script:TimesyncStatus` | 292 |
| `kube:container:nginx` | 282 |
| `kube:container:metrics-server` | 232 |
| `MSAD:NT6:SiteInfo` | 212 |
| `Powershell:ScriptExecutionErrorRecord` | 211 |
| `XmlWinEventLog:Microsoft-Windows-PowerShell/Operational` | 165 |
| `azure:aad:signin` | 159 |
| `kube:container:splunk-fluentd-k8s-metrics-agg` | 149 |
| `stream:ftp` | 147 |
| `zoom:webhook` | 147 |
| `kube:container:alb-ingress-controller` | 146 |
| `kube:kubelet` | 142 |
| `ActiveDirectory` | 139 |
| `access_log-too_small` | 129 |
| `kube:container:ordersservice` | 128 |
| `kube:container:backend-tools` | 104 |
| `aws:cloudtrail` | 101 |
| `gmail:enforce:audit:ta:output` | 98 |
| `kube:docker` | 92 |
| `dpkg-too_small` | 88 |
| `Script:NetworkConfiguration` | 78 |
| `ucp_out` | 77 |
| `osquery:warning` | 74 |
| `MSAD:NT6:DNS-Health` | 62 |
| `osquery:snapshots` | 62 |
| `kube:container:splunk-operator` | 55 |
| `gmail:audit:headers` | 54 |
| `kube:container:splunk-fluentd-k8s-metrics` | 43 |
| `kube:container:kube-hunter` | 41 |
| `kube:container:splunk-fluentd-k8s-objects` | 41 |
| `GSuiteForSplunk:error` | 36 |
| `gapps:report:drive` | 35 |
| `kube:container:adminfrontend2` | 27 |
| `gsuite:directory:user` | 24 |
| `error-too_small` | 17 |
| `ucp_err` | 15 |
| `Script:GetEndpointInfo` | 11 |
| `gapps:report:admin` | 11 |
| `term-too_small` | 11 |
| `gapps:report:token` | 10 |
| `ssl_access_log-too_small` | 10 |
| `ssl_request_log-too_small` | 10 |
| `XmlWinEventLog:Microsoft-Windows-Windows Defender/Operational` | 7 |
| `gapps:report:mobile` | 7 |
| `Perfmon:Free Disk Space` | 6 |
| `history-too_small` | 6 |
| `pod-file-changes` | 6 |
| `kube:container:coredns` | 4 |
| `gapps:report:user_accounts` | 3 |
| `o365:service:message` | 3 |
| `pulse:connectsecure` | 3 |
| `queue_log-too_small` | 3 |
| `fail2ban-2` | 2 |
| `freepbx_security-too_small` | 2 |
| `gapps:usage:customer` | 1 |
| `kern-too_small` | 1 |
| `kern.log-too_small` | 1 |
| `pod-bash-history` | 1 |

### `index=notable` (restored as `index=botsv5-ess-notable`)

**1 sourcetypes**, 14 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 14 |

### `index=risk` (restored as `index=botsv5-ess-risk`)

**1 sourcetypes**, 28 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 28 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
