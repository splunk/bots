# BOTS v5 K8S: Pre-indexed Dataset Recovery

This guide restores the BOTS v5 K8S scenario into a dedicated index on Splunk Enterprise 10.x. See the [BOTS v5 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v5 K8S dataset will be hosted in an Amazon S3 bucket. The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v5 K8S | Pre-indexed buckets, restored as `botsv5-k8s-main` | `.tgz` archive | [Download BOTS v5 K8S dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-k8s.tgz) |
| BOTS v5 K8S Q&A | Questions and answers for the BOTS v5 K8S exercise | `.zip` archive | [Download BOTS v5 K8S Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv5/botsv5-k8s-qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv5-k8s.tgz`. The archive is about 191 MB. Its SHA-256 checksum is `ebcdcbfe51a6ac6c44a94f8abf563146be70520c596039b83ec336c18d5ac59b`. The link should be finalized after uploading to S3.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `defaultdb/` | `botsv5-k8s-main` | `$SPLUNK_DB/botsv5-k8s-main/` | 2 |

The archive contains only the `defaultdb` index. There are no notable, risk, or additional indexes for this scenario.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory is empty and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv5-k8s.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv5-k8s.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`; the archive should contain 2 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
sha256sum "$ARCHIVE"   # or: shasum -a 256 "$ARCHIVE" on macOS
# expected: ebcdcbfe51a6ac6c44a94f8abf563146be70520c596039b83ec336c18d5ac59b
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv5-k8s-main]
homePath = $SPLUNK_DB/botsv5-k8s-main/db
coldPath = $SPLUNK_DB/botsv5-k8s-main/colddb
thawedPath = $SPLUNK_DB/botsv5-k8s-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv5-k8s-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv5-k8s-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv5-k8s-stage
sudo find /tmp/botsv5-k8s-stage -name journal.gz -type f | wc -l   # expect 2
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv5-k8s-main'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv5-k8s-stage/$srcname/$tier"
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
for idx in botsv5-k8s-main; do
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
| tstats count WHERE index=botsv5-k8s-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv5-k8s-main
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
for idx in botsv5-k8s-main; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following **45 sourcetypes** (1,110,598 events) were reported by the source Splunk instance for `index=main` (restored as `index=botsv5-k8s-main`). Counts reflect the supplied inventory and may differ on another restoration.

| Sourcetype | Events |
| --- | ---: |
| `stream:dns` | 292,666 |
| `stream:ip` | 226,427 |
| `stream:tcp` | 191,799 |
| `stream:http` | 120,187 |
| `stream:udp` | 109,873 |
| `aws:cloudwatchlogs:eks` | 94,470 |
| `fluentd:monitor-agent` | 11,086 |
| `aws:elb:accesslogs` | 9,981 |
| `kube:container:webfrontend` | 9,022 |
| `kube:container:adminfrontend` | 8,757 |
| `kube:objects:service_accounts` | 5,632 |
| `kube:objects:pods` | 5,006 |
| `kube:container:splunk-fluentd-k8s-logs` | 4,924 |
| `stream:mysql` | 2,972 |
| `kube:objects:replica_sets` | 2,817 |
| `kube:container:kubernetes-dashboard` | 2,341 |
| `kube:objects:deployments` | 2,051 |
| `kube:objects:config_maps` | 1,905 |
| `kube:objects:services` | 1,778 |
| `kube:container:dashboard-metrics-scraper` | 1,157 |
| `kube:objects:namespaces` | 889 |
| `kube:objects:daemon_sets` | 663 |
| `kube:objects:events` | 634 |
| `kube:objects:nodes` | 508 |
| `kube:container:cartservice` | 426 |
| `kube:objects:component_statuses` | 381 |
| `kube` | 344 |
| `kube:container:productservice` | 308 |
| `kube:container:nginx` | 282 |
| `kube:container:metrics-server` | 232 |
| `kube:container:splunk-fluentd-k8s-metrics-agg` | 149 |
| `kube:container:alb-ingress-controller` | 146 |
| `kube:kubelet` | 142 |
| `kube:container:ordersservice` | 128 |
| `kube:container:backend-tools` | 104 |
| `aws:cloudtrail` | 101 |
| `kube:docker` | 92 |
| `kube:container:splunk-operator` | 55 |
| `kube:container:splunk-fluentd-k8s-metrics` | 43 |
| `kube:container:kube-hunter` | 41 |
| `kube:container:splunk-fluentd-k8s-objects` | 41 |
| `kube:container:adminfrontend2` | 27 |
| `pod-file-changes` | 6 |
| `kube:container:coredns` | 4 |
| `pod-bash-history` | 1 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
