# Boss of the SOC (BOTS)

Boss of the SOC (BOTS) is a hands-on cybersecurity investigation challenge created by Splunk. Participants take on the role of security analysts, investigate realistic attack scenarios, and use Splunk to find evidence, answer questions, and piece together what happened.

BOTS brings together security telemetry from multiple technologies and environments into a single investigative experience. The challenges are designed to develop practical threat-hunting, incident-response, and security-analysis skills, whether you are learning Splunk, teaching a class, researching detection techniques, or practicing in a SOC.

> ### 🔒 Coming soon: dataset downloads
>
> The BOTS datasets are currently locked while we put the finishing touches on this repository. Download links and restore guides are being staged and will be released in the near future. Thank you for your patience. Check back soon, and happy hunting!

## About this repository

This repository is the home for the publicly available BOTS datasets and their documentation. Its goal is to make historical BOTS data accessible for education, research, training, and independent investigation.

The datasets are distributed as pre-indexed Splunk data where available. You can explore the underlying events without participating in a live BOTS competition. The data is separate from the BOTS competition platform, scoreboard, and any commercial Splunk software or third-party apps used to create or analyze it.

Each BOTS version has its own page with download information, installation instructions, dataset details, and credits. **Authors are credited on the individual version pages**, since contributors change from year to year.

## Available datasets

| Version | Documentation | Dataset download |
| --- | --- | --- |
| BOTS v1 | [Version 1](botsv1/README.md) | See version page |
| BOTS v2 | [Version 2](botsv2/README.md) | See version page |
| BOTS v3 | [Version 3](botsv3/README.md) | See version page |
| BOTS v4 | [Version 4](botsv4/README.md) | See version page |
| BOTS v5 | [Version 5](botsv5/README.md) | See version page |
| BOTS v6 | [Version 6](botsv6/README.md) | See version page |
| BOTS v7 | [Version 7](botsv7/README.md) | See version page |
| BOTS v8 | [Version 8](botsv8/README.md) | See version page |
| BOTS v9 | [Version 9](botsv9/README.md) | See version page |

Additional versions can be added as their datasets are prepared for release. A link in this table does not imply that the corresponding dataset has already been published; check its version page for availability.

## Run multiple BOTS versions on one Splunk server

The long-term goal is to make it possible to investigate multiple BOTS datasets on a single Splunk Enterprise instance, without having to deploy a separate server for each version.

To avoid collisions with existing Splunk indexes and other BOTS releases, restored datasets use version-specific index names. For example, BOTS v4 uses:

```text
botsv4-main
botsv4-notable
botsv4-risk
```

This makes it straightforward to investigate one version:

```spl
index=botsv4-* earliest=0
```

Or search primary event data across installed versions that follow the same naming convention:

```spl
index=botsv*-main earliest=0
```

Not every historical release was originally packaged this way. Follow the instructions on each version page for the supported archive layout, index configuration, and restoration steps. Back up any existing Splunk data and review index retention settings before restoring pre-indexed buckets.

## Open datasets and permitted use

BOTS datasets are made available to support the security community. They are intended to be useful for:

- Security education, labs, and classroom exercises
- Threat hunting and incident-response practice
- Detection engineering and analytics research
- Exploring Splunk search and investigation workflows
- Reproducing or extending investigations from historical BOTS challenges

Dataset availability does not mean that Splunk Enterprise, the BOTS competition platform, or third-party applications are open source. Those components have their own licenses and distribution terms.

**Licensing is documented per release.** Refer to each version page and any license included with its download for the terms applicable to that dataset. Do not assume every historical dataset has the same license.

## Installation and compatibility

Each version page provides its own download and installation guidance. Depending on the release, you may need to:

1. Download the archive from the link on the version page.
2. Verify the archive checksum, if one is provided.
3. Configure dedicated indexes and appropriate retention settings.
4. Restore the pre-indexed data into the configured Splunk storage paths.
5. Verify the recovered buckets and search the data using `earliest=0`.

Historical BOTS datasets were created with older Splunk releases. Compatibility with current versions, including Splunk Enterprise 10, may vary by dataset and bucket format. Consult the relevant version page before installing.

## A note about the data

BOTS datasets may contain evidence from simulated attacks, realistic lab environments, or security incidents. Some events may include offensive language, suspicious commands, or other material that reflects the scenarios being investigated. Use discretion when sharing datasets or using them in a classroom or public demonstration.

## Credits

BOTS has been built and maintained by contributors across multiple releases. Because the people involved differ from version to version, **each dataset page contains its own authors and acknowledgments** rather than maintaining a single combined author list here.

## Disclaimer

**The BOTS datasets are provided "as is," without warranty of any kind, and are not supported.** No technical support, maintenance, updates, or service-level commitments are offered for any dataset, archive, or accompanying documentation, whether express or implied. Use of these materials is at your own risk. Refer to the license and terms accompanying each release for the full terms applicable to that dataset.

Please note the following limitations when using these datasets:

- **Out-of-date add-ons.** Many datasets were created with Splunk technical add-ons (TAs) and apps that were current at the time of release but have since been updated, replaced, or retired. Field extractions, CIM mappings, lookups, and dashboards that depend on those add-ons may behave differently with current versions, or may be unavailable.
- **Display and parsing differences.** As a result, some events may not display, parse, or normalize correctly in current Splunk releases. Fields referenced in a challenge's questions or answers may be missing, renamed, or populated differently.
- **Historical timestamps.** Events carry their original timestamps, so searches must use an all-time range (for example, `earliest=0`). Time-relative features, such as dashboards and detections that assume recent data, will not work as expected.
- **Version compatibility.** Pre-indexed bucket compatibility with current Splunk Enterprise versions is not guaranteed. See [Installation and compatibility](#installation-and-compatibility).

These datasets are intended for educational and research purposes. We recommend evaluating them in an isolated, non-production environment.
