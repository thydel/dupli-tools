# Duplicity tools to maintain a secondary backup

## The context

- I use [duplicity][] do backup a set a *node/volume* on a primary backup
  storage host
- I use rsync to copy primary backup on a secondary backup storage
  host at a distant location

## The need

- The [duplicity][] setup takes care of removing old backup but
  sometimes the balance between available space for backup and growth
  of the data to backup impose to manually remove some older and
  bigger backup to avoid backup space exhaustion
- The secondary backup is not managed by [duplicity][] but still
  required supression of older backups and examination of who take
  how many storage space

[duplicity]: http://duplicity.nongnu.org/ "duplicity"

## The tools

Provided the backup space is organised as a two level *node/volume*
filetree containing the usual [duplicity][] file sets for full and
incremental backup the tools allows

- To segment backup sets in three groups (`small`, `medium` and `big`)
  depending on the size of the set
- To list backup sets with their size and summed size
- To list the [duplicity][] file sets of a backup set
- To select [duplicity][] file sets from a backup set older or newer
  than a date

# Usage

```
./dupli-install
dupli help
dupli sets help
dupli files help
dupli sync help
dupli loop help
```

## sets

```console
$ dupli sets help

dupli-sets [list] [sum] [rev] [help|all|small|medium|big]

list sets of volumes (<node>/<vol>) by size segment
use cmd
	"small"  to list vol <= 9G
	"medium" to list vol > 9G && <= 99G
	"big"    to list vol > 99G
	"all"    to list all vol

default to list "all" volumes
default to list reverse sorted by size
default to show size and volume

use option "list" to show volume only (to pipe to other dupli module)
use option "rev" to reverse sort (lower first)
use option "sum" to sum all volume size

options must precede cmd
when using options cmd is required
```

## files

```console
$ dupli files help

dupli-files [short] [newer] [info] [verb] [debug] help|raw|size|list|sets|fulls set=<node>/<vol> [weeks=<n(12)>]

select files older (default) or "newer" (option) than the last full backup make <n> "weeks" ago (default 12)

use cmd "raw" for bare full path list
use cmd "list" for sorted "ls(1) -lt"
         option "short" suppress store prefix
use cmd "size" to get the total size of selected files
use cmd "sets" to reduce output to <node>/<vol> when selected file set is not empty
use cmd "full" to see all full starting points

use option info to show <node>/<vol> and date for "weeks"
```

## sync

```console
$ echo 'secondary.backup := node.domain.tld' | sudo tee /etc/local/duply-local.mk
$ dupli sync help

dupli-sync [time] [dry] [run] [help|main] [BWP=<power of two exponent sequence("8 9 10 11")>]

read a list of volumes (<node>/<vol>) on stdin and generate rsync cmd lines
use cmd "main" to allow options

default to show generated lines

use option "dry" to get dry-run rsync
use option "time" to insert time wrapper
use option "run" to pipe exec generated lines

option time must precede option run

argument BWP is used to compute bandwidth limit in KiB for rsync (eg. BWP="$(seq 8 11)" gives 3840 KiB or 30720 Kibit)
```

## loop

Infinite `rsync` loop using `flock` to enforce synchronization of
concurrent backups sets

No help yet

# Exemples

## List big backup set with size and summed size

```
dupli sets sum big
```

## List duplicity files sets oldest than 3 weeks ago from second biggest backup set

```
dupli sets list big | sed -ne 2p | xargs -i dupli files list set={} weeks=3
```

## Removes all small backup sets older than 3 months

```
dupli sets list small | xargs -i dupli files raw set={} weeks=3 | xargs rm
```

## Show duplicity files sets sizes of medium backup sets older, newer than 2 weeks

```
dupli sets list medium | xargs -i dupli files size set={} weeks=2
dupli sets list medium | xargs -i dupli files newer size set={} weeks=2
```

## Sync big sets to secondary backup using default bandwidth limit

```
dupli sets list big | dupli-sync | dash
```

## Uses loop to continuously sync to secondary preventing concurrent running of different sets

- Mainly to avoid bypassing bandwidth limit
- Use default bandwidth limit and sleep duration between sync

```
for i in small medium big; do dupli loop $i& done
```
