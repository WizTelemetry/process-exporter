# process-exporter
Prometheus exporter that mines /proc to report on selected processes.

[release]: https://github.com/ncabatoff/process-exporter/releases/latest

[![Release](https://img.shields.io/github/release/ncabatoff/process-exporter.svg?style=flat-square")][release]
[![Powered By: GoReleaser](https://img.shields.io/badge/powered%20by-goreleaser-green.svg?branch=master)](https://github.com/goreleaser)
![Build](https://github.com/ncabatoff/process-exporter/actions/workflows/build.yml/badge.svg)

Some apps are impractical to instrument directly, either because you
don't control the code or they're written in a language that isn't easy to
instrument with Prometheus.  We must instead resort to mining /proc.

## Installation

Either grab a package for your OS from the [Releases][release] page, or
install via [docker](https://hub.docker.com/r/ncabatoff/process-exporter/).

## Running

Usage:

```
  process-exporter [options] -config.path filename.yml
```

or via docker:

```
  docker run -d --rm -p 9256:9256 --privileged -v /proc:/host/proc -v `pwd`:/config ncabatoff/process-exporter --procfs /host/proc -config.path /config/filename.yml

```

Important options (run process-exporter --help for full list):

-children (default:true) makes it so that any process that otherwise
isn't part of its own group becomes part of the first group found (if any) when
walking the process tree upwards.  In other words, resource usage of
subprocesses is added to their parent's usage unless the subprocess identifies
as a different group name.

-threads (default:true) means that metrics will be broken down by thread name
as well as group name.

-recheck (default:false) means that on each scrape the process names are
re-evaluated. This is disabled by default as an optimization, but since
processes can choose to change their names, this may result in a process
falling into the wrong group if we happen to see it for the first time before
it's assumed its proper name. You can use -recheck-with-time-limit to enable this
feature only for a specific duration after process starts.

-procnames is intended as a quick alternative to using a config file.  Details
in the following section.

-remove-empty-groups (default:false) forget process groups with no processes.
This is particularly useful if you have some process groups that you expect will 
never return (e.g. if you have process groups named "scan-<scan-id>", and once 
the scan is completed no more process will ever run for that scan again).

To disable any of these options, use the `-option=false`.

## Configuration and group naming

To select and group the processes to monitor, either provide command-line
arguments or use a YAML configuration file.

The recommended option is to use a config file via -config.path, but for
convenience and backwards compatibility the -procnames/-namemapping options
exist as an alternative.

### Using a config file

The general format of the -config.path YAML file is a top-level
`process_names` section, containing a list of name matchers:

```
process_names:
  - matcher1
  - matcher2
  ...
  - matcherN
```

The default config shipped with the deb/rpm packages is:

```
process_names:
  - name: "{{.Comm}}"
    cmdline:
    - '.+'
```

A process may only belong to one group: even if multiple items would match, the
first one listed in the file wins.

(Side note: to avoid confusion with the cmdline YAML element, we'll refer to
the command-line arguments of a process `/proc/<pid>/cmdline` as the array
`argv[]`.)

#### Using a config file: group name

Each item in `process_names` gives a recipe for identifying and naming
processes.  The optional `name` tag defines a template to use to name
matching processes; if not specified, `name` defaults to `{{.ExeBase}}`.

Template variables available:
- `{{.Comm}}` contains the basename of the original executable, i.e. 2nd field in `/proc/<pid>/stat`
- `{{.ExeBase}}` contains the basename of the executable
- `{{.ExeFull}}` contains the fully qualified path of the executable
- `{{.Username}}` contains the username of the effective user
- `{{.Matches}}` map contains all the matches resulting from applying cmdline regexps
- `{{.PID}}` contains the PID of the process.  Note that using PID means the group
  will only contain a single process.
- `{{.StartTime}}` contains the start time of the process.  This can be useful
  in conjunction with PID because PIDs get reused over time.
- `{{.Cgroups}}` contains (if supported) the cgroups of the process
  (`/proc/self/cgroup`). This is particularly useful for identifying to which container
  a process belongs.

Using `PID` or `StartTime` is discouraged: this is almost never what you want,
and is likely to result in high cardinality metrics which Prometheus will have
trouble with.

#### Using a config file: process selectors

Each item in `process_names` must contain one or more selectors (`comm`, `exe`
or `cmdline`); if more than one selector is present, they must all match.  Each
selector is a list of strings to match against a process's `comm`, `argv[0]`,
or in the case of `cmdline`, a regexp to apply to the command line.  The cmdline
regexp uses the [Go syntax](https://golang.org/pkg/regexp).

For `comm` and `exe`, the list of strings is an OR, meaning any process
matching any of the strings will be added to the item's group.

For `cmdline`, the list of regexes is an AND, meaning they all must match.  Any
capturing groups in a regexp must use the `?P<name>` option to assign a name to
the capture, which is used to populate `.Matches`.

Performance tip: give an exe or comm clause in addition to any cmdline
clause, so you avoid executing the regexp when the executable name doesn't
match.

```

process_names:
  # comm is the second field of /proc/<pid>/stat minus parens.
  # It is the base executable name, truncated at 15 chars.
  # It cannot be modified by the program, unlike exe.
  - comm:
    - bash

  # exe is argv[0]. If no slashes, only basename of argv[0] need match.
  # If exe contains slashes, argv[0] must match exactly.
  - exe:
    - postgres
    - /usr/local/bin/prometheus

  # cmdline is a list of regexps applied to argv.
  # Each must match, and any captures are added to the .Matches map.
  - name: "{{.ExeFull}}:{{.Matches.Cfgfile}}"
    exe:
    - /usr/local/bin/process-exporter
    cmdline:
    - -config.path\s+(?P<Cfgfile>\S+)

```

Here's the config I use on my home machine:

```

process_names:
  - comm:
    - chromium-browse
    - bash
    - prometheus
    - gvim
  - exe:
    - /sbin/upstart
    cmdline:
    - --user
    name: upstart:-user

```

### Using -procnames/-namemapping instead of config.path

Every name in the procnames list becomes a process group. The default name of
a process is the value found in the second field of /proc/<pid>/stat
("comm"), which is truncated at 15 chars.  Usually this is the same as the
name of the executable.

If -namemapping isn't provided, every process with a comm value present
in -procnames is assigned to a group based on that name, and any other
processes are ignored.

The -namemapping option is a comma-separated list of alternating
name,regexp values. It allows assigning a name to a process based on a
combination of the process name and command line. For example, using

  -namemapping "python2,([^/]+)\.py,java,-jar\s+([^/]+).jar"

will make it so that each different python2 and java -jar invocation will be
tracked with distinct metrics. Processes whose remapped name is absent from
the procnames list will be ignored. On a Ubuntu Xenian machine being used as
a workstation, here's a good way of tracking resource usage for a few
different key user apps:

  process-exporter -namemapping "upstart,(--user)" \
    -procnames chromium-browse,bash,gvim,prometheus,process-exporter,upstart:-user

Since upstart --user is the parent process of the X11 session, this will
make all apps started by the user fall into the group named "upstart:-user",
unless they're one of the others named explicitly with -procnames, like gvim.

## Group Metrics

There's no meaningful way to name a process that will only ever name a single process, so process-exporter assumes that every metric will be attached
to a group of processes - not a
[process group](https://en.wikipedia.org/wiki/Process_group) in the technical
sense, just one or more processes that meet a configuration's specification
of what should be monitored and how to name it.

All these metrics start with `namedprocess_namegroup_` and have at minimum
the label `groupname`.

### num_procs gauge

Number of processes in this group.

### cpu_seconds_total counter

CPU usage based on /proc/[pid]/stat fields utime(14) and stime(15) i.e. user and system time. This is similar to the node\_exporter's `node_cpu_seconds_total`.

### read_bytes_total counter

Bytes read based on /proc/[pid]/io field read_bytes.  The man page
says

> Attempt to count the number of bytes which this process really did cause to be fetched from the storage layer.  This is accurate for block-backed filesystems.

but I would take it with a grain of salt.

As `/proc/[pid]/io` are set by the kernel as read only to the process' user (see #137), to get these values you should run `process-exporter` either as that user or as `root`. Otherwise, we can't read these values and you'll get a constant 0 in the metric.

### write_bytes_total counter

Bytes written based on /proc/[pid]/io field write_bytes.  As with
read_bytes, somewhat dubious.  May be useful for isolating which processes
are doing the most I/O, but probably not measuring just how much I/O is happening.

### major_page_faults_total counter

Number of major page faults based on /proc/[pid]/stat field majflt(12).

### minor_page_faults_total counter

Number of minor page faults based on /proc/[pid]/stat field minflt(10).

### context_switches_total counter

Number of context switches based on /proc/[pid]/status fields voluntary_ctxt_switches
and nonvoluntary_ctxt_switches.  The extra label `ctxswitchtype` can have two values:
`voluntary` and `nonvoluntary`.

### memory_bytes gauge

Number of bytes of memory used.  The extra label `memtype` can have three values:

*resident*: Field rss(24) from /proc/[pid]/stat, whose doc says:

> This is just the pages which count toward text, data, or stack space.  This does not include pages which have not been demand-loaded in, or which are swapped out.

*virtual*: Field vsize(23) from /proc/[pid]/stat, virtual memory size.

*swapped*: Field VmSwap from /proc/[pid]/status, translated from KB to bytes.

If gathering smaps file is enabled, two additional values for `memtype` are added:

*proportionalResident*: Sum of "Pss" fields from /proc/[pid]/smaps, whose doc says:

> The "proportional set size" (PSS) of a process is the count of pages it has
> in memory, where each page is divided by the number of processes sharing it.

*proportionalSwapped*: Sum of "SwapPss" fields from /proc/[pid]/smaps

### open_filedesc gauge

Number of file descriptors, based on counting how many entries are in the directory
/proc/[pid]/fd.

### deleted_filedesc gauge

Number of deleted but still open file descriptors, based on counting the number of 
entries in the directory /proc/[pid]/fd that contain the (deleted) field.

### worst_fd_ratio gauge

Worst ratio of open filedescs to filedesc limit, amongst all the procs in the
group. The limit is the fd soft limit based on /proc/[pid]/limits.

Normally Prometheus metrics ought to be as "basic" as possible (i.e. the raw
values rather than a derived ratio), but we use a ratio here because nothing
else makes sense. Suppose there are 10 procs in a given group, each with a
soft limit of 4096, and one of them has 4000 open fds and the others all have
40, their total fdcount is 4360 and total soft limit is 40960, so the ratio
is 1:10, but in fact one of the procs is about to run out of fds. With
worst_fd_ratio we're able to know this: in the above example it would be
0.97, rather than the 0.10 you'd see if you computed sum(open_filedesc) /
sum(limit_filedesc).

### oldest_start_time_seconds gauge

Epoch time (seconds since 1970/1/1) at which the oldest process in the group
started.  This is derived from field starttime(22) from /proc/[pid]/stat, added
to boot time to make it relative to epoch.

### num_threads gauge

Sum of number of threads of all process in the group.  Based on field num_threads(20)
from /proc/[pid]/stat.

### states gauge

Number of threads in the group in each of various states, based on the field
state(3) from /proc/[pid]/stat.

The extra label `state` can have these values: `Running`, `Sleeping`, `Waiting`, `Zombie`, `Other`.

## Group Thread Metrics

Since publishing thread metrics adds a lot of overhead, use the `-threads` command-line argument to disable them, 
if necessary.

All these metrics start with `namedprocess_namegroup_` and have at minimum
the labels `groupname` and `threadname`.  `threadname` is field comm(2) from
/proc/[pid]/stat.  Just as groupname breaks the set of processes down into
groups, threadname breaks a given process group down into subgroups.

### thread_count gauge

Number of threads in this thread subgroup.

### thread_cpu_seconds_total counter

Same as cpu_user_seconds_total and cpu_system_seconds_total, but broken down
per-thread subgroup.  Unlike cpu_user_seconds_total/cpu_system_seconds_total,
the label `cpumode` is used to distinguish between `user` and `system` time.

### thread_io_bytes_total counter

Same as read_bytes_total and write_bytes_total, but broken down
per-thread subgroup.  Unlike read_bytes_total/write_bytes_total,
the label `iomode` is used to distinguish between `read` and `write` bytes.

### thread_major_page_faults_total counter

Same as major_page_faults_total, but broken down per-thread subgroup.

### thread_minor_page_faults_total counter

Same as minor_page_faults_total, but broken down per-thread subgroup.

### thread_context_switches_total counter

Same as context_switches_total, but broken down per-thread subgroup.

## Instrumentation cost

process-exporter will consume CPU in proportion to the number of processes in
the system and the rate at which new ones are created.  The most expensive
parts - applying regexps and executing templates - are only applied once per
process seen, unless the command-line option -recheck is provided.

If you have mostly long-running processes process-exporter overhead should be
minimal: each time a scrape occurs, it will parse of /proc/$pid/stat and
/proc/$pid/cmdline for every process being monitored and add a few numbers.

## Dashboards

An example Grafana dashboard to view the metrics is available at https://grafana.net/dashboards/249

## Building

Requires Go 1.21 (at least) installed.
```
make
```

## Exposing metrics through HTTPS

web-config.yml
```
# Minimal TLS configuration example. Additionally, a certificate and a key file
# are needed.
tls_server_config:
  cert_file: server.crt
  key_file: server.key
```
Running
```
$ ./process-exporter -web.config.file web-config.yml &
$ curl -sk https://localhost:9256/metrics | grep process

# HELP namedprocess_scrape_errors general scrape errors: no proc metrics collected during a cycle
# TYPE namedprocess_scrape_errors counter
namedprocess_scrape_errors 0
# HELP namedprocess_scrape_partial_errors incremented each time a tracked proc's metrics collection fails partially, e.g. unreadable I/O stats
# TYPE namedprocess_scrape_partial_errors counter
namedprocess_scrape_partial_errors 0
# HELP namedprocess_scrape_procread_errors incremented each time a proc's metrics collection fails
# TYPE namedprocess_scrape_procread_errors counter
namedprocess_scrape_procread_errors 0
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0.21
# HELP process_exporter_build_info A metric with a constant '1' value labeled by version, revision, branch, and goversion from which process_exporter was built.
# TYPE process_exporter_build_info gauge
process_exporter_build_info{branch="",goversion="go1.17.3",revision="",version=""} 1
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1.048576e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 10
```

For further information about TLS configuration, please visit: [exporter-toolkit](https://github.com/prometheus/exporter-toolkit/blob/master/docs/web-configuration.md)
