# Alloy / Loki Journal Relabeling — Conversation Summary

## Problem

The Alloy container was repeatedly exiting during startup with:

```text
Error: could not perform the initial load successfully
2026/08/19 04:08:35 collector server run finished with error: could not perform the initial load successfully
alloy exited with code 1 (restarting)
```

The relevant Alloy configuration was:

loki.relabel "journal_relabel" {

  forward_to = [loki.write.local.receiver]

  

  rule {

    action        = "replace"

    source_labels = ["__journal__systemd_unit"]

    regex         = "unit"

  }

  

  rule {

    action        = "replace"

    source_labels = ["__journal_syslog_identifier__"]

    regex         = "syslog_identifier"

  }

  

  rule {

    action        = "replace"

    source_labels = ["__journal_priority__"]

    regex         = "priority"

  }

}

  

loki.source.journal "system" {

  forward_to = [loki.write.local.receiver]

  

  labels = {

    job  = "systemd-journal",

    host = "ubuntu-laptop",

  }

}

  

loki.write "local" {

  endpoint {

    url = "http://loki:3100/loki/api/v1/push"

  }

}

## Root Causes

### 1. `replace` rules were missing `target_label`

The `regex` field was incorrectly being used as if it specified the destination label.

For example:

regex = "unit"

does **not** mean "create a label named `unit`".

For a relabel `replace` rule, the destination should be specified with:

target_label = "unit"

The regex is used to match the source value.

### 2. The regex values were incorrect

The configuration used:

regex = "unit"

regex = "syslog_identifier"

regex = "priority"

These would attempt to match literal values rather than rename/copy the journal metadata.

Because the goal is to copy the complete source value into a Loki label, no explicit regex is required; Alloy's default match is sufficient.

### 3. The relabel component was not connected to the journal source

The journal source was sending directly to Loki:

forward_to = [loki.write.local.receiver]

Therefore:

loki.source.journal -> loki.write

and the `loki.relabel` component was bypassed entirely.

The journal source needs to reference the relabel rules with:

relabel_rules = loki.relabel.journal_relabel.rules

### 4. Incorrect internal label name for `SYSLOG_IDENTIFIER`

The configuration used:

__journal_syslog_identifier__

The correct Alloy internal label is:

__journal_syslog_identifier

The `_SYSTEMD_UNIT` and `PRIORITY` labels are:

__journal__systemd_unit

__journal_priority

## Final Configuration

Use:

loki.relabel "journal_relabel" {

  forward_to = []

  

  rule {

    source_labels = ["__journal__systemd_unit"]

    target_label  = "unit"

  }

  

  rule {

    source_labels = ["__journal_syslog_identifier"]

    target_label  = "syslog_identifier"

  }

  

  rule {

    source_labels = ["__journal_priority"]

    target_label  = "priority"

  }

}

  

loki.source.journal "system" {

  forward_to = [loki.write.local.receiver]

  

  relabel_rules = loki.relabel.journal_relabel.rules

  

  labels = {

    job  = "systemd-journal",

    host = "ubuntu-laptop",

  }

}

  

loki.write "local" {

  endpoint {

    url = "http://loki:3100/loki/api/v1/push"

  }

}

## Resulting Data Flow

The corrected pipeline is:

systemd journal

      |

      v

loki.source.journal

      |

      v

loki.relabel

      |

      v

loki.write

      |

      v

Loki

The journal metadata is transformed into Loki labels such as:

job="systemd-journal"

host="ubuntu-laptop"

unit="docker.service"

priority="6"

syslog_identifier="dockerd"

## Useful Queries

Query all system journal logs:

{job="systemd-journal"}

Filter by systemd unit:

{job="systemd-journal", unit="docker.service"}

Filter by syslog identifier:

{job="systemd-journal", syslog_identifier="dockerd"}

## Validation

Before restarting Alloy, validate the configuration from inside the container if the Alloy binary/path is available:

docker exec alloy alloy validate /etc/alloy/config.alloy

Then inspect startup logs:

docker compose logs -f alloy

The main things to verify are that Alloy no longer exits with `could not perform the initial load successfully` and that journal entries appear in Loki with the expected labels.

## Important Cardinality Note

Only promote stable, low-cardinality journal metadata to Loki labels.

The following are reasonable choices for this setup:

- `job`
- `host`
- `unit`
- `syslog_identifier`
- `priority`

Avoid turning highly variable fields such as PID, timestamps, command arguments, or arbitrary journal fields into labels, because high-cardinality labels can significantly increase Loki's storage and query overhead.