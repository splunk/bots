# BOTS v6 MCT: Pre-indexed Dataset Recovery

This guide restores the BOTS v6 MCT scenario into a dedicated index on Splunk Enterprise 10.x. See the [BOTS v6 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v6 MCT dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v6 MCT | Pre-indexed buckets, restored as `botsv6-mct-main` | `.tgz` archive | [Download BOTS v6 MCT dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv6/botsv6-mct.tgz) |
| BOTS v6 Q&A | Questions and answers for all BOTS v6 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v6 Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv6/botsv6-qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv6-mct.tgz`. The archive is about 3.1 GB (about 5.0 GB extracted). Its SHA-256 checksum is `343b492a72abf370dbcd969e63cc2531bc7116c4e39dc1a2d77cf2e0127cc1df`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses a new logical index name to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv6-mct-main` | `$SPLUNK_DB/botsv6-mct-main/` | 5.0 GB | 1 |

The archive contains only the `defaultdb` index. There are no notable, risk, or additional indexes for this scenario.

The original archive directory names are unchanged. The new name applies to the index **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory contains acceleration summaries and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 5.0 GB for the extracted staging copy, plus the same again for the restored index), and a backup of the destination.
- The recovery archive, named `botsv6-mct.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv6-mct.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`. The archive should contain 1 `journal.gz` file. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 343b492a72abf370dbcd969e63cc2531bc7116c4e39dc1a2d77cf2e0127cc1df
```

## 2. Configure the dedicated index

Add this stanza to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv6-mct-main]
homePath = $SPLUNK_DB/botsv6-mct-main/db
coldPath = $SPLUNK_DB/botsv6-mct-main/colddb
thawedPath = $SPLUNK_DB/botsv6-mct-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv6-mct-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv6-mct-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv6-mct-stage
sudo find /tmp/botsv6-mct-stage -name journal.gz -type f | wc -l   # expect 1
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new index

These commands copy the original `defaultdb` directory's `db` and `colddb` buckets to the new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv6-mct-main'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv6-mct-stage/$srcname/$tier"
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
sudo chown -R splunk:splunk "$SPLUNK_DB/botsv6-mct-main"
```

The archive's `datamodel_summary/` directory is intentionally not copied. Do not import it as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv6-mct-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv6-mct-main
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
for idx in botsv6-mct-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following **57 sourcetypes** (21,377,568 events) were reported by the source Splunk instance for `index=main` (restored as `index=botsv6-mct-main`). Counts reflect the supplied inventory and may differ on another restoration.

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
