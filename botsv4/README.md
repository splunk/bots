# Boss of the SOC (BOTS) Version 4: Pre-indexed Dataset Recovery

This guide restores historical BOTS v4 bucket files from Splunk Enterprise 7.3.1.1 into a dedicated set of indexes on Splunk Enterprise 10.x. The archive contains pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across these versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v4 dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v4 | Original `main`, `notable`, and `risk` index buckets, restored as `botsv4-main`, `botsv4-notable`, and `botsv4-risk` | `.tgz` archive | [Download BOTS v4 dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv4/botsv4.tgz) |
| BOTS v4 Q&A | Questions and answers for the BOTS v4 exercise | `.zip` archive | [Download BOTS v4 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv4/BOTSv4qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv4.tgz`.

## Index naming

The original BOTS v4 system stored the logical `main` index in the physical `botsv4/` directory, through `SA-bots/default/indexes.conf`. **This restoration intentionally uses new logical index names** to avoid interfering with an existing installation's `main`, `notable`, or `risk` indexes.

| Source logical index | Archive directory | Restored logical index | Destination directory |
| --- | --- | --- | --- |
| `main` | `botsv4/` | `botsv4-main` | `$SPLUNK_DB/botsv4-main/` |
| `notable` | `notabledb/` | `botsv4-notable` | `$SPLUNK_DB/botsv4-notable/` |
| `risk` | `riskdb/` | `botsv4-risk` | `$SPLUNK_DB/botsv4-risk/` |

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The recovered source had 7 bucket directories under `botsv4`, 18 under `notabledb`, and 16 under `riskdb`. Separate `notable/` and `risk/` directories may contain acceleration summaries and are not imported by this procedure.

KV Store, original configurations, apps, and add-ons are not included.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space, and a backup of the destination.
- The recovery archive, named `botsv4.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on the original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv4.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `botsv4/db/db_...`, `notabledb/db/db_...`, and `riskdb/db/db_...`. If the archive has a different layout, adjust the staging paths accordingly. Verify a published SHA-256 checksum if one is provided; none is specified here.

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv4-main]
homePath = $SPLUNK_DB/botsv4-main/db
coldPath = $SPLUNK_DB/botsv4-main/colddb
thawedPath = $SPLUNK_DB/botsv4-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv4-notable]
homePath = $SPLUNK_DB/botsv4-notable/db
coldPath = $SPLUNK_DB/botsv4-notable/colddb
thawedPath = $SPLUNK_DB/botsv4-notable/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv4-risk]
homePath = $SPLUNK_DB/botsv4-risk/db
coldPath = $SPLUNK_DB/botsv4-risk/colddb
thawedPath = $SPLUNK_DB/botsv4-risk/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv4-main botsv4-notable botsv4-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv4-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv4-stage
sudo find /tmp/botsv4-stage -name journal.gz -type f | wc -l
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'botsv4:botsv4-main' 'notabledb:botsv4-notable' 'riskdb:botsv4-risk'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv4-stage/$srcname/$tier"
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
sudo chown -R splunk:splunk \
  "$SPLUNK_DB/botsv4-main" \
  "$SPLUNK_DB/botsv4-notable" \
  "$SPLUNK_DB/botsv4-risk"
```

The archive's `notable/` and `risk/` summary directories are intentionally not copied. Do not import them as event buckets.

## 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv4-main earliest=0 BY sourcetype
| sort -count
```

```spl
| tstats count WHERE index=botsv4-notable earliest=0
```

```spl
| tstats count WHERE index=botsv4-risk earliest=0
```

Inspect registered buckets:

