# BOTS v5 GCP: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 GCP scenario into a dedicated index on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 GCP dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 GCP | Pre-indexed buckets, restored as `botsv5-gcp-main` | `.tgz` archive | [Download BOTS v5 GCP dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-gcp.tgz) |
| BOTS v5 GCP Q&A | Questions and answers for the BOTS v5 GCP exercise | PDF | [Download BOTS v5 GCP Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-gcp-qa.pdf) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-gcp.tgz`. The archive is about 207 MB (about 337 MB extracted). Its SHA-256 checksum is `ce04cc6002185024085848380c66d3aaa14efc32d57afc8d0f4d4aadb7bc57fd`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `defaultdb/` | `botsv5-gcp-main` | `$SPLUNK_DB/botsv5-gcp-main/` | 21 |

The archive contains only the `defaultdb` index. There are no notable, risk, or additional indexes for this scenario.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory contains acceleration summaries and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv5-gcp.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-gcp.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`; the archive should contain 21 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: ce04cc6002185024085848380c66d3aaa14efc32d57afc8d0f4d4aadb7bc57fd
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-gcp-main]
homePath = $SPLUNK_DB/botsv5-gcp-main/db
coldPath = $SPLUNK_DB/botsv5-gcp-main/colddb
thawedPath = $SPLUNK_DB/botsv5-gcp-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-gcp-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-gcp-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-gcp-stage
sudo find /tmp/botsv5-gcp-stage -name journal.gz -type f | wc -l   # expect 21
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-gcp-main'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-gcp-stage/$srcname/$tier"
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
for idx in botsv5-gcp-main; do
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
| tstats count WHERE index=botsv5-gcp-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-gcp-main
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
for idx in botsv5-gcp-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.
