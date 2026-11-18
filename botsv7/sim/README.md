# BOTS v7 SIM: Pre-indexed Dataset Recovery

This guide restores the BOTS v7 SIM (Splunk Infrastructure Migration) scenario into 2 dedicated indexes on Splunk Enterprise 10.x: one event index and one metrics index. See the [BOTS v7 overview](../README.md) for the other scenarios, licensing, and content notices.

> Compatibility: Test on a disposable Splunk 10.x instance first. Bucket compatibility across versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

## Download

The pre-indexed BOTS v7 SIM dataset is hosted in an Amazon S3 bucket.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v7 SIM | Pre-indexed buckets, restored as `botsv7-sim-main`, `botsv7-sim-em_metrics` | `.tgz` archive | [Download BOTS v7 SIM dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv7/botsv7-sim.tgz) |
| BOTS v7 Q&A | Questions and answers for all BOTS v7 scenarios. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v7 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv7/BOTSv7qa.zip) |

Download the archive to your Splunk server before following the installation steps. The examples below expect it at `/tmp/botsv7-sim.tgz`. The archive is about 6.3 GB (about 13.9 GB extracted). Its SHA-256 checksum is `5fea4f79da6dcfc7cbc251cd1b2f8a9214e6a1e0874f3b6deae7c600430d6dee`.

## Index naming

This restoration intentionally uses new logical index names to avoid interfering with an existing installation's indexes.

| Archive directory | Restored logical index | Destination directory | Size | Buckets |
| --- | --- | --- | ---: | ---: |
| `defaultdb/` | `botsv7-sim-main` | `$SPLUNK_DB/botsv7-sim-main/` | 13.5 GB | 15 |
| `em_metrics/` | `botsv7-sim-em_metrics` | `$SPLUNK_DB/botsv7-sim-em_metrics/` | 380 MB | 6 |

The `defaultdb/` archive directory is restored as `botsv7-sim-main`. The `em_metrics/` directory holds a **metrics** index, so its stanza sets `datatype = metric`. The archive has no notable or risk indexes.

The original archive directory names are unchanged. The new names apply to the indexes **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

The archive's `datamodel_summary/` directories (under `defaultdb/` and `em_metrics/`) contain acceleration summaries and are not imported by this procedure.

## Prerequisites

- Splunk Enterprise 10.x installed on a supported Linux host, preferably an isolated test system.
- Shell access, sufficient disk space (about 13.9 GB for the extracted staging copy, plus the same again for the restored indexes), and a backup of the destination.
- The recovery archive, named `botsv7-sim.tgz` in these examples.
- The ability to stop Splunk before copying index buckets.

The commands below assume `/opt/splunk`; change `SPLUNK_HOME` if needed. **Do not run this procedure on an original recovery AMI.** On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

## 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv7-sim.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$'
```

These instructions expect archive paths like `defaultdb/db/db_...` and `em_metrics/db/db_...`. The archive should contain 21 `journal.gz` files. If the layout differs, adjust the staging paths accordingly. Verify the SHA-256 checksum:

```bash
shasum -a 256 "$ARCHIVE"   # or: sha256sum "$ARCHIVE"
# expected: 5fea4f79da6dcfc7cbc251cd1b2f8a9214e6a1e0874f3b6deae7c600430d6dee
```

## 2. Configure dedicated indexes

Add these stanzas to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv7-sim-main]
homePath = $SPLUNK_DB/botsv7-sim-main/db
coldPath = $SPLUNK_DB/botsv7-sim-main/colddb
thawedPath = $SPLUNK_DB/botsv7-sim-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400

[botsv7-sim-em_metrics]
homePath = $SPLUNK_DB/botsv7-sim-em_metrics/db
coldPath = $SPLUNK_DB/botsv7-sim-em_metrics/colddb
thawedPath = $SPLUNK_DB/botsv7-sim-em_metrics/thaweddb
datatype = metric
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

Keep `datatype = metric` on the `em_metrics` index. Metrics buckets restored into an event index will not work.

The retention period is approximately 60 years. The 100 GiB per-index size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long. Keep backups.

Check the effective configuration without starting Splunk:

```bash
for idx in botsv7-sim-main botsv7-sim-em_metrics; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|datatype|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

## 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