```spl
| dbinspect index=botsv4-main
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
for idx in botsv4-main botsv4-notable botsv4-risk; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following **83 sourcetypes** were reported by the source Splunk instance for the source `index=main` (restored as `index=botsv4-main`). Counts reflect the supplied inventory and may differ on another restoration.

| Sourcetype | Events |
| --- | ---: |
| `stream:ip` | 465,097 |
| `xmlwineventlog:microsoft-windows-powershell/operational` | 412,614 |
| `stream:arp` | 296,077 |
| `stream:dns` | 295,708 |
| `stream:udp` | 288,074 |
| `stream:tcp` | 147,039 |
| `wineventlog` | 122,469 |
| `stream:smb` | 114,051 |
| `bro:dns:json` | 63,793 |
| `bit9:carbonblack:json` | 54,676 |
| `xmlwineventlog:microsoft-windows-sysmon/operational` | 47,614 |
| `stream:icmp` | 43,438 |
| `stream:http` | 33,797 |
| `suricata` | 33,741 |
| `aws:rds:audit` | 28,785 |
| `bro:conn:json` | 27,877 |
| `fgt_traffic` | 24,016 |
| `aws:cloudtrail` | 23,715 |
| `fgt_utm` | 22,312 |
| `stream:igmp` | 19,576 |
| `xmlwineventlog` | 14,601 |
| `aws:cloudwatch` | 9,938 |
| `wineventlog:microsoft-windows-powershell/operational` | 7,575 |
| `suricata:fastlog` | 7,383 |
| `stream:ldap` | 7,330 |
| `bro:files:json` | 6,288 |
| `mguard` | 4,329 |
| `aws:s3:accesslogs` | 3,900 |
| `script:listeningports` | 3,656 |
| `windowsupdatelog` | 3,419 |
| `bro:notice:json` | 3,391 |
| `bro:http:json` | 2,933 |
| `ms:aad:user` | 2,440 |
| `mscs:azure:audit` | 2,376 |
| `bro:ntp:json` | 1,498 |
| `dragos_alert` | 1,423 |
| `o365:management:activity` | 1,417 |
| `bro:modbus:json` | 1,410 |
| `wineventlog:microsoft-windows-taskscheduler/operational` | 1,212 |
| `stream:dhcp` | 1,042 |
| `aws:description` | 1,036 |
| `fgt_event` | 777 |
| `script:installedapps` | 743 |
| `bro:weird:json` | 682 |
| `ms:aad:signin` | 537 |
| `bro:ssl:json` | 415 |
| `xmlwineventlog:microsoft-windows-windows defender/operational` | 413 |
| `bro:dhcp:json` | 327 |
| `stream:snmp` | 317 |
| `gsuiteforsplunk:error` | 316 |
| `wdtap:alerts` | 310 |
| `bro:x509:json` | 260 |
| `ms:o365:reporting:messagetrace` | 208 |
| `gapps:chrome:api` | 155 |
| `aws:cloudwatchlogs:vpcflow` | 142 |
| `script:timesyncconfiguration` | 123 |
| `script:timesyncstatus` | 123 |
| `ms:aad:audit` | 110 |
| `bro:modbus_register_change:json` | 100 |
| `xmlwineventlog:microsoft-windows-taskscheduler/operational` | 99 |
| `aws:config:rule` | 70 |
| `stoq` | 66 |
| `script:getendpointinfo` | 40 |
| `script:networkconfiguration` | 38 |
| `bro:rdp:json` | 37 |
| `winregistry` | 36 |
| `bro:known_hosts:json` | 33 |
| `bro:known_services:json` | 29 |
| `advsim:atr` | 24 |
| `stream:ftp` | 22 |
| `activedirectory` | 18 |
| `bro:ntlm:json` | 16 |
| `bro:smb_mapping:json` | 16 |
| `bro:capture_loss:json` | 13 |
| `bro:software:json` | 13 |
| `bro:known_certs:json` | 7 |
| `powershell` | 5 |
| `aws:cloudwatch:guardduty` | 3 |
| `bro:known_modbus:json` | 3 |
| `aws:config` | 2 |
| `gapps:usage:api` | 2 |
| `wineventlog:security\u0000\u0000` | 2 |
| `bro:pe:json` | 1 |

**Total events in supplied inventory:** 2,659,719. The `wineventlog:security\u0000\u0000` value is preserved as supplied; it represents a sourcetype with escaped NUL characters and may display differently in Splunk.

## Licensing

This archive contains **pre-indexed Splunk data**. Copying existing buckets does not itself re-ingest the original raw events. Splunk Enterprise licensing and any app/add-on licensing remain subject to their respective terms. Re-indexing exported raw events would be a different workflow.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.

## Scope of this guide

This README documents recovery of the original `main`, `notable`, and `risk` event buckets into dedicated destination indexes `botsv4-main`, `botsv4-notable`, and `botsv4-risk`. It does not include the original KV Store, configuration files, saved searches, dashboards, data models, or required apps/add-ons. Those components may be needed to reproduce the complete original BOTS v4 experience.

## Authors

Written in 2019 by Ryan Kovar, David Herrald, James Brodsky, John Stoner, Jim Apger, David Veuve, Lily Lee, and Tom Smit.
