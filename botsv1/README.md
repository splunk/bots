

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
| BOTS v1 | Full BOTS v1 dataset (6.1GB compressed) | Pre-indexed Splunk `.tgz` | [botsv1_data_set.tgz](https://s3.amazonaws.com/botsdataset/botsv1/splunk-pre-indexed/botsv1_data_set.tgz) |
| BOTS v1 (Attack Only) | Much smaller version containing only attack data: "just the needles, no haystack" (135MB compressed) | Pre-indexed Splunk `.tgz` | [botsv1-attack-only.tgz](https://s3.amazonaws.com/botsdataset/botsv1/botsv1-attack-only.tgz) |
| BOTS v1 Q&A | Questions and answers for the BOTS v1 exercise | PDF | [Download BOTS v1 Q&A (placeholder)](https://YOUR-BUCKET.s3.amazonaws.com/botsv1/botsv1-qa.pdf) |


## Warning
**Please be advised that this dataset may contain profanity, slang, vulgar expressions, and/or generally offensive terminology. Please use with discretion.** 

This dataset contains evidence captured during actual computer security incidents, or from realistic lab recreations of security incidents. As such, the dataset **may** contain profanity, slang, vulgar expressions, and/or generally offensive terminology. The authors believe that the educational benefits of preserving the realism of the dataset outweigh the risk of offending some users. If the possibility of encountering this type of offensive material is a concern to you or to any audience with whom you plan to share the dataset, please stop now and do not continue.

## Authors
Written in 2016 by Ryan Kovar, David Herrald, and James Brodsky

## Copyright and License
To the extent possible under law, the author(s) have dedicated
all copyright and related and neighboring rights to this software
to the public domain worldwide. This software is distributed
without any warranty. You should have received a copy of the CC0
Public Domain Dedication along with this software. If not, see
http://creativecommons.org/publicdomain/zero/1.0/.
