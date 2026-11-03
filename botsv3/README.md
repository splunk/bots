# Boss of the SOC (BOTS) Dataset Version 3
A sample security dataset and CTF platform for information security professionals, researchers, students, and enthusiasts. This page hosts information regarding the version 3 *dataset*.

## Download

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v3 | Full BOTS v3 dataset (335MB, SHA-256 `bc7519b649a61564d83c435a10cc8652c89aad3614d6053d0ace653861a4bd92`) | Pre-indexed Splunk `.tgz` | [Download BOTS v3 dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv3/botsv3.tgz) |
| BOTS v3 Q&A | Questions and answers for the BOTS v3 exercise | `.zip` archive | [Download BOTS v3 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv3/botsv3qa.zip) |

## Restoring the dataset into a dedicated index

This section restores the pre-indexed BOTS v3 buckets into a dedicated `botsv3-main` index on Splunk Enterprise 10.x, so they can sit alongside other BOTS versions on one server. The archive contains pre-indexed buckets, not raw event files or a Splunk app.

> Compatibility: BOTS v3 was built with Splunk Enterprise 7.1.7. Test on a disposable Splunk 10.x instance first. Bucket compatibility across these versions is not guaranteed. Preserve the original archive and check `splunkd.log` for errors.

### Index naming

| Archive directory | Restored index | Destination directory | Buckets |
| --- | --- | --- | ---: |
| `botsv3/` | `botsv3-main` | `$SPLUNK_DB/botsv3-main/` | 16 |

The new name applies to the index **on the destination Splunk instance**. Do not rename bucket directories such as `db_<newest>_<oldest>_<id>`.

### Prerequisites

- Splunk Enterprise 10.x on a supported Linux host, preferably an isolated test system.
- Shell access, enough disk space for the extracted archive (about 600 MB) plus the index, and a backup of the destination.
- The ability to stop Splunk before copying index buckets.
- The apps and add-ons listed in *Required Software* below are only needed for field extractions, not to search the raw events.

The commands assume `/opt/splunk`; change `SPLUNK_HOME` if needed. On indexer clusters, deploy index definitions through the cluster manager instead of editing individual peers.

### 1. Verify the archive

```bash
export SPLUNK_HOME=/opt/splunk
export SPLUNK_DB="$SPLUNK_HOME/var/lib/splunk"
export ARCHIVE=/tmp/botsv3.tgz

gzip -t "$ARCHIVE"
tar -tzf "$ARCHIVE" | head -40
tar -tzf "$ARCHIVE" | grep '/rawdata/journal.gz$' | wc -l   # expect 16
sha256sum "$ARCHIVE"   # or: shasum -a 256 "$ARCHIVE"
# expected: bc7519b649a61564d83c435a10cc8652c89aad3614d6053d0ace653861a4bd92
```

You should see paths like `botsv3/db/db_...`. If the layout is different, adjust the staging paths below.

### 2. Configure the index

Add this stanza to `$SPLUNK_HOME/etc/system/local/indexes.conf`, merging with any existing contents rather than replacing the file:

```ini
[botsv3-main]
homePath = $SPLUNK_DB/botsv3-main/db
coldPath = $SPLUNK_DB/botsv3-main/colddb
thawedPath = $SPLUNK_DB/botsv3-main/thaweddb
frozenTimePeriodInSecs = 1893456000
maxTotalDataSizeMB = 102400
```

The retention period is approximately 60 years. The 100 GiB size limit is an example, **not** a reservation of disk space. Adjust it for your environment. Size limits can freeze buckets even when age-based retention is long.

Check the effective configuration without starting Splunk:

```bash
"$SPLUNK_HOME/bin/splunk" btool indexes list botsv3-main --debug | \
  grep -E 'homePath|coldPath|thawedPath|frozenTimePeriodInSecs|maxTotalDataSizeMB'
```

### 3. Stop Splunk and stage the archive

```bash
sudo "$SPLUNK_HOME/bin/splunk" stop
ps -ef | grep '[s]plunkd'

mkdir -p /tmp/botsv3-stage
tar -xzf "$ARCHIVE" -C /tmp/botsv3-stage
find /tmp/botsv3-stage -name journal.gz -type f | wc -l   # expect 16
```

