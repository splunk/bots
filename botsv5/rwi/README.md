# BOTS v5 RWI: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 RWI scenario into a dedicated index on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 RWI dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 RWI | Pre-indexed buckets, restored as `botsv5-rwi-main` | `.tgz` archive | [Download BOTS v5 RWI dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-rwi.tgz) |
| BOTS v5 RWI Q&A | Questions and answers for the BOTS v5 RWI exercise | PDF | [Download BOTS v5 RWI Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-rwi-qa.pdf) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-rwi.tgz`. The archive is about 1.7 GB (about 2.7 GB extracted). Its SHA-256 checksum is `bc27c1a3093994757e34a982be686e2c4cd73a1266e3a842c09d24311d97cd11`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `defaultdb/` | `botsv5-rwi-main` | `$SPLUNK_DB/botsv5-rwi-main/` | 10 |

The archive contains only the `defaultdb` index. There are no notable, risk, or additional indexes for this scenario.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory contains acceleration summaries and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv5-rwi.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-rwi.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`; the archive should contain 10 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: bc27c1a3093994757e34a982be686e2c4cd73a1266e3a842c09d24311d97cd11
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-rwi-main]
homePath = $SPLUNK_DB/botsv5-rwi-main/db
coldPath = $SPLUNK_DB/botsv5-rwi-main/colddb
thawedPath = $SPLUNK_DB/botsv5-rwi-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-rwi-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-rwi-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-rwi-stage
sudo find /tmp/botsv5-rwi-stage -name journal.gz -type f | wc -l   # expect 10
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-rwi-main'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-rwi-stage/$srcname/$tier"
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
for idx in botsv5-rwi-main; do
  sudo chown -R splunk:splunk "$SPLUNK_DB/$idx"
done
```

The archive's `datamodel_summary/` directory is intentionally not copied. Do not import it as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv5-rwi-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-rwi-main
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
for idx in botsv5-rwi-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following **70 sourcetypes** (9,947,304 events) were reported by the source Splunk instance for `index=main` (restored as `index=botsv5-rwi-main`). Counts reflect the supplied inventory and may differ on another restoration.

| Sourcetype | Events |
| --- | ---: |
| `stream:ip` | 3,327,217 |
| `stream:dns` | 2,484,363 |
| `stream:tcp` | 1,986,984 |
| `juniper:junos:firewall` | 535,439 |
| `stream:http` | 421,879 |
| `XmlWinEventLog` | 398,434 |
| `stream:sip` | 165,580 |
| `linux_audit` | 131,779 |
| `stream:udp` | 118,253 |
| `stream:smb` | 82,119 |
| `access-too_small` | 48,856 |
| `syslog` | 47,104 |
| `stream:ldap` | 46,774 |
| `freepbx-2` | 33,245 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 21,584 |
| `juniper:ive` | 21,224 |
| `stream:icmp` | 16,719 |
| `osquery:info` | 11,151 |
| `Script:ListeningPorts` | 10,386 |
| `XmlWinEventLog:Microsoft-Windows-Powershell/Operational` | 7,877 |
| `gapps:admin:directory:users` | 7,614 |
| `Unix:Service` | 6,011 |
| `osquery:results` | 4,062 |
| `gapps:chrome:api` | 1,874 |
| `Script:InstalledApps` | 1,850 |
| `Powershell:ScriptExecutionSummary` | 1,571 |
| `MSAD:NT6:Health` | 1,315 |
| `Unix:ListeningPorts` | 1,166 |
| `redis` | 1,152 |
| `postfix_syslog` | 486 |
| `MSAD:NT6:DNS-Zone-Information` | 481 |
| `apache_error` | 386 |
| `Script:TimesyncConfiguration` | 264 |
| `Script:TimesyncStatus` | 264 |
| `MSAD:NT6:SiteInfo` | 212 |
| `Powershell:ScriptExecutionErrorRecord` | 211 |
| `stream:ftp` | 147 |
| `zoom:webhook` | 147 |
| `ActiveDirectory` | 139 |
| `access_log-too_small` | 129 |
| `gmail:enforce:audit:ta:output` | 98 |
| `dpkg-too_small` | 88 |
| `ucp_out` | 77 |
| `osquery:warning` | 74 |
| `Script:NetworkConfiguration` | 70 |
| `MSAD:NT6:DNS-Health` | 62 |
| `osquery:snapshots` | 62 |
| `gmail:audit:headers` | 54 |
| `XmlWinEventLog:Microsoft-Windows-PowerShell/Operational` | 52 |
| `GSuiteForSplunk:error` | 36 |
| `gapps:report:drive` | 35 |
| `gsuite:directory:user` | 24 |
| `error-too_small` | 17 |
| `ucp_err` | 15 |
| `Script:GetEndpointInfo` | 11 |
| `gapps:report:admin` | 11 |
| `term-too_small` | 11 |
| `gapps:report:token` | 10 |
| `ssl_access_log-too_small` | 10 |
| `ssl_request_log-too_small` | 10 |
| `gapps:report:mobile` | 7 |
| `history-too_small` | 6 |
| `gapps:report:user_accounts` | 3 |
| `pulse:connectsecure` | 3 |
| `queue_log-too_small` | 3 |
| `fail2ban-2` | 2 |
| `freepbx_security-too_small` | 2 |
| `gapps:usage:customer` | 1 |
| `kern-too_small` | 1 |
| `kern.log-too_small` | 1 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
