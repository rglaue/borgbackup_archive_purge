# borgbackup_archive_purge
Purge archives in a borg repository by week, month, year using a datestamp in the archive name

Borg has its own purge machanisim, `borg prune ...` with the following caveats
that this `borg-purge-archives` program attempts to satisfy:

1. `borg prune` prunes according to the timestamp of when the archive was
created.

    If there is a process that moves multiple days of content from a remote
system into different archives based on the age of the content, the archives
are recognized by the day they were created and not the day representing the
age of the content. This is understandable, and could be reconciled if the
borg archive's timestamp could be changed.

    `borg-purge-archives` uses a timestamp that is part of the archive name
to determine the date of the content.

1. `borg prune` only keeps the last archive of the specific time period, and
cannot be redifined to be another day.

    The day that is kept is the last day of the week, month, or year. If the
first day of the week is preferred, that cannot be defined.

    `borg-purge-archives` allows the preferred day of the week, month, year to be
selected. If there is not an achive for the preferred day, a threshold is
evaluated to determine how many days before or after in which an alternative can
be selected. The last available before or first available after the preferred
day within this threshold period is chosen.


Of course `borg-purge-archives` is not as advanced as `borg prune`. The purge
process currently only evaluates down to the day, and currently only recognizes
timestamps of YYYY-MM-DD.

##### Motivation

The motivation for creating `borg-purge-archives` was because the author deals
with multi-site backups which get funneled back to a central location for long
term archiving. Borg Backup is used for long-term archiving, but the actual
daily archives in borg are not created on the same day they are taken from the
end system. Also, due to the long backup processes which begin after midnight,
our archives are stamped the day after the age of the content. And it is the
first day of the week, month, year that we wish to keep, not the last.

So another mechanisim was needed to handle the purge/prune of long-term
daily archives in the borg that did not use the borg archive timestamps, and
which allowed the selection of alternate days for keeping weekly, monthly,
yearly archives.

##### Version 1.0
Version 1.0 is the inital release

## Why you would not want to use this program

There are two reasons why, in some distant future, one would not want to use
this program.

1. Borg Backup maintainers added support to change the repository archive timestamp.
1. Borg Backup maintainers added support to change the preferred day of week, month, year that is desired to be kept.

## Using for your purposes

Currently only general options are configurable with arguments. The options
for configuring preferred keep days and thresholds are configured inside the
script. This is intended to change as this program continues to mature the
use of daily, weekly, monthly, yearly keep and threshold values.

### Configuring daily, weekly, monthly, yearly keep and threshold values

``borg-purge-archives`` evaluates based on historical days. Today being 0,
yesterday being 1, a week ago being 7, etc.. This program does not really
evaluate time periods on calendar weeks, months, and years. Instead it
determines if an archive falls within a keep-day-range, and if it is the
preferred day of the week, month, year (or within the threshold) that is
desired to be kept.

Each keep range begins with 0 and ends with the value given. These numbers
represent days. So "KEEP_WEEKS=185" would only keep weekly archives on the
preferred day of each week for each week within the first 185 days of
evaluation.

    KEEP_DAYS=90
    KEEP_WEEKS=185
    KEEP_MONTHS=366
    KEEP_YEARS=3660

The preferred day of an archive to keep is defined in this value. The limitation
here is that the last day of the month and year cannot be defined with a number.

    KEEP_WEEK_DAY=1  # Monday
    KEEP_MONTH_DAY=1
    KEEP_YEAR_DAY=1

Threshold of days, before/after preferred day, to choose an archive for keeping.
If an archive is not found on the preferred day, indicate a threshold value in
which archives before and after the preferred day are acceptable alternatives.
If no archive is found on the preferred day, nor within the threshold, no
archive is kept.

    PURGE_WEEK_THRESHOLD=6
    PURGE_MONTH_THRESHOLD=27
    PURGE_YEAR_THRESHOLD=352

### Parameters and usage

    `borg-purge-archives` version 1.0

    This program will purge archives from a Borg Backup repository based on
    archival dates set in the archive name, e.g. archive-2019-08-01, rather than
    the actual timestamp recorded by borg for when the archive was created.
    
    The `borg --purge` performs purge operations based on the recorded timestamp
    which is not desired if multiple-day archives are created on a single day,
    or created on an a day that differs from the content in the archive.
    
    --help              This help message
    --verbose [0-9]     Set the verbosity level
    --test		Show what would happen without doing it
                          this sets VERBOSE=1 if not set with --verbose
    --repositiory <dir> (required) The Borg repository to purge from
    --prefix <string>   (required) The name prefix of the archives to consider
    --borg-base-dir=v   Set the BORG_BASE_DIR environment variable. This
                          directory is where borg looks for the .cache/ and
                          .config/ directories for the repository.
    --borg-command=v    Set the path to borg and any other parameters that are
                          desired to be passed in
    # Simulation testing does not require any other arguments, unless the
    # simulate directory will be populated with --sim-populate. However, the
    # --verbose and --test flags are evaluated for simulation runs.
    --simulate <dir>    Perform the purge on a directory containing files
                          that are named as the archive names. This can be
                          populated manually, or with --sim-populate
    --sim-populate      Populate the --simulate directory with empty files
                          named after each archive in the borg repository, then
                          exit. The simulate directory must exist. The --prefix
                          arg is not used, but --repository must be provided.

#### Example

```bash
  borg-purge-archive --test --repository /var/lib/borg --prefix "daily-" --verbose
```

#### Simulation

  `borg-purge-archive --simulate <dir>` will run a simulation test for the
  purpose of either testing the logic of itself, or for testing the purging
  outcome. The simulation will conduct the purging on a directory of files
  with names that are the same as the borg archive names.

  `borg-purge-archive --simulate <dir> --sim-populate --repository <repo>`
  will populate the simulation directory based on the existing archives in a
  given repository. This simulation directory can also be populated with the
  following script:

```bash
TODAY=$(date +%Y-%m-%d)
LASTDAY="2018-10-03"
FPREFIX="archive-daily-"
SIM_DIR="borgtest"

for i in {0..1000}; do
    TDATE=$(date -d "$TODAY -$i day" +%Y-%m-%d)
    if [ -f "$SIM_DIR/${FPREFIX}${TDATE}" ]; then
        continue
    fi
    touch $SIM_DIR/${FPREFIX}${TDATE}
    if [[ $TDATE = $LASTDAY ]]; then
        break
    fi
done
```

## Note about time period archiving

The `borg-purge-archives` does not retain a certain number of daily, weekly,
monthly, or yearly archives. Instead it retains these archives which fall
within a certain keep range of days defined by KEEP_* variables. If a particular
archive falls outside of a keep range of days, it is no longer considered for
that keep range.

## License

Copyright (C) 2018 [Center for the Application of Information Technologies](http://www.cait.org),
[Western Illinois University](http://www.wiu.edu). All rights reserved.

Apache License 2.0, see [LICENSE](https://github.com/prometheus/haproxy_exporter/blob/master/LICENSE).

This program is free software.