Do not extract directly over an existing index.

### 4. Copy buckets into the new index

This copies only the bucket directories (`db_*`), and **refuses to copy into a nonempty destination**, rather than risking a bucket collision or accidental merge.

```bash
set -e

src_root=/tmp/botsv3-stage/botsv3

for tier in db colddb; do
  src="$src_root/$tier"
  dst="$SPLUNK_DB/botsv3-main/$tier"
  [ -d "$src" ] || continue

  sudo mkdir -p "$dst"
  if [ -n "$(sudo find "$dst" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
    echo "ERROR: destination is not empty: $dst" >&2
    exit 1
  fi
  sudo find "$src" -mindepth 1 -maxdepth 1 -type d -name 'db_*' -exec cp -a {} "$dst/" \;
done

sudo mkdir -p "$SPLUNK_DB/botsv3-main/thaweddb"
```

Make sure the copied files are owned by the Splunk service account. For example, **if Splunk runs as `splunk`**:

```bash
sudo chown -R splunk:splunk "$SPLUNK_DB/botsv3-main"
```

### 5. Start Splunk and validate

```bash
sudo "$SPLUNK_HOME/bin/splunk" start
```

Use an all-time search because the events are historical:

```spl
| tstats count WHERE index=botsv3-main earliest=0 BY sourcetype
| sort -count
```

Inspect registered buckets:

```spl
| dbinspect index=botsv3-main
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
index=botsv3-main earliest=0
```

## Data Sourcetypes included
* access_combined
* alternatives
* amazon-ssm-agent
* amazon-ssm-agent-too_small
* apache_error
* aws:cloudtrail
* aws:cloudwatch
* aws:cloudwatch:guardduty
* aws:cloudwatchlogs
* aws:cloudwatchlogs:vpcflow
* aws:config:rule
* aws:description
* aws:elb:accesslogs
* aws:rds:audit
* aws:rds:error
* aws:s3:accesslogs
* bandwidth
* bash_history
* bootstrap
* cisco:asa
* cloud-init
* cloud-init-output
* code42:api
* code42:computer
* code42:org
* code42:security
* code42:user
* config_file
* cpu
* cron-too_small
* df
* dmesg
* dpkg
* error-too_small
* errors
* errors-too_small
* ess_content_importer
* hardware
* history-2
* interfaces
* iostat
* lastlog
* linux_audit
* linux_secure
* localhost-5
* lsof
* maillog-too_small
* ms:aad:audit
* ms:aad:signin
* ms:o365:management
* ms:o365:reporting:messagetrace
* netstat
* o365:management:activity
* openports
* osquery:info
* osquery:results
* osquery:warning
* out-3
* package
* perfmonmk:process
* protocol
* ps
* script:getendpointinfo
* script:installedapps
* script:listeningports
* stream:arp
* stream:dhcp
* stream:dns
* stream:http
* stream:icmp
* stream:igmp
* stream:ip
* stream:mysql
* stream:smb
* stream:smtp
* stream:tcp
* stream:udp
* symantec:ep:agent:file
* symantec:ep:agt_system:file
* symantec:ep:behavior:file
* symantec:ep:packet:file
* symantec:ep:risk:file
* symantec:ep:scm_system:file
* symantec:ep:security:file
* symantec:ep:traffic:file
* syslog
* time
* top
* unix:listeningports
* unix:service
* unix:sshdconfig
* unix:update
* unix:uptime
* unix:useraccounts
* unix:version
* userswithloginprivs
* vmstat
* who
* wineventlog
* winhostmon
* xmlwineventlog:microsoft-windows-sysmon/operational
* yum-too_small


