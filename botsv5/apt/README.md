# BOTS v5 APT: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 APT scenario into 3 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 APT dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 APT | Pre-indexed buckets, restored as `botsv5-apt-main`, `botsv5-apt-notable`, `botsv5-apt-risk` | `.tgz` archive | [Download BOTS v5 APT dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-apt.tgz) |
| BOTS v5 Q&A | Questions and answers for all BOTS v5 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v5 Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-apt.tgz`. The archive is about 974 MB. Its SHA-256 checksum is `2ef05ceea74edc4de21a7ead0ad816187d541629474306186f55c603dec19b12`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `defaultdb/` | `botsv5-apt-main` | `$SPLUNK_DB/botsv5-apt-main/` | 13 |
| `notabledb/` | `botsv5-apt-notable` | `$SPLUNK_DB/botsv5-apt-notable/` | 3 |
| `riskdb/` | `botsv5-apt-risk` | `$SPLUNK_DB/botsv5-apt-risk/` | 2 |

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv5-apt.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-apt.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`; the archive should contain 18 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 2ef05ceea74edc4de21a7ead0ad816187d541629474306186f55c603dec19b12
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-apt-main]
homePath = $SPLUNK_DB/botsv5-apt-main/db
coldPath = $SPLUNK_DB/botsv5-apt-main/colddb
thawedPath = $SPLUNK_DB/botsv5-apt-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-apt-notable]
homePath = $SPLUNK_DB/botsv5-apt-notable/db
coldPath = $SPLUNK_DB/botsv5-apt-notable/colddb
thawedPath = $SPLUNK_DB/botsv5-apt-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-apt-risk]
homePath = $SPLUNK_DB/botsv5-apt-risk/db
coldPath = $SPLUNK_DB/botsv5-apt-risk/colddb
thawedPath = $SPLUNK_DB/botsv5-apt-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-apt-main botsv5-apt-notable botsv5-apt-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-apt-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-apt-stage
sudo find /tmp/botsv5-apt-stage -name journal.gz -type f | wc -l   # expect 18
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-apt-main' 'notabledb:botsv5-apt-notable' 'riskdb:botsv5-apt-risk'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-apt-stage/$srcname/$tier"
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
for idx in botsv5-apt-main botsv5-apt-notable botsv5-apt-risk; do
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
| tstats count WHERE index=botsv5-apt-main earliest=0 BY sourcetype
| sort -count
```

```spl
| tstats count WHERE index=botsv5-apt-* earliest=0 BY index
| sort index
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-apt-main
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
for idx in botsv5-apt-main botsv5-apt-notable botsv5-apt-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv5-apt-main`)

**26 sourcetypes**, 704,492 events.

| Sourcetype | Events |
| --- | ---: |
| `stream:smb` | 347,055 |
| `XmlWinEventLog` | 114,788 |
| `stream:ip` | 66,716 |
| `stream:dns` | 60,564 |
| `stream:udp` | 42,713 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 26,303 |
| `stream:tcp` | 23,903 |
| `stream:http` | 10,504 |
| `Perfmon:Network Interface` | 2,508 |
| `Perfmon:CPU Load` | 2,435 |
| `stream:ldap` | 1,428 |
| `Perfmon:Available Memory` | 1,254 |
| `o365:management:activity` | 968 |
| `o365:service:status` | 924 |
| `Script:ListeningPorts` | 828 |
| `stream:dhcp` | 633 |
| `azure:eventhub` | 382 |
| `Script:InstalledApps` | 234 |
| `azure:aad:signin` | 159 |
| `XmlWinEventLog:Microsoft-Windows-PowerShell/Operational` | 113 |
| `Script:TimesyncConfiguration` | 28 |
| `Script:TimesyncStatus` | 28 |
| `Script:NetworkConfiguration` | 8 |
| `XmlWinEventLog:Microsoft-Windows-Windows Defender/Operational` | 7 |
| `Perfmon:Free Disk Space` | 6 |
| `o365:service:message` | 3 |

### `index=notable` (restored as `index=botsv5-apt-notable`)

**1 sourcetype**, 27 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 27 |

### `index=risk` (restored as `index=botsv5-apt-risk`)

**1 sourcetype**, 26 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 26 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
