# BOTS v7 APT: Reused Dataset with New Questions

The BOTS v7 APT scenario reuses the **same dataset as the [BOTS v6 APT scenario](../../botsv6/apt/README.md)**. The data is unchanged. Only the questions are new. There is no separate v7 APT archive, and no separate set of v7 APT indexes or sourcetypes.

See the [BOTS v7 overview](../README.md) for the other scenarios, licensing, and content notices.

## Download

Use the BOTS v6 APT archive for the data, and the BOTS v7 Q&A zip for the v7 questions. The v7 Q&A zip contains a separate `.xls` file for each v7 scenario, including new APT questions.

| Dataset | Description | Format | Download |
| --- | --- | --- | --- |
| BOTS v6 APT | Pre-indexed buckets, the dataset used by the v7 APT scenario. See the [v6 APT guide](../../botsv6/apt/README.md) | `.tgz` archive | [Download BOTS v6 APT dataset](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv6/botsv6-apt.tgz) |
| BOTS v7 Q&A | Questions and answers for all BOTS v7 scenarios, including the new APT questions. The zip contains a separate `.xls` file for each scenario | `.zip` archive | [Download BOTS v7 Q&A](https://splunk-bots-datasets.s3.amazonaws.com/BOTSv7/BOTSv7qa.zip) |

## Restoring the dataset

Follow the [BOTS v6 APT restore guide](../../botsv6/apt/README.md). It covers the archive checksum, index definitions, staging, copying buckets, and validation.

The data is restored into the `botsv6-apt-*` indexes (`botsv6-apt-main`, `botsv6-apt-dtex`, and `botsv6-apt-notable`). **There are no `botsv7-apt-*` indexes.** If you have already restored the v6 APT scenario, you do not need to restore anything else. Use the v7 questions against the same indexes.

## Sourcetypes

The sourcetypes and event counts are identical to v6. See [Data sourcetypes included](../../botsv6/apt/README.md#data-sourcetypes-included) in the v6 APT guide.

## Security and content notice

BOTS datasets simulate security incidents and may contain realistic indicators, credentials used in lab scenarios, profanity, or other potentially sensitive or offensive content. Use in an appropriately isolated lab environment.