## Required Software
The following software is distributed and licensed separately. It is only needed for
field extractions and the original dashboards and content, not to search the raw events. The versions listed are
those that were used to create the dataset. Different versions of the software
may or may not work properly. If you are new to Splunk, follow [these instructions](http://docs.splunk.com/Documentation/Splunk/latest/Installation/Whatsinthismanual) to install the free Splunk Enterprise trial and [these instructions](https://docs.splunk.com/Documentation/AddOns/released/Overview/Singleserverinstall) to install apps and add-ons.


|	App / Add-on	|	Version	|	Download
| ----------- | ------- | -------- |
| Splunk Enterprise                               | 7.1.7 | http://www.splunk.com
|	Aws_guardduty	                                  |	1.0.4	|	https://splunkbase.splunk.com/app/3790/
|	CiscoNVM	                                      |	1.0.346	|	https://splunkbase.splunk.com/app/2992/
|	Code42 App For Splunk	                          |	3.0.6	|	https://splunkbase.splunk.com/app/3736/
|	Code42ForSplunk Technology Add-On	              |	3.0.4	|	https://splunkbase.splunk.com/app/3746/
|	Splunk Add-on for Cisco ASA	                    |	3.3.0	|	https://splunkbase.splunk.com/app/1620/
|	Splunk Add-on for Microsoft Cloud Services	    |	2.1.0	|	https://splunkbase.splunk.com/app/3110/
|	Splunk Add-on for Microsoft Office 365	        |	1.0.0	|	https://splunkbase.splunk.com/app/4055/
|	Splunk Add-on for Microsoft Windows	            |	4.8.4	|	https://splunkbase.splunk.com/app/742/
|	Splunk Add-on for Symantec Endpoint Protection	|	2.3.0	|	https://splunkbase.splunk.com/app/2772/
|	Splunk Add-on for Tenable	                      |	5.1.3	|	https://splunkbase.splunk.com/app/1710/
|	Splunk Add-on for Unix and Linux	              |	5.2.4	|	https://splunkbase.splunk.com/app/833/
|	Splunk Common Information Model	                |	4.11.0	|	https://splunkbase.splunk.com/app/1621/
|	Splunk Security Essentials	                    |	2.2.0	|	https://splunkbase.splunk.com/app/3435/
|	Splunk Stream Add-on	                          |	7.1.2	|	https://splunkbase.splunk.com/app/1809/
|	TA-VirusTotalActions	                          |	0.2.0	|	https://splunkbase.splunk.com/app/3446/
|	URL Toolbox	                                    |	1.6	  |	https://splunkbase.splunk.com/app/2734/
|	DecryptCommands	                                |	2	|	https://splunkbase.splunk.com/app/2655/
|	Microsoft Azure Active Directory Reporting Add-on for Splunk	|	1.0.1	|	https://splunkbase.splunk.com/app/3757/
|	Microsoft Cloud App for Splunk	                |	1.0.1	|	https://splunkbase.splunk.com/app/3786/
|	Microsoft Office 365 Reporting Add-on for Splunk  |	1.0.1	|	https://splunkbase.splunk.com/app/3720/
|	Microsoft Sysmon Add-on	                        |	8.0.0	|	https://splunkbase.splunk.com/app/1914/
|	OSquery App for Splunk	                        |	0.6.0	|	https://splunkbase.splunk.com/app/3902/
|	Splunk Add-on for AWS	                          |	4.5.0	|	https://splunkbase.splunk.com/app/1876/
|	ES Content Updates	                            |	1.0.25	|	https://splunkbase.splunk.com/app/3449/
|	SA-cim_vladiator	                              |	1.2	|	https://splunkbase.splunk.com/app/2968/


## Warning
**Please be advised that this dataset may contain profanity, slang, vulgar expressions, and/or generally offensive terminology. Please use with discretion.**

This dataset contains evidence captured during actual computer security incidents, or from realistic lab recreations of security incidents. As such, the dataset **may** contain profanity, slang, vulgar expressions, and/or generally offensive terminology. The authors believe that the educational benefits of preserving the realism of the dataset outweigh the risk of offending some users. If the possibility of encountering this type of offensive material is a concern to you or to any audience with whom you plan to share the dataset, please stop now and do not continue.

## Authors
Written in 2018 by Ryan Kovar, David Herrald, James Brodsky, John Stoner, Jim Apger, David Veuve, Lily Lee, and Matt Valites

## Copyright and License
To the extent possible under law, the author(s) have dedicated all copyright and related and neighboring rights to this software to the public domain worldwide. This software is distributed
without any warranty. You should have received a copy of the CC0 Public Domain Dedication along with this software. If not, see http://creativecommons.org/publicdomain/zero/1.0/.

