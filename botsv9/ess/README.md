# BOTS v9 ESS/SAA: Pre-indexed Dataset Recovery

This guide restores the BOTS v9 ESS/SAA (Enterprise Security and Attack Analyzer) scenario into 3 dedicated indexes on Splunk Enterprise 10.x. See the [BOTS v9 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v9 ESS/SAA dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v9 ESS/SAA | Pre-indexed buckets, restored as `botsv9-ess-main`, `botsv9-ess-notable`, `botsv9-ess-risk` | `.tgz` archive | [Download BOTS v9 ESS/SAA dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv9/botsv9-ess.tgz) |
| BOTS v9 Q&A | Questions and answers for all BOTS v9 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v9 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv9/BOTSv9qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv9-ess.tgz`. The archive is about 181 MB (about 305 MB extracted). Its SHA-256 checksum is `229df3895b302dbaed6076435808bca1b64a593bf794482add5deef24a4b2c9d`.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv9-ess-main` | `$SPLUNK_DB/botsv9-ess-main/` | 303 MB | 9 |
| `notabledb/` | `botsv9-ess-notable` | `$SPLUNK_DB/botsv9-ess-notable/` | 50 KB | 2 |
| `riskdb/` | `botsv9-ess-risk` | `$SPLUNK_DB/botsv9-ess-risk/` | 1.1 MB | 3 |

The `defaultdb/` archive directory is restored as `botsv9-ess-main`, `notabledb/` as `botsv9-ess-notable`, and `riskdb/` as `botsv9-ess-risk`.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directory (under `defaultdb/`) contains acceleration summaries and is not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 305 MB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv9-ess.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv9-ess.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep -E '^[^/]+/db/db_[^/]+/$'
```

These instructions expect archive paths like `defaultdb/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`. The archive should contain 14 bucket directories. Its journals are zstd-compressed (`rawdata/journal.zst`) rather than `journal.gz`. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 229df3895b302dbaed6076435808bca1b64a593bf794482add5deef24a4b2c9d
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv9-ess-main]
homePath = $SPLUNK_DB/botsv9-ess-main/db
coldPath = $SPLUNK_DB/botsv9-ess-main/colddb
thawedPath = $SPLUNK_DB/botsv9-ess-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv9-ess-notable]
homePath = $SPLUNK_DB/botsv9-ess-notable/db
coldPath = $SPLUNK_DB/botsv9-ess-notable/colddb
thawedPath = $SPLUNK_DB/botsv9-ess-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv9-ess-risk]
homePath = $SPLUNK_DB/botsv9-ess-risk/db
coldPath = $SPLUNK_DB/botsv9-ess-risk/colddb
thawedPath = $SPLUNK_DB/botsv9-ess-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv9-ess-main botsv9-ess-notable botsv9-ess-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv9-ess-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv9-ess-stage
sudo find /tmp/botsv9-ess-stage -type d -name 'db_*' | wc -l   # expect 14
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv9-ess-main' 'notabledb:botsv9-ess-notable' 'riskdb:botsv9-ess-risk'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv9-ess-stage/$srcname/$tier"
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
for idx in botsv9-ess-main botsv9-ess-notable botsv9-ess-risk; do
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
| tstats count WHERE index=botsv9-ess-* earliest=0 BY index
| sort index
```

```spl
| tstats count WHERE index=botsv9-ess-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv9-ess-*
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
for idx in botsv9-ess-main botsv9-ess-notable botsv9-ess-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following sourcetypes were reported by the source Splunk instance for the indexes whose buckets are restored by this procedure. Counts reflect the supplied inventory and may differ on another restoration.

### `index=main` (restored as `index=botsv9-ess-main`)

**43 sourcetypes**, 679,678 events.

| Sourcetype | Events |
| --- | ---: |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 158,865 |
| `pfsense:fw` | 133,621 |
| `o365:management:activity` | 104,324 |
| `WindowsUpdateLog` | 84,857 |
| `stream:dns` | 70,856 |
| `stream:ip` | 37,010 |
| `XmlWinEventLog` | 24,462 |
| `splunk:aa:forensic:registrykeys` | 12,510 |
| `stream:tcp` | 11,535 |
| `stream:udp` | 7,638 |
| `XmlWinEventLog:Security` | 7,266 |
| `stream:http` | 5,737 |
| `splunk:aa:forensic:urls` | 3,983 |
| `slack:messages` | 3,318 |
| `splunk:aa:forensic:http` | 3,259 |
| `splunk:aa:forensic:files` | 3,055 |
| `Script:ListeningPorts` | 1,346 |
| `splunk:aa:forensic:detections` | 787 |
| `powershell` | 728 |
| `splunk:aa:forensic:hosts` | 705 |
| `splunk:aa:forensic:strings` | 568 |
| `splunk:aa:job:task` | 452 |
| `OktaIM2:log` | 446 |
| `Script:InstalledApps` | 371 |
| `o365:reporting:messagetrace` | 346 |
| `XmlWinEventLog:System` | 328 |
| `o365:graph:api` | 311 |
| `XmlWinEventLog:Application` | 252 |
| `splunk:aa:job:resource` | 189 |
| `splunk:aa:forensic:network` | 101 |
| `splunk:aa:job` | 71 |
| `OktaIM2:groupUser` | 62 |
| `Script:TimesyncConfiguration` | 52 |
| `Script:TimesyncStatus` | 52 |
| `slack:audit` | 49 |
| `OktaIM2:group` | 38 |
| `splunk:aa:forensic:processes` | 32 |
| `splunk:aa:forensic:dnsrequests` | 25 |
| `splunk:aa:forensic:tls` | 25 |
| `OktaIM2:user` | 19 |
| `Script:NetworkConfiguration` | 16 |
| `Script:GetEndpointInfo` | 6 |
| `splunk:aa:forensic:mitreattacks` | 5 |

### `index=notable` (restored as `index=botsv9-ess-notable`)

**1 sourcetype**, 4 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 4 |

### `index=risk` (restored as `index=botsv9-ess-risk`)

**1 sourcetype**, 1,744 events.

| Sourcetype | Events |
| --- | ---: |
| `stash` | 1,744 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
