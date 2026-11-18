# BOTS v6 APT: Pre-indexed Dataset Recovery

This guide restores the BOTS v6 APT scenario into 3 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v6 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v6 APT dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v6 APT | Pre-indexed buckets, restored as `botsv6-apt-main`, `botsv6-apt-dtex`, `botsv6-apt-notable` | `.tgz` archive | [Download BOTS v6 APT dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv6/botsv6-apt.tgz) |
| BOTS v6 Q&A | Questions and answers for all BOTS v6 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v6 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv6/BOTSv6qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv6-apt.tgz`. The archive is about 1.9 GB (about 3.3 GB extracted). Its SHA-256 checksum is `13ec2103e11939485a32904bcf3ed826872e27e39abab003eb434d3976b53c2d`.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv6-apt-main` | `$SPLUNK_DB/botsv6-apt-main/` | 3.2 GB | 5 |
| `dtex/` | `botsv6-apt-dtex` | `$SPLUNK_DB/botsv6-apt-dtex/` | 49 MB | 4 |
| `notabledb/` | `botsv6-apt-notable` | `$SPLUNK_DB/botsv6-apt-notable/` | 37 KB | 3 |

The `defaultdb/` archive directory is restored as `botsv6-apt-main`, and `notabledb/` as `botsv6-apt-notable`. There is no risk index for this scenario.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `defaultdb/` and `dtex/`) contain acceleration summaries and are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 3.3 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv6-apt.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv6-apt.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `dtex/db/db_...`, and `notabledb/db/db_...`. The archive should contain 12 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 13ec2103e11939485a32904bcf3ed826872e27e39abab003eb434d3976b53c2d
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv6-apt-main]
homePath = $SPLUNK_DB/botsv6-apt-main/db
coldPath = $SPLUNK_DB/botsv6-apt-main/colddb
thawedPath = $SPLUNK_DB/botsv6-apt-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-apt-dtex]
homePath = $SPLUNK_DB/botsv6-apt-dtex/db
coldPath = $SPLUNK_DB/botsv6-apt-dtex/colddb
thawedPath = $SPLUNK_DB/botsv6-apt-dtex/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv6-apt-notable]
homePath = $SPLUNK_DB/botsv6-apt-notable/db
coldPath = $SPLUNK_DB/botsv6-apt-notable/colddb
thawedPath = $SPLUNK_DB/botsv6-apt-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv6-apt-main botsv6-apt-dtex botsv6-apt-notable; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv6-apt-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv6-apt-stage
sudo find /tmp/botsv6-apt-stage -name journal.gz -type f | wc -l   # expect 12
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv6-apt-main' 'dtex:botsv6-apt-dtex' 'notabledb:botsv6-apt-notable'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv6-apt-stage/$srcname/$tier"
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
for idx in botsv6-apt-main botsv6-apt-dtex botsv6-apt-notable; do
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
| tstats count WHERE index=botsv6-apt-* earliest=0 BY index
| sort index
```

```spl
| tstats count WHERE index=botsv6-apt-* earliest=0 BY index, sourcetype
| sort index, -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv6-apt-*
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
for idx in botsv6-apt-main botsv6-apt-dtex botsv6-apt-notable; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv6-apt-main`)

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

### `index=dtex` (restored as `index=botsv6-apt-dtex`)

**1 sourcetype**, 8,061 events.

| Sourcetype | Events |
| --- | ---: |
| `dtex_st_activities` | 8,061 |

### `index=notable` (restored as `index=botsv6-apt-notable`)

**1 sourcetype**, 18 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 18 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
