# BOTS v9 Serverless Surprise: Pre-indexed Dataset Recovery

This guide restores the BOTS v9 Serverless Surprise scenario into a dedicated index on Splunk Enterprise 10.x. See the [BOTS v9 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v9 Serverless Surprise dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v9 Serverless Surprise | Pre-indexed buckets, restored as `botsv9-serverless-main` | `.tgz` archive | [Download BOTS v9 Serverless Surprise dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv9/botsv9-serverless-surprise.tgz) |
| BOTS v9 Q&A | Questions and answers for all BOTS v9 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v9 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv9/BOTSv9qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv9-serverless-surprise.tgz`. The archive is about 30.4 GB (about 42 GB extracted). Its SHA-256 checksum is `e71112d70ddb95f15c48d57bd024bb8d2dd2219958af79301997dc9bf8057f9e`.

## Index naming

This restoration intentionally uses a new logical index name to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv9-serverless-main` | `$SPLUNK_DB/botsv9-serverless-main/` | 42 GB | 4 |

The `defaultdb/` archive directory is restored as `botsv9-serverless-main`.

The original archive directory names are unchanged. The new name applies to the index **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory (under `defaultdb/`) is empty and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 42 GB for the extracted staging copy, plus the same again for the restored index, plus the 30.4 GB archive itself; roughly 115 GB in total), and a backup of the destination.
- The recovery archive, named `botsv9-serverless-surprise.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv9-serverless-surprise.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep -E '^[^/]+/db/db_[^/]+/$'
```

These instructions expect archive paths like `defaultdb/db/db_...`. The archive should contain 4 bucket directories. Its journals are zstd-compressed (`rawdata/journal.zst`) rather than `journal.gz`. If the layout differs, adjust the staging paths accordingly. Listing and hashing a 30 GB archive takes several minutes. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: e71112d70ddb95f15c48d57bd024bb8d2dd2219958af79301997dc9bf8057f9e
```

## 2. Configure the dedicated index

Add this stanza to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv9-serverless-main]
homePath = $SPLUNK_DB/botsv9-serverless-main/db
coldPath = $SPLUNK_DB/botsv9-serverless-main/colddb
thawedPath = $SPLUNK_DB/botsv9-serverless-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. The restored data is about 42 GB, so it fits under the limit. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
"$SPLUNK_HOME/bin/splunk" btool indexes list botsv9-serverless-main --debug | \
  grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv9-serverless-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv9-serverless-stage
sudo find /tmp/botsv9-serverless-stage -type d -name 'db_*' | wc -l   # expect 4
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new index

These commands copy the original `defaultdb` directory's `db` and `colddb` buckets to the new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv9-serverless-main'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv9-serverless-stage/$srcname/$tier"
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
sudo chown -R splunk:splunk "$SPLUNK_DB/botsv9-serverless-main"
```

The archive's `datamodel_summary/` directory is intentionally not copied. Do not import it as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv9-serverless-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv9-serverless-main
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
"$SPLUNK_HOME/bin/splunk" btool indexes list botsv9-serverless-main --debug | \
  grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the index whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv9-serverless-main`)

**4 sourcetypes**, 520,458 events.

| Sourcetype | Events |
| --- | ---: |
| `aws:cloudwatchlogs:vpcflow` | 455,091 |
| `aws:cloudtrail` | 58,688 |
| `aws:cloudwatchlogs` | 4,995 |
| `aws:cloudwatch` | 1,684 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
