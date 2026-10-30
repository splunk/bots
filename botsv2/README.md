# Boss of the SOC (BOTS) Dataset Version 2
A sample security dataset and CTF platform for information security professionals, researchers, students, and enthusiasts. This page hosts information regarding the version 2 *dataset*.

## Download

The download URLs below are **placeholders** and must be updated before publication.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v2 | Full BOTS v2 dataset (17.6GB compressed, SHA-256 `5f8141ef9bc91741f4f54da1a0a5d01c8fc91eab5a9d26f7ebe020900b3fa083`) | Pre-indexed Splunk `.tgz` | [Download BOTS v2 dataset (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv2/botsv2.tgz) |
| BOTS v2 Q&A | Questions and answers for the BOTS v2 exercise | `.zip` archive | [Download BOTS v2 Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv2/botsv2-qa.zip) |

## Restoring the dataset into a dedicated index

This section restores the pre-indexed BOTS v2 buckets into a dedicated `botsv2-main` index on Splunk Enterprise 10.x, so they can sit alongside other BOTS versions on one server. The archive contains pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: BOTS v2 was built with Splunk Enterprise 7.2.1. Test on a disposable Splunk 10.x instance first. Bucket compatibility across these versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

### Index naming

| Archive directory | Restored index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `botsv2/` | `botsv2-main` | `$SPLUNK_DB/botsv2-main/` | 57 |

The new name applies to the index **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`. The archive's `summary/` directory contains only manifest files and is not used.

### Prerequisites

- Splunk Enterprise 10.x on a supported Linux host, preferably an isolated test system.
- Shell access, enough disk space for the extracted archive (about 23 GB) plus the index, and a backup of the destination.
- The ability to stop Splunk before copying index buckets.
- The apps and add-ons listed in *Required Software* below are only needed for field extractions, not to search the raw events.

The commands assume `/opt/splunk`; change `SPLUNK_HOME` if needed. On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

### 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv2.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$' | wc -l   # expect 57
sha256sum "$ARCHIVE"   # or: shasum -a 256 "$ARCHIVE"
# expected: 5f8141ef9bc91741f4f54da1a0a5d01c8fc91eab5a9d26f7ebe020900b3fa083
```

You should see paths like `botsv2/db/db_...`. If the layout is different, adjust the staging paths below.

### 2. Configure the index

Add this stanza to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv2-main]
homePath = $SPLUNK_DB/botsv2-main/db
coldPath = $SPLUNK_DB/botsv2-main/colddb
thawedPath = $SPLUNK_DB/botsv2-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long.

Check the effective configuration without starting Splunk:

```bash
"$SPLUNK_HOME/bin/splunk" btool indexes list botsv2-main --debug | \
  grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
```

### 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

mkdir -p /tmp/botsv2-stage
tar -xzf "$ARCHIVE" -C /tmp/botsv2-stage
find /tmp/botsv2-stage -name journal.gz -type f | wc -l   # expect 57
```

Do not extract directly over an existing index.

### 4. Copy buckets into the new index

This copies only the bucket directories (`db_*`), and **refuses to copy into a nonempty destination**, rather than risking a bucket collision or accidental merge.

```bash
set -e

src_root=/tmp/botsv2-stage/botsv2

