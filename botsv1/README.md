

# Boss of the SOC (BOTS) Dataset Version 1
A sample security dataset and CTF platform for information security professionals, researchers, students, and enthusiasts. This page hosts information regarding the version 1 "Dataset." 

## Data Sourcetypes Included
* WinEventLog:Application
* WinEventLog:Security
* WinEventLog:System
* XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
* fgt_event
* fgt_traffic
* fgt_utm
* iis
* nessus:scan
* stream:dhcp
* stream:dns
* stream:http
* stream:icmp
* stream:ip
* stream:ldap
* stream:mapi
* stream:sip
* stream:smb
* stream:snmp
* stream:tcp
* suricata
* winregistry

## Retrieving the BOTS v1 Dataset
The dataset is available in several formats:
### Splunk Indexed

This is recommended way to explore and analyze the BOTS dataset.

Download the pre-indexed dataset and Q&A from the table below.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v1 | Full BOTS v1 dataset (6.5GB compressed, SHA-256 `0d6770d9d832c6ea9ae1d46974491b8487bb7303665e5f10154e17bc3c762c7b`) | Pre-indexed Splunk `.tgz` | [Download BOTS v1 dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv1/botsv1.tgz) |
| BOTS v1 Q&A | Questions and answers for the BOTS v1 exercise | `.zip` archive | [Download BOTS v1 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv1/BOTSv1qa.zip) |


## Restoring the dataset into a dedicated index

This section restores the pre-indexed BOTS v1 buckets into a dedicated `botsv1-main` index on Splunk Enterprise 10.x, so they can sit alongside other BOTS versions on one server (see the main [README](../README.md)). The archive contains pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: BOTS v1 was built with Splunk Enterprise 6.5.2. Test on a disposable Splunk 10.x instance first. Bucket compatibility across these versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

### Index naming

| Archive directory | Restored index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `botsv1/` | `botsv1-main` | `$SPLUNK_DB/botsv1-main/` | 3 |

The new name applies to the index **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`. The archive's `datamodel_summary/` directory is not used.

### Prerequisites

- Splunk Enterprise 10.x on a supported Linux host, preferably an isolated test system.
- Shell access, enough disk space for the extracted archive plus the index (the archive is 6.5GB compressed and about 8.6GB extracted), and a backup of the destination.
- The ability to stop Splunk before copying index buckets.
- The add-ons listed for the original dataset are only needed for field extractions, not to search the raw events.

The commands assume `/opt/splunk`; change `SPLUNK_HOME` if needed. On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

### 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv1.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$' | wc -l   # expect 3
sha256sum "$ARCHIVE"   # or: shasum -a 256 "$ARCHIVE"
# expected: 0d6770d9d832c6ea9ae1d46974491b8487bb7303665e5f10154e17bc3c762c7b
```

You should see paths like `botsv1/db/db_...`. If the layout is different, adjust the staging paths below.

### 2. Configure the index

Add this stanza to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv1-main]
homePath = $SPLUNK_DB/botsv1-main/db
coldPath = $SPLUNK_DB/botsv1-main/colddb
thawedPath = $SPLUNK_DB/botsv1-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long.

Check the effective configuration without starting Splunk:

```bash
"$SPLUNK_HOME/bin/splunk" btool indexes list botsv1-main --debug | \
  grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
```

### 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

mkdir -p /tmp/botsv1-stage
tar -xzf "$ARCHIVE" -C /tmp/botsv1-stage
find /tmp/botsv1-stage -name journal.gz -type f | wc -l   # expect 3
```

Do not extract directly over an existing index.

### 4. Copy buckets into the new index

This copies only the bucket directories (`db_*`), and **refuses to copy into a nonempty destination**, rather than risking a bucket collision or accidental merge.

```bash
set -e

src_root=/tmp/botsv1-stage/botsv1

for tier in db colddb; do
  src="$src_root/$tier"
  dst="$SPLUNK_DB/botsv1-main/$tier"
  [ -d "$src" ] || continue

  sudo mkdir -p "$dst"
  if [ -n "$(sudo find "$dst" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
    echo "ERROR: destination is not empty: $dst" >&2
    exit 1
  fi
  sudo find "$src" -mindepth 1 -maxdepth 1 -type d -name 'db_*' -exec cp -a {} "$dst/" \;
done

sudo mkdir -p "$SPLUNK_DB/botsv1-main/thaweddb"
```

Make sure the copied files are owned by the Splunk service account. For example, **if Splunk runs as `splunk`**:

```bash
sudo chown -R splunk:splunk "$SPLUNK_DB/botsv1-main"
```

### 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv1-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv1-main
| table bucketId state eventCount path
```

If data does not appear, check the logs before attempting another import:

```bash
grep -Ei 'bucket|tsidx|journal|corrupt|incompatible|freeze|frozen' \
  "$SPLUNK_HOME/var/log/splunk/splunkd.log" | tail -100
```

Do not delete the original archive or repeatedly restart against the only copy of the buckets. If Splunk 10.x rejects the files, an intermediate compatible Splunk version or another supported migration method may be necessary.

### 6. Verify retention

Re-run the `btool` command from step 2, and check `splunkd.log` for freezing, deletion, or bucket-validation errors before declaring the restoration complete. Once everything searches correctly, you can search the data with:

```spl
index=botsv1-main earliest=0
```

## Warning
**Please be advised that this dataset may contain profanity, slang, vulgar expressions, and/or generally offensive terminology. Please use with discretion.** 

This dataset contains evidence captured during actual computer security incidents, or from realistic lab recreations of security incidents. As such, the dataset **may** contain profanity, slang, vulgar expressions, and/or generally offensive terminology. The authors believe that the educational benefits of preserving the realism of the dataset outweigh the risk of offending some users. If the possibility of encountering this type of offensive material is a concern to you or to any audience with whom you plan to share the dataset, please stop now and do not continue.

## Authors
Written in 2016 by Ryan Kovar, David Herrald, and James Brodsky

## Copyright and License

Licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file in this directory and the repository [NOTICE](../NOTICE) for details. Unless required by applicable law or agreed to in writing, this software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.

Earlier releases of this version were distributed under the CC0 1.0 Universal Public Domain Dedication. That dedication continues to apply to any copies obtained under it before this relicensing; this Apache License 2.0 applies going forward.
