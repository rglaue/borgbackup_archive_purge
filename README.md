# borgbackup_archive_purge
Purge archives in a borg repository by week, month, year using a datestamp in the archive name

Borg has its own purge machanisim, `borg prune ...` with the following caveats
that this `borg-purge-archives` program attempts to satisfy:

1. `borg prune` prunes according to the timestamp of when the archive was
created, or as set with `borg create --timestamp=<D>`.

    If there is a process that moves multiple days of content from a remote
system into different archives based on the age of the content, the archives
are recognized by the day they were created, or with the provided value of the
`--timestamp` argument. The timestamp cannot be changed after the archive's
initial creation. [Issue #2962](https://github.com/borgbackup/borg/issues/2961)

    `borg-purge-archives` uses a timestamp that is part of the archive name
to determine the date of the content. The timestamp in the archive name of
course can be changed.

1. `borg prune` only keeps the last archive of the specific time period, and
cannot be redifined to be another day.

    The day that is kept is the last day of the week, month, or year. If the
first day of the week is preferred, that cannot be defined.

    `borg-purge-archives` allows the preferred day of the week, month, year to be
selected. If there is not an archive for the preferred day, a threshold is
evaluated to determine how many days before or after in which an alternative can
be selected. The last available before or first available after the preferred
day within this threshold period is chosen.

1. `borg prune` only evaluates the archive pruning starting backwards from the
day of execution.

    `borg-purge-archives` allows for a starting date to be defined with the
`--start-date` argument. Evaluation begins from the Starting date, and going
backwards. Archives after (younger than) the starting date are purged.

    This is desireable if a copy of the repository is made somewhere, and the
copy is intended to only hold a set of archives within a specific time span. An
example would be if a borg repository is copied with rsync to an off-site
location with the intention to only hold archives for the previous month. On
the second day of the new month, say March 2, 2019, the repository is rsynced.
Then `borg-purge-archives --start-date "2019-02-28" ...` is executed on the
repository copy.

   See the note below about creating copies of borg repositories.

   See the note below about the purging process and candidate archives.


Of course `borg-purge-archives` is not as advanced as `borg prune`. The purge
process currently only evaluates to the day (not hour, minute, second), and
currently only recognizes timestamps of YYYY-MM-DD.

##### Borg Backup wishlist

Life would be easier, and this `borg-purge-archives` is designed to support
them when they eventually happen, when wishes are fulfilled. In addition to
the caveats above, here is the author's wishlist for Borg Backup.

1. Support for the import-tar feature. [Issue #2233](https://github.com/borgbackup/borg/issues/2233)

    This feature will allow a borg repository to be copied. But more precisely
    to only copy the archives wanted. So with this feature, one can init a
    new borg repository, and import Borg Backup archives from one repository
    into the new repository within a certain time range.

    The import-tar feature could be used to make copies of a Borg Backup
    repository, and avoid the need to use rsync to archive copies of them.


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

In addition to the central location for long term backup, the borg repository
is copied and archived onto an off-site disk for longer term cold storage, for
disaster recovery purposes. There was a need to set the start date for pruning
archives from the archived copy of the borg repository so that it maintains
archives within a certain time range.

##### Version 1.3
* 1.3 - better evaluation for ranges overlapping with thresholds
* 1.2 - better evaluation for determining saved archives
* 1.1 - adds support for defining the Starting Date to begin evaluation of purging archives.
* 1.0 - initial version

## Why you would not want to use this program

There are three reasons why, in some distant future, one would not want to use
this program.

1. Borg Backup maintainers added support to change the timestamp of a repository
     archive after its creation.
1. Borg Backup maintainers added support to change the preferred day of week,
     month, year that is desired to be kept.
1. Borg Backup maintainers added support to define the evaluation starting date.

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

    borg-purge-archives version 1.3

    This program will purge archives from a Borg Backup repository based on
    archival dates set in the archive name, e.g. daily-2019-08-01, rather than
    the timestamp recorded by borg when the archive was created or as set with
    `borg create --timestamp=<D>`.

    --help              This help message
    --verbose [0-9]     Set the verbosity level
    --test              Show what would happen without doing it
                          this sets VERBOSE=1 if not set with --verbose
    --start-date <D>    Start on this date for evaluation, as though it was
                          today. Days after this date are purged. The string
                          <D> is any date recognized by the GNU date program.
    --repositiory <dir> (required) The Borg repository to purge from
    --prefix <string>   (required) The name prefix of the archives to consider
    --borg-base-dir <s> Set the BORG_BASE_DIR environment variable. This
                          directory is where borg looks for the .cache/ and
                          .config/ directories for the repository.
    --borg-command <s>  Set the path to the borg command and any other
                          parameters that are desired to be passed in
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

#### Verbosity

The `--verbose N` argument's value is the desired level of verbosity to output
what the program is doing. Generally, this is what the levels output:

* `1` The purged archives
* `2` The saved archives
* `3` The kept archives (those that are weekly, monthly, yearly candidates)
* `4` Things that happened that were not severe enough to become a warning
* `5-7` information about processes and cycles
* `8-9` information about why something didn't happen (e.g. did not purge an archive because it is a weekly candidate)

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

## Note about creating copies of borg repositories

The borg maintainers do not endorse the use of rsync to create a copy of a borg
repository. The reason is because the Repository ID will be duplicated. Having
multiple copies of a repository with the same ID on the same machine will be
problematic.

See the Borg Backup FAQ question "Can I copy or synchronize my repo to another location?"
https://borgbackup.readthedocs.io/en/stable/faq.html#can-i-copy-or-synchronize-my-repo-to-another-location

The recommended way to keep a second copy of the repository is to backup the
client machine twice, once to each repository. However, this is not ideal
when archiving a borg repository to an off-site storage, because it
requires the off-site storage to be online all the time, and be relatively
fast.

Archiving borg respositories can be achieved with rsync when taking special
care to not cause conflict in the borg repository config and cache.

The `.cache` and `.config` directories are kept in `BORG_BASE_DIR/`. If the
`BORG_BASE_DIR` variable is changed for every copy of the repository, then it
is possible to create multiple copies with the same Repository ID using rsync.
Here is an example script to access a borg repository copy:

```bash
#!/bin/bash

THIS_PATH=$(readlink -f $0)
THIS_DIR=$(dirname $THIS_PATH)

export BORG_BASE_DIR=${THIS_DIR}

borg $*
```

Name this script `localborg`, and place it in the system at
`BORG_BASE_DIR/localborg`. When working with the borg repository, be sure to
always use the `localborg` for the copy of the repository you reference. For
example, this might be the file system tree:
```bash
  + /var/backup/borg-archives/
  |--+ 2019-02/
  |  |--- .cache
  |  |--- .config
  |  |--- borgrepo
  |  `--- localborg
  `--+ 2019-03/
     |--- .cache
     |--- .config
     |--- borgrepo
     `--- localborg
```

When you access the repository copy with `localborg` the first time, you will
receive a warning message you can answer yes to:
```bash
linux ~# cd /var/backup/borg-archives/2019-02/
linux 2019-02# ./localborg list borgrepo
Warning: The repository at location /var/backup/borg-archives/2019-02/borgrepo was previously located at /var/backup/borg/MyRepo
Do you want to continue? [yN]
```

After the archived copy of the borg repository is created with rsync, it can
then be purged with a command like one of the following:
```bash
borg-purge-archive \
    --start-date "2019-02-28" \
    --repository /var/backup/borg-archives/2019-02/borgrepo \
    --borg-command /var/backup/borg-archives/2019-02/localborg \
    --prefix "daily-"
```
```bash
borg-purge-archive \
    --start-date "2019-02-28" \
    --repository /var/backup/borg-archives/2019-02/borgrepo \
    --borg-base-dir /var/backup/borg-archives/2019-02 \
    --prefix "daily-"
```
```bash
export BORG_BASE_DIR=/var/backup/borg-archives/2019-02
borg-purge-archive \
    --start-date "2019-02-28" \
    --repository /var/backup/borg-archives/2019-02/borgrepo \
    --prefix "daily-"
```

## Note about time period archiving

The `borg-purge-archives` does not retain a certain number of daily, weekly,
monthly, or yearly archives. Instead it retains these archives which fall
within a certain keep range (time period) of days defined by `KEEP_*`
variables. If a particular archive falls outside of a keep range (time
period) of days, it is no longer considered for that keep range and will be
purged. Within each daily, weekly, monthly keep range, the program looks for
the archive on or closest to the preferred day of that keep range.

```bash
# Peroid of days, starting from today, to analyze for archiving.
# Each starts from 0 to the day given
KEEP_DAYS=90
KEEP_WEEKS=185
KEEP_MONTHS=366
KEEP_YEARS=3660
```

Assuming there is a borg archive for every day, for the last 365 days,
including today. The value `KEEP_DAYS=90` will keep every archive for the most
recent 91 days, or 91 days starting from the date provided in the
`--start-date` argument. Today is day 0, and yesterday is 1 day ago.

If there were 5 daily archives missing during the time from yesterday (1 days
ago) until 90 days ago, there would be 86 archives. And 86 archives would be
retained because we are retaining all archives between day 0 and day 90.

Only one archive is retained for the time periods of WEEKS, MONTHS, and YEARS.
So the preferred day to retain for each time period must be defined. The
preferred day is identified as the day number for each period.

```bash
# Preferred day of saved archive to keep
KEEP_WEEK_DAY=1  # Monday=1, Sunday=7
KEEP_MONTH_DAY=1 # First day of the month
KEEP_YEAR_DAY=1  # First day of the year
```

If there is not an archive available for every day, then an archive for the
preferred day may not be available. The threshold values are used for each time
period to allow archives preceeding and following the preferred day to be
alternatively chosen.

See the above section "Configuring daily, weekly, monthly, yearly keep and threshold values"

## Note about the purging process and candidate archives

This program goes backwards in history, beginning the evaluation of archives
with the newest. For daily keep range, all archives are preferred. For weekly,
monthly, and yearly archives, a preferred day is defined for the day
that is desired to be kept within that time period. Other archives that are
not the preferred may be kept as candidates until determined they do not
satisfy the preferred day criteria.

Since this program uses a threshold value to help determine what archives to
keep, it uses a candidacy process. An archive within a threshold for weekly,
monthly, or yearly can become a candidate for the weekly, monthly, or yearly
archive to save permanently.

The program will look for an archive that is datestamped for the preferred
weekly, monthly, or yearly day desired to be kept. However, it will observe
archives before and after the preferred day, which lay within the defined
threshold, as potential candidates to be this archive. Should there not be an
archive for the preferred day, the candidate is saved as the alternate
preferred archive.

This program prefers the candidate archive with a datestamp closest to the
preferred day, which follows the preferred day. Should there be no candidate
following the preferred day, a candidate archive before the preferred day is
chosen. Again, the archive datestamped closest to the preferred day is chosen.

Example: If the preferred day of month is `1`, and the month threshold is `10`,
this application will begin keeping candidates starting on day `11` of the
month. As it gets closer to day `1`, it will keep that next day's archive as
the candidate, and purge the previous candidate. If there is no archive on days
`1-11`, then this program begins keeping archives on the last day of the
previous month as candidates. Let's say the last day is `31`. It will keep
archives from days `31-22`. The first candidate archive kept after the day `1`
is the one saved. The remaining are purged.


## License

Copyright (C) 2019 [Center for the Application of Information Technologies](http://www.cait.org),
[Western Illinois University](http://www.wiu.edu). All rights reserved.

Apache License 2.0, see [LICENSE](https://github.com/prometheus/haproxy_exporter/blob/master/LICENSE).

This program is free software.