for tier in db colddb; do
  src="$src_root/$tier"
  dst="$SPLUNK_DB/botsv2-main/$tier"
  [ -d "$src" ] || continue

  sudo mkdir -p "$dst"
  if [ -n "$(sudo find "$dst" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
    echo "ERROR: destination is not empty: $dst" >&2
    exit 1
  fi
  sudo find "$src" -mindepth 1 -maxdepth 1 -type d -name 'db_*' -exec cp -a {} "$dst/" \;
done

sudo mkdir -p "$SPLUNK_DB/botsv2-main/thaweddb"
```

Make sure the copied files are owned by the Splunk service account. For example, **if Splunk runs as `splunk`**:

```bash
sudo chown -R splunk:splunk "$SPLUNK_DB/botsv2-main"
```

### 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv2-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv2-main
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
index=botsv2-main earliest=0
```

## Data Sourcetypes included
* access_combined
* activedirectory
* apache:error
* apache_error
* auditd
* bandwidth
* collectd
* cpu
* csp-violation
* df
* ess_content_importer
* hardware
* interfaces
* iostat
* lastlog
* linux:selinuxconfig
* linux_audit
* linux_secure
* ms:o365:management
* msad:nt6:health
* msad:nt6:siteinfo
* mysql:connection:stats
* mysql:database
* mysql:errorlog
* mysql:instance:stats
* mysql:server:stats
* mysql:status
* mysql:table_io_waits_summary_by_index_usage
* mysql:tablestatus
* mysql:transaction:details
* mysql:transaction:stats
* mysql:user
* mysql:variables
* mysqld-8
* netstat
* openports
* osquery_info
* osquery_results
* osquery_warning
* package
* pan:system
* pan:threat
* pan:traffic
* perfmon:cpu
* perfmon:logicaldisk
* perfmon:memory
* perfmon:network
* perfmon:network_interface
* perfmon:ntds
* perfmon:physicaldisk
* perfmon:process
* perfmon:processor
* perfmon:system
* powershell:scriptexecutionsummary
* protocol
* ps
* script:installedapps
* script:listeningports
* stream:arp
* stream:dhcp
* stream:dns
* stream:ftp
* stream:http
* stream:icmp
* stream:ip
* stream:irc
* stream:ldap
* stream:mysql
* stream:smb
* stream:smtp
* stream:tcp
* stream:udp
* suricata
* symantec:ep:agent:file
* symantec:ep:agt_system:file
* symantec:ep:behavior:file
* symantec:ep:packet:file
* symantec:ep:scan:file
* symantec:ep:scm_system:file
* symantec:ep:security:file
* symantec:ep:traffic:file
* syslog
* time
* top
* unix:listeningports
* unix:service
* unix:update
* unix:uptime
* unix:useraccounts
* unix:version
* userswithloginprivs
* vmstat
* web_ping
* weblogic_access_combined
* weblogic_stdout
* who
* windowsupdatelog
* wineventlog:application
* wineventlog:directory-service
* wineventlog:security
* wineventlog:system
* winhostmon
* winregistry
* xmlwineventlog:microsoft-windows-sysmon/operational

## Required Software
The following software is distributed and licensed separately. It is only needed for
field extractions and the original dashboards and content, not to search the raw events. The versions listed are
those that were used to create the dataset. Different versions of the software
may or may not work properly. If you are new to Splunk, follow [these instructions](http://docs.splunk.com/Documentation/Splunk/latest/Installation/Whatsinthismanual) to install the free Splunk Enterprise trial and [these instructions](https://docs.splunk.com/Documentation/AddOns/released/Overview/Singleserverinstall) to install apps and add-ons. 

| App / Add-on | Version | Download |
| ----------- | ------- | -------- |
| Splunk Enterprise                               | 7.2.1  | http://www.splunk.com
| SA-Investigator                                 | 1.3.1  | https://splunkbase.splunk.com/app/3749/
| Base64	                                        | 1.1	   | https://splunkbase.splunk.com/app/1922/
| URL Toolbox	                                    | 1.6	   | https://splunkbase.splunk.com/app/2734/
| Splunk Security Essentials                      | 2.3.0  | https://splunkbase.splunk.com/app/3435/
| JellyFisher	                                    | 0.1.0  | https://splunkbase.splunk.com/app/3626/
| Splunk Common Information Model	                | 4.12.0 | https://splunkbase.splunk.com/app/1621/
| Splunk Add-on for Apache                        | 1.0.0	 | https://splunkbase.splunk.com/app/3186/
| Splunk Add-on for Microsoft Cloud Services      | 2.0.3  | https://splunkbase.splunk.com/app/3110/
| Palo Alto Networks Add-on for Splunk            | 3.8.2	 | https://splunkbase.splunk.com/app/2757/
| Splunk Add-on for Symantec Endpoint Protection  | 2.3.0	 | https://splunkbase.splunk.com/app/2772/
| TA-Suricata	                                    | 2.3.3	 | https://splunkbase.splunk.com/app/2760/
| Microsoft Sysmon Add-on	                        | 6.0.4	 | https://splunkbase.splunk.com/app/1914/
| Collectd App for Splunk Enterprise              | 1.1    | https://splunkbase.splunk.com/app/2875/
| OSquery                                         |	1      | https://splunkbase.splunk.com/app/3278/
| SSL Certificate Checker                         | 3.2	   | https://splunkbase.splunk.com/app/3172/
| Website Monitoring	                            | 2.5    | https://splunkbase.splunk.com/app/1493/	
| Splunk Add-on for Microsoft IIS                 | 1.0.0	 | https://splunkbase.splunk.com/app/3185/
| Splunk Add-on for Unix and Linux                | 6.0.0  | https://splunkbase.splunk.com/app/833/
| Splunk Stream Add-on                            | 7.1.1  | https://splunkbase.splunk.com/app/1809/
| Splunk Add-on for Microsoft Windows             | 5.0.1  | https://splunkbase.splunk.com/app/742/


## Warning
**Please be advised that this dataset may contain profanity, slang, vulgar expressions, and/or generally offensive terminology. Please use with discretion.** 

This dataset contains evidence captured during actual computer security incidents, or from realistic lab recreations of security incidents. As such, the dataset **may** contain profanity, slang, vulgar expressions, and/or generally offensive terminology. The authors believe that the educational benefits of preserving the realism of the dataset outweigh the risk of offending some users. If the possibility of encountering this type of offensive material is a concern to you or to any audience with whom you plan to share the dataset, please stop now and do not continue.

## Authors
Written in 2017 by Ryan Kovar, David Herrald, James Brodsky, John Stoner, Jim Apger, and David Veuve

## Copyright and License
To the extent possible under law, the author(s) have dedicated
all copyright and related and neighboring rights to this software
to the public domain worldwide. This software is distributed
without any warranty. You should have received a copy of the CC0
Public Domain Dedication along with this software. If not, see
http://creativecommons.org/publicdomain/zero/1.0/.