sudo mkdir -p /tmp/botsv7-sim-stage
sudo tar -xzf "$ARCHIVE" -C /tmp/botsv7-sim-stage
sudo find /tmp/botsv7-sim-stage -name journal.gz -type f | wc -l   # expect 21
```

Do not extract directly over an existing index. Ensure Splunk is fully stopped before importing buckets.

## 4. Copy buckets into the new indexes

These commands copy each original source directory's `db` and `colddb` buckets to its new dedicated index. They **refuse to copy into nonempty destination tiers**, rather than risking a bucket collision or accidental merge.

```bash
set -e

for mapping in 'defaultdb:botsv7-sim-main' 'em_metrics:botsv7-sim-em_metrics'; do
  srcname=${mapping%%:*}
  dstname=${mapping#*:}

  for tier in db colddb; do
    src="/tmp/botsv7-sim-stage/$srcname/$tier"
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
for idx in botsv7-sim-main botsv7-sim-em_metrics; do
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
| tstats count WHERE index=botsv7-sim-main earliest=0 BY sourcetype
| sort -count
```

The `em_metrics` index holds metrics rather than events, so it does not appear in event searches. Check it with `mstats` or `mcatalog`:

```spl
| mcatalog values(metric_name) AS metric_name WHERE index=botsv7-sim-em_metrics earliest=0
```

Inspect registered buckets:

```spl
| dbinspect index=botsv7-sim-*
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
for idx in botsv7-sim-main botsv7-sim-em_metrics; do
  "$SPLUNK_HOME/bin/splunk" btool indexes list "$idx" --debug | \
    grep -E 'homePath|coldPath|thawedPath|datatype|frozenTimePeriodInSecs|maxTotalDataSizeMB'
done
```

Check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete.

## Data sourcetypes included

The following **267 sourcetypes** (20,936,595 events) were reported by the source Splunk instance for `index=main` (restored as `index=botsv7-sim-main`). The `em_metrics` index holds metrics, not events, so it has no sourcetype table. Counts reflect the supplied inventory and may differ on another restoration.

| Sourcetype | Events |
| --- | ---: |
| `WinEventLog` | 11,702,008 |
| `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` | 8,618,404 |
| `aws:cloudwatchlogs:vpcflow` | 299,325 |
| `google:gcp:pubsub:message` | 66,507 |
| `bro:conn:json` | 55,611 |
| `bro:tunnel:json` | 53,308 |
| `suricata` | 47,922 |
| `apache:james` | 27,614 |
| `bro:ssl:json` | 11,747 |
| `bro:dce_rpc:json` | 10,024 |
| `WinEventLog:Microsoft-Windows-PowerShell/Operational` | 7,657 |
| `bro:rdp:json` | 5,843 |
| `bro:dns:json` | 5,747 |
| `bro:files:json` | 4,810 |
| `aws:cloudtrail` | 3,212 |
| `ms:o365:email` | 3,109 |
| `syslog` | 3,064 |
| `bro:http:json` | 2,743 |
| `bro:smb_files:json` | 1,763 |
| `bro:ldap_search:json` | 1,465 |
| `bro:kerberos:json` | 1,053 |
| `bro:ntp:json` | 776 |
| `bro:ldap:json` | 635 |
| `bro:smb_mapping:json` | 611 |
| `apache:error:json` | 471 |
| `bro:ocsp:json` | 350 |
| `apache:access:json` | 168 |
| `apache:modsec:json` | 90 |
| `stoq:results` | 71 |
| `bro:x509:json` | 54 |
| `bro:dhcp:json` | 43 |
| `bro:pe:json` | 40 |
| `bro:notice:json` | 36 |
| `bro:ntlm:json` | 21 |
| `unattended-upgrades` | 15 |
| `letsencrypt-too_small` | 12 |
| `bro:smtp:json` | 9 |
| `user-too_small` | 8 |
| `buffer.b5dc7670e3d6c1ebb819f0ce1041cbc-too_small` | 3 |
| `buffer.b5dc6dad14b362679883b8441b1b55dec-too_small` | 2 |
| `buffer.b5dc6de2e7931e7da6e69d90fe786d8c-too_small` | 2 |
| `buffer.b5dc6f5a4b3cedffc5fb5110b53a9476c-too_small` | 2 |
| `buffer.b5dc714ffc77aa66b2ca2e006d-too_small` | 2 |
| `buffer.b5dc7214ae7771d9a6c6621112b-too_small` | 2 |
| `buffer.b5dc725c9ddad35aed94cc7db947bb4f-too_small` | 2 |
| `buffer.b5dc74c3bc90ad0077a395bfd9bf-too_small` | 2 |
| `buffer.b5dc752f0660555f41c354b4bce1fde-too_small` | 2 |
| `buffer.b5dc75706b3dcee5727dd5039527acc-too_small` | 2 |
| `buffer.b5dc76059a059a25981555d25b771c5cd-too_small` | 2 |
| `buffer.b5dc77161f5dae24e80e73581feb0b-too_small` | 2 |
| `buffer.b5dc77b2c14ec0b65eaa147dd6f58cf-too_small` | 2 |
| `buffer.b5dc79533870ba53b905fd86dba65193e-too_small` | 2 |
| `buffer.b5dc7a368b7cf476ce77edb67d42b716a-too_small` | 2 |
| `buffer.b5dc7aa1d54c279fb0b2917a9be57e57b-too_small` | 2 |
| `buffer.b5dc7be644dd4cc42a2e1cd0b5991c-too_small` | 2 |
| `buffer.b5dc7c0dd8b4d1731748257b74bbe-too_small` | 2 |
| `buffer.b5dc7c4f5824437c9461eb5e8c32914a-too_small` | 2 |
| `buffer.b5dc6db52d790a191e2829bd73657d-too_small` | 1 |
| `buffer.b5dc6dc37dff0ea724faf39d3ee70db7c-too_small` | 1 |
| `buffer.b5dc6dc9279607345312e37417485b56b-too_small` | 1 |
| `buffer.b5dc6dca0c777a010d8202fd60a-too_small` | 1 |
| `buffer.b5dc6dcae216246ae4d710db240af2b-too_small` | 1 |
| `buffer.b5dc6dce665e317f14c400dc724e2c-too_small` | 1 |
| `buffer.b5dc6e0f1f6e317e3fa7038678076257d-too_small` | 1 |
| `buffer.b5dc6e1514dcc8f48a42f0b9e8e-too_small` | 1 |
| `buffer.b5dc6e2b230fc91fb46c546b38ad0d01b-too_small` | 1 |
| `buffer.b5dc6e2bbba8fe730ba09d0798252faa-too_small` | 1 |
| `buffer.b5dc6e3af09e1329e9773b026168db0c-too_small` | 1 |
| `buffer.b5dc6e7039f91f3e48fa1e7636748bde-too_small` | 1 |
| `buffer.b5dc6e708644b7aa3bb3e1b25da20b9d-too_small` | 1 |
| `buffer.b5dc6e71c6b3768082617fce44a6e7b2f-too_small` | 1 |
| `buffer.b5dc6e7871426c38b35ea3ab872b19d-too_small` | 1 |
| `buffer.b5dc6e7a6940b547a60e2bd897919ae-too_small` | 1 |
| `buffer.b5dc6e8b86804e706591dff527a93981d-too_small` | 1 |
| `buffer.b5dc6e9f9cb783300c7f6428156d256c-too_small` | 1 |
| `buffer.b5dc6ecb2c7648dd9b1cc8d190e9bbe5c-too_small` | 1 |
| `buffer.b5dc6ed4c78cc1aad580e65a30ae664a-too_small` | 1 |
| `buffer.b5dc6ed5bbb1a63ff06c7513b8eeae7be-too_small` | 1 |
| `buffer.b5dc6ed7e100d9b14802aed695c21d-too_small` | 1 |
| `buffer.b5dc6ee5b312fab144bf89b097a7e6acb-too_small` | 1 |
| `buffer.b5dc6f024f4bd7625bfaf8fdd23cb9d-too_small` | 1 |
| `buffer.b5dc6f09859a8255ece25d773683e43de-too_small` | 1 |
| `buffer.b5dc6f1eeb81dd4f3f2f1c3ea7fe39d3f-too_small` | 1 |
| `buffer.b5dc6f21038a5366fcc87ac720b2b-too_small` | 1 |
| `buffer.b5dc6f26f7006fa1944b6b0c0d2129c8a-too_small` | 1 |
| `buffer.b5dc6f41428d9f67385c4899af098fe-too_small` | 1 |
| `buffer.b5dc6f50fce705b80a26f193a1d76c23d-too_small` | 1 |
| `buffer.b5dc6f5ab60ea0745379f8d34c5d8070a-too_small` | 1 |
| `buffer.b5dc6f5d9279ffbaf90104adc8fe84e-too_small` | 1 |
| `buffer.b5dc6f5fc70d205e50c2206c10c6bb59e-too_small` | 1 |
| `buffer.b5dc6f62d13ffee61a7d8b0d5dcd0ad-too_small` | 1 |
| `buffer.b5dc6f7353ededde4b2db794c1073b-too_small` | 1 |
| `buffer.b5dc6f7965e70d38a5318af026e7a3fe-too_small` | 1 |
| `buffer.b5dc6f8b2b705a96d689f1e35d500bab-too_small` | 1 |
| `buffer.b5dc6f9fe9b1ed8f73d59aaef233725fb-too_small` | 1 |
| `buffer.b5dc6fa0915b9bf7071a88d81014101d-too_small` | 1 |
| `buffer.b5dc6fa60d28b7206af10f71039e152a-too_small` | 1 |
| `buffer.b5dc6fbc46b975cb47b053a28e73b1d2e-too_small` | 1 |
| `buffer.b5dc6fd8e2edd2cea44656e87011520ab-too_small` | 1 |
| `buffer.b5dc6fda7e773f109d70a9efd8f6c02bc-too_small` | 1 |
| `buffer.b5dc6fe7703e94766e19c54e5e61fd11e-too_small` | 1 |
| `buffer.b5dc70279087c88c2cd7d2e591c369ac-too_small` | 1 |
| `buffer.b5dc70332f8b3a9e4761bedde36b1a-too_small` | 1 |
| `buffer.b5dc70634bd2ae43f700ccc578cc5366e-too_small` | 1 |
| `buffer.b5dc706be144883a544ce9f59e7247a9f-too_small` | 1 |
| `buffer.b5dc70873a6fcd872800a61782a-too_small` | 1 |
| `buffer.b5dc709280c8a7e147a8599b5f99c48c-too_small` | 1 |
| `buffer.b5dc7092da5baa02442ecafb-too_small` | 1 |
| `buffer.b5dc70af76948de5c51c3807c73b2dca-too_small` | 1 |
| `buffer.b5dc70d9d71fe076c21c9dd8d3adf94ee-too_small` | 1 |
| `buffer.b5dc70db570665244e55a7699a7cd5d-too_small` | 1 |
| `buffer.b5dc70dfb0e88ef2a2d7077ba1901ce-too_small` | 1 |
| `buffer.b5dc70fe242e15d2b2ac4176a517b2fb-too_small` | 1 |
| `buffer.b5dc712ef4acef6b1e999f5c152f-too_small` | 1 |
| `buffer.b5dc71696e012c90f27b4b2f9a8dc9efe-too_small` | 1 |
| `buffer.b5dc717e5bce01667a10447c0960ca7b-too_small` | 1 |
| `buffer.b5dc71860a3640c8a9bd971ea1a3e7dab-too_small` | 1 |
| `buffer.b5dc719846f32129782da8681e2d0cce-too_small` | 1 |
| `buffer.b5dc71c25c1e273a039f5910ca6bb1ea-too_small` | 1 |
| `buffer.b5dc71c2b7aac1f49816e8f92e2f056e-too_small` | 1 |
| `buffer.b5dc71d4b7d36be7d242f691e-too_small` | 1 |
| `buffer.b5dc71f767f96239c1b21a3ec4f292fa-too_small` | 1 |
| `buffer.b5dc71f9713358a605c823c02bbd3c63e-too_small` | 1 |
| `buffer.b5dc71fa446584998f0726dbdf68c6bc-too_small` | 1 |
| `buffer.b5dc72104553cab5c7060026bd0a-too_small` | 1 |
| `buffer.b5dc721279e46d76d5796e33308f1ae-too_small` | 1 |
| `buffer.b5dc7239ef666c23c094f7f3764d4d0ab-too_small` | 1 |
| `buffer.b5dc724001a3ae90480a85f9c9b7d6e-too_small` | 1 |
| `buffer.b5dc725cea2635cd269b2f892094b58ee-too_small` | 1 |
| `buffer.b5dc726a73dd84c1c458258cc43956f-too_small` | 1 |
| `buffer.b5dc72ab4b74f977d3c2400ba23c983d-too_small` | 1 |
| `buffer.b5dc72b09b7712d76389a346c8e1b657c-too_small` | 1 |
| `buffer.b5dc72c7b971a468ba2844386516cf2cf-too_small` | 1 |
| `buffer.b5dc731695478c1eb181fbc9f92aa32e-too_small` | 1 |
| `buffer.b5dc7318f72e341409b98f9567e68a5c-too_small` | 1 |
| `buffer.b5dc733331817457fcc5740c2f068f5e-too_small` | 1 |
| `buffer.b5dc7371c6c64b59c2b5e1b41538862f-too_small` | 1 |
| `buffer.b5dc7381df171f5b674e4a1bb47a74ae-too_small` | 1 |
| `buffer.b5dc73a974e93a6666d121e9ddda8d93b-too_small` | 1 |
| `buffer.b5dc73c66a40909f331420466c8871b-too_small` | 1 |
| `buffer.b5dc73ccf92e305a3351c03df2185385f-too_small` | 1 |
| `buffer.b5dc73ed28ed023b0312ef70de4c745c-too_small` | 1 |
| `buffer.b5dc7408682a4c7f2abe5e-too_small` | 1 |
| `buffer.b5dc7409c5249a311d13dedf967130e9e-too_small` | 1 |
| `buffer.b5dc742ad94ffa876e166aa6d-too_small` | 1 |
| `buffer.b5dc743dc0c7abae79142c8a5c861e5c-too_small` | 1 |
| `buffer.b5dc744932763baf910960a-too_small` | 1 |
| `buffer.b5dc745872bdfe1af1ed17d850452d98e-too_small` | 1 |
| `buffer.b5dc7460345d1dd75dd154db052c6c-too_small` | 1 |
| `buffer.b5dc746080a76f2a80e64ced3abf598d-too_small` | 1 |
| `buffer.b5dc748b4a0a231d73f01a085b821f57a-too_small` | 1 |
| `buffer.b5dc7491f971deb896003254b9d9000bf-too_small` | 1 |
| `buffer.b5dc74e058c5d45de4717be70e9f3330d-too_small` | 1 |
| `buffer.b5dc74e949244ba6de1bf7c8f9675c-too_small` | 1 |
| `buffer.b5dc74fc02d292843b261cd366178a-too_small` | 1 |
| `buffer.b5dc750a7eb78fc28a70f8beb81357ff-too_small` | 1 |
| `buffer.b5dc75408e6def846ee6f8e98a4b-too_small` | 1 |
| `buffer.b5dc7543e28045e023924b5ef-too_small` | 1 |
| `buffer.b5dc75716ea63f054f21344ad70b8c6ac-too_small` | 1 |
| `buffer.b5dc7596cb78cd8ebd6773ed0047d-too_small` | 1 |
| `buffer.b5dc759a50326649c9034d8f83e685aaa-too_small` | 1 |
| `buffer.b5dc75b6ec6a139554d99e4ab072cda1c-too_small` | 1 |
| `buffer.b5dc75d66709a282fb9bcbe63e46ec3b-too_small` | 1 |
| `buffer.b5dc75fdbb6729b9865aeb41e84d2d63e-too_small` | 1 |
| `buffer.b5dc762e33398f26e6f6b60fcd2b-too_small` | 1 |
| `buffer.b5dc763198faff9ac32f1a6a09c02f-too_small` | 1 |
| `buffer.b5dc763c538ff9e9c368da1dba8168e-too_small` | 1 |
| `buffer.b5dc76505a82523823ee7cf65b5b0591f-too_small` | 1 |
| `buffer.b5dc766811156fcb7ed74ae8c60d1a-too_small` | 1 |
| `buffer.b5dc7685f02c1facc23bebb5571c7fd-too_small` | 1 |
| `buffer.b5dc768d800ec319d387469267e74ee-too_small` | 1 |
| `buffer.b5dc76a2124f22469406e5f48cf60f2bf-too_small` | 1 |
| `buffer.b5dc76a5b518088651229f478d96cb11b-too_small` | 1 |
| `buffer.b5dc76abf7470b569eda7a50c6c244d-too_small` | 1 |
| `buffer.b5dc76c094650d0628758c424940ba1c-too_small` | 1 |
| `buffer.b5dc76dc2dab58e262f76c2e4f5e0379e-too_small` | 1 |
| `buffer.b5dc76dc79f6fa6165fbf242a4bbefaaa-too_small` | 1 |
| `buffer.b5dc76ee0203b0d41b8c253c245e-too_small` | 1 |
| `buffer.b5dc76feed152b1a746a43a6d91e3f8fc-too_small` | 1 |
| `buffer.b5dc7707c1d09b13e802d11d4bd15cf-too_small` | 1 |
| `buffer.b5dc771bd59936c583695935549a-too_small` | 1 |
| `buffer.b5dc7747777bea75551f2ef15fb72e28b-too_small` | 1 |
| `buffer.b5dc7747c3c5cc89c59d9af267f9e945f-too_small` | 1 |
| `buffer.b5dc7758b34084f07f88ffcb818efcfca-too_small` | 1 |
| `buffer.b5dc776413b2232f68730b991cf-too_small` | 1 |
| `buffer.b5dc777bfa04d6d453872a03f499117df-too_small` | 1 |
| `buffer.b5dc77e0f26aaf7b7ac89c24a31b4e35c-too_small` | 1 |
| `buffer.b5dc781e0b1de6faaadcfdb036481bcd-too_small` | 1 |
| `buffer.b5dc781e5769d91848cd4dc31b41ea-too_small` | 1 |
| `buffer.b5dc783aa759110a83ac4893a5ce-too_small` | 1 |
| `buffer.b5dc783bf69365683c4dd7e02345f-too_small` | 1 |
| `buffer.b5dc7861e198f03182e9bac206e-too_small` | 1 |
| `buffer.b5dc786a2a927c8a7f04617473ea-too_small` | 1 |
| `buffer.b5dc788954f2fe4cbdf8ed821a1db3b-too_small` | 1 |
| `buffer.b5dc7889a13e955e44ea6c6252d8db-too_small` | 1 |
| `buffer.b5dc788f3b1b4c16201d66c2eadcdde-too_small` | 1 |
| `buffer.b5dc78ef4105ff272a28b55f543e-too_small` | 1 |
| `buffer.b5dc78f49ec4c0856f3cefba738e-too_small` | 1 |
| `buffer.b5dc78f4fa4ee16d3b7401479fd533bee-too_small` | 1 |
| `buffer.b5dc790125144653032a-too_small` | 1 |
| `buffer.b5dc79113afbcea291e60b1b635748b-too_small` | 1 |
| `buffer.b5dc794971fc4c119ca3c36c141d0500a-too_small` | 1 |
| `buffer.b5dc7953b0126f0c08e95bbdbfc84a8e-too_small` | 1 |
| `buffer.b5dc795fe8965f9b90aa50ca84fdbf88a-too_small` | 1 |
| `buffer.b5dc7960442242103dc685e8a53ca-too_small` | 1 |
| `buffer.b5dc79984d5d6f02aa82123cf50381af-too_small` | 1 |
| `buffer.b5dc799db9ebd6850d8921772d103fc-too_small` | 1 |
| `buffer.b5dc79a89fdac69b91ebcdc17360f69a-too_small` | 1 |
| `buffer.b5dc79b4ac8dd98ac04b2e0ca43a6f4a-too_small` | 1 |
| `buffer.b5dc79ba932cab431dbffe4a72e4c5fd-too_small` | 1 |
| `buffer.b5dc79cb3267c768a43c2d19fdf801cd-too_small` | 1 |
| `buffer.b5dc79cb8df666703e1b2b2e0d49f-too_small` | 1 |
| `buffer.b5dc79e7dddf94147ba1a557120d-too_small` | 1 |
| `buffer.b5dc7a0ef733d41d13d96de03cb-too_small` | 1 |
| `buffer.b5dc7a23f4b6e5a53b74685c6b34fb09f-too_small` | 1 |
| `buffer.b5dc7a272cb072ac2277e127f09f7d-too_small` | 1 |
| `buffer.b5dc7a2a64a9fc26578d00bcc2267067b-too_small` | 1 |
| `buffer.b5dc7a449e88affab687de88ae226f98b-too_small` | 1 |
| `buffer.b5dc7a474d2e703ff743d3ece1ea6fb3c-too_small` | 1 |
| `buffer.b5dc7a6071f59b8ac9158aa44a38d0f-too_small` | 1 |
| `buffer.b5dc7abe7183753612634162846f7d-too_small` | 1 |
| `buffer.b5dc7ac2e9786378cd86bad7971f35e-too_small` | 1 |
| `buffer.b5dc7ae0d7cfd8617ec0b9f617a55bd7c-too_small` | 1 |
| `buffer.b5dc7b0d1f1fb3030112770295ee7b2c-too_small` | 1 |
| `buffer.b5dc7b0d6b6c1abe00af6d6cc65ab51c-too_small` | 1 |
| `buffer.b5dc7b1816ebc693977637dfcc2c357ed-too_small` | 1 |
| `buffer.b5dc7b6cf6cd5a0caaf7de8185dfc-too_small` | 1 |
| `buffer.b5dc7b7868f35eb749c8e0aa49b3f037a-too_small` | 1 |
| `buffer.b5dc7b78c47db9e20fb9ede33008aab6e-too_small` | 1 |
| `buffer.b5dc7b795d1488ad1f4c44e1ad17c-too_small` | 1 |
| `buffer.b5dc7b9505285b376db04f569b-too_small` | 1 |
| `buffer.b5dc7b9d40da9d8761a501f2f94b3509b-too_small` | 1 |
| `buffer.b5dc7bd0cd461de389a22f10dddc-too_small` | 1 |
| `buffer.b5dc7bd90968adf75b0a21026377adecb-too_small` | 1 |
| `buffer.b5dc7be3b2c3eee67dd0c1ba5a4dd67dc-too_small` | 1 |
| `buffer.b5dc7be40e4ed3d5fbd7e78c8068e0f-too_small` | 1 |
| `buffer.b5dc7c01ddae21f10f1208fc65706507f-too_small` | 1 |
| `buffer.b5dc7c0662ebe97da3a29c574157fc-too_small` | 1 |
| `buffer.b5dc7c1fab1322fe9f3b53f61c26a5e4d-too_small` | 1 |
| `buffer.b5dc7c32aca7ca42b595ced9f5b7301cf-too_small` | 1 |
| `buffer.b5dc7c351dd1514aac85cdaf318823c-too_small` | 1 |
| `buffer.b5dc7c4efc951cf3fc15ead037909495e-too_small` | 1 |
| `buffer.b5dc7c6b98cf76cd4a7e955ba235b-too_small` | 1 |
| `buffer.b5dc7c91deeadbdccee869b94e6ba-too_small` | 1 |
| `buffer.b5dc7c992211b287c805724f10d3eb2f-too_small` | 1 |
| `buffer.b5dc7cba27e1d661b8d37a066fe0e01ac-too_small` | 1 |
| `buffer.b5dc7cc30b6ab967f76d00830216970b-too_small` | 1 |
| `buffer.b5dc7cd5c2ac6d4fb6a33fdc7cb6bce0a-too_small` | 1 |
| `buffer.b5dc7cf62f752a579fb58f31418408ec-too_small` | 1 |
| `buffer.b5dc7d0292dcb87513a5e5d39cd7e35c-too_small` | 1 |
| `buffer.b5dc7d2451c4d58c07e343f130a21b62d-too_small` | 1 |
| `buffer.b5dc7d2571b442c5bfa4152bf9ff20a9a-too_small` | 1 |
| `buffer.q5dc6e2bbba8fe730ba09d0798252faa-too_small` | 1 |
| `buffer.q5dc70af76948de5c51c3807c73b2dca-too_small` | 1 |
| `buffer.q5dc71860a3640c8a9bd971ea1a3e7dab-too_small` | 1 |
| `buffer.q5dc73ccf92e305a3351c03df2185385f-too_small` | 1 |
| `buffer.q5dc7409c5249a311d13dedf967130e9e-too_small` | 1 |
| `buffer.q5dc781e0b1de6faaadcfdb036481bcd-too_small` | 1 |
| `buffer.q5dc783aa759110a83ac4893a5ce-too_small` | 1 |
| `buffer.q5dc788954f2fe4cbdf8ed821a1db3b-too_small` | 1 |
| `buffer.q5dc78f49ec4c0856f3cefba738e-too_small` | 1 |
| `buffer.q5dc79113afbcea291e60b1b635748b-too_small` | 1 |
| `buffer.q5dc7abe7183753612634162846f7d-too_small` | 1 |
| `buffer.q5dc7b0d1f1fb3030112770295ee7b2c-too_small` | 1 |
| `buffer.q5dc7be3b2c3eee67dd0c1ba5a4dd67dc-too_small` | 1 |
| `buffer.q5dc7c4efc951cf3fc15ead037909495e-too_small` | 1 |
| `buffer.q5dc7c6b98cf76cd4a7e955ba235b-too_small` | 1 |

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
