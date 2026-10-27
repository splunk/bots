# BOTS v5 AWS: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 AWS scenario into 9 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 AWS dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 AWS | Pre-indexed buckets, restored as `botsv5-aws-main`, `botsv5-aws-notable`, `botsv5-aws-risk` and six additional AWS indexes | `.tgz` archive | [Download BOTS v5 AWS dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-aws.tgz) |
| BOTS v5 AWS Q&A | Questions and answers for the BOTS v5 AWS exercise | PDF | [Download BOTS v5 AWS Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-aws-qa.pdf) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-aws.tgz`. The archive is about 42 MB. Its SHA-256 checksum is `4454f684e67a0e64694847d5d5b584074f5bc4c8e92897839e6189ca41475d78`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv5-aws-main` | `$SPLUNK_DB/botsv5-aws-main/` | 41 MB | 2 |
| `notabledb/` | `botsv5-aws-notable` | `$SPLUNK_DB/botsv5-aws-notable/` | 320 KB | 5 |
| `riskdb/` | `botsv5-aws-risk` | `$SPLUNK_DB/botsv5-aws-risk/` | 28 KB | 0 |
| `aws_anomaly_detection/` | `botsv5-aws-anomaly_detection` | `$SPLUNK_DB/botsv5-aws-anomaly_detection/` | 36 KB | 0 |
| `aws_topology_daily_snapshot/` | `botsv5-aws-topology_daily_snapshot` | `$SPLUNK_DB/botsv5-aws-topology_daily_snapshot/` | 55 MB | 209 |
| `aws_topology_history/` | `botsv5-aws-topology_history` | `$SPLUNK_DB/botsv5-aws-topology_history/` | 3.4 MB | 210 |
| `aws_topology_monthly_snapshot/` | `botsv5-aws-topology_monthly_snapshot` | `$SPLUNK_DB/botsv5-aws-topology_monthly_snapshot/` | 372 KB | 1 |
| `aws_topology_playback/` | `botsv5-aws-topology_playback` | `$SPLUNK_DB/botsv5-aws-topology_playback/` | 1.7 MB | 9 |
| `aws_vpc_flow_logs/` | `botsv5-aws-vpc_flow_logs` | `$SPLUNK_DB/botsv5-aws-vpc_flow_logs/` | 11 MB | 9 |

The `aws_` prefix is dropped from the six additional index names because `botsv5-aws-` already identifies the scenario.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive also contains `summary/` and `datamodel_summary/` directories, which are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv5-aws.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-aws.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`; the archive should contain 445 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 4454f684e67a0e64694847d5d5b584074f5bc4c8e92897839e6189ca41475d78
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-aws-main]
homePath = $SPLUNK_DB/botsv5-aws-main/db
coldPath = $SPLUNK_DB/botsv5-aws-main/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-notable]
homePath = $SPLUNK_DB/botsv5-aws-notable/db
coldPath = $SPLUNK_DB/botsv5-aws-notable/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-risk]
homePath = $SPLUNK_DB/botsv5-aws-risk/db
coldPath = $SPLUNK_DB/botsv5-aws-risk/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-anomaly_detection]
homePath = $SPLUNK_DB/botsv5-aws-anomaly_detection/db
coldPath = $SPLUNK_DB/botsv5-aws-anomaly_detection/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-anomaly_detection/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-topology_daily_snapshot]
homePath = $SPLUNK_DB/botsv5-aws-topology_daily_snapshot/db
coldPath = $SPLUNK_DB/botsv5-aws-topology_daily_snapshot/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-topology_daily_snapshot/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-topology_history]
homePath = $SPLUNK_DB/botsv5-aws-topology_history/db
coldPath = $SPLUNK_DB/botsv5-aws-topology_history/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-topology_history/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-topology_monthly_snapshot]
homePath = $SPLUNK_DB/botsv5-aws-topology_monthly_snapshot/db
coldPath = $SPLUNK_DB/botsv5-aws-topology_monthly_snapshot/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-topology_monthly_snapshot/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-topology_playback]
homePath = $SPLUNK_DB/botsv5-aws-topology_playback/db
coldPath = $SPLUNK_DB/botsv5-aws-topology_playback/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-topology_playback/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv5-aws-vpc_flow_logs]
homePath = $SPLUNK_DB/botsv5-aws-vpc_flow_logs/db
coldPath = $SPLUNK_DB/botsv5-aws-vpc_flow_logs/colddb
thawedPath = $SPLUNK_DB/botsv5-aws-vpc_flow_logs/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-aws-main botsv5-aws-notable botsv5-aws-risk botsv5-aws-anomaly_detection botsv5-aws-topology_daily_snapshot botsv5-aws-topology_history botsv5-aws-topology_monthly_snapshot botsv5-aws-topology_playback botsv5-aws-vpc_flow_logs; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-aws-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-aws-stage
sudo find /tmp/botsv5-aws-stage -name journal.gz -type f | wc -l   # expect 445
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-aws-main' 'notabledb:botsv5-aws-notable' 'riskdb:botsv5-aws-risk' 'aws_anomaly_detection:botsv5-aws-anomaly_detection' 'aws_topology_daily_snapshot:botsv5-aws-topology_daily_snapshot' 'aws_topology_history:botsv5-aws-topology_history' 'aws_topology_monthly_snapshot:botsv5-aws-topology_monthly_snapshot' 'aws_topology_playback:botsv5-aws-topology_playback' 'aws_vpc_flow_logs:botsv5-aws-vpc_flow_logs'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-aws-stage/$srcname/$tier"
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
for idx in botsv5-aws-main botsv5-aws-notable botsv5-aws-risk botsv5-aws-anomaly_detection botsv5-aws-topology_daily_snapshot botsv5-aws-topology_history botsv5-aws-topology_monthly_snapshot botsv5-aws-topology_playback botsv5-aws-vpc_flow_logs; do
  sudo chown -R splunk:splunk "$SPLUNK_DB/$idx"
done
```

The archive's `summary/` and `datamodel_summary/` directories are intentionally not copied. Do not import them as event buckets.

### Thaw the frozen buckets

The `aws_topology_history` (210 buckets) and `aws_vpc_flow_logs` (9 buckets) directories keep their buckets in `frozendb/`, not `db/`. The copy loop above does not touch `frozendb/`, so without this step those two indexes restore empty. Frozen buckets contain only raw data, so they are copied into each index's `thaweddb/` and rebuilt. Splunk must still be stopped.

```bash
set -e

for mapping in 'aws_topology_history:botsv5-aws-topology_history' 'aws_vpc_flow_logs:botsv5-aws-vpc_flow_logs'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}
  src="/tmp/botsv5-aws-stage/$srcname/frozendb"
  dst="$SPLUNK_DB/$dstname/thaweddb"

  sudo mkdir -p "$dst"
  if [ -n "$(sudo find "$dst" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
    echo "ERROR: destination is not empty: $dst" >&2
    exit 1
  fi
  sudo cp -a "$src/." "$dst/"
  sudo chown -R splunk:splunk "$dst"

  for bucket in "$dst"/db_*; do
    sudo -u splunk "$SPLUNK_HOME/bin/splunk" rebuild "$bucket" "$dstname"
  done
done
```

Run the rebuild as the Splunk service account (`splunk` here) so the generated index files have the right ownership. Rebuilding 219 buckets can take a while. Thawed buckets are not subject to `frozenTimePeriodInSecs` or `maxTotalDataSizeMB`, so they are not frozen again.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv5-aws-main earliest=0 BY sourcetype
| sort -count
```

```spl
| tstats count WHERE index=botsv5-aws-* earliest=0 BY index
| sort index
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-aws-main
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
for idx in botsv5-aws-main botsv5-aws-notable botsv5-aws-risk botsv5-aws-anomaly_detection botsv5-aws-topology_daily_snapshot botsv5-aws-topology_history botsv5-aws-topology_monthly_snapshot botsv5-aws-topology_playback botsv5-aws-vpc_flow_logs; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
