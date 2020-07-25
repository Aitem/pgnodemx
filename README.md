# pgnodemx

## Overview
SQL functions that allow capture of node OS metrics from PostgreSQL

## Security
Executing role must have been granted pg_monitor membership.

## cgroup Related Functions

### General Access Functions

cgroup virtual files fall into (at least) the following general categories, each with a generic SQL access function:

* BIGINT single line scalar values - ```SELECT cgroup_scalar_bigint(filename);```
  * cgroup v2 examples: cgroup.freeze, cgroup.max.depth, cgroup.max.descendants, cpu.weight, cpu.weight.nice, memory.current, memory.high, memory.low, memory.max, memory.min, memory.oom.group, memory.swap.current, memory.swap.max, pids.current, pids.max
* FLOAT8 single line scalar values - ```SELECT cgroup_scalar_float8(filename);```
  * cgroup v2 examples: cpu.uclamp.max, cpu.uclamp.min
* TEXT single line scalar values - ```SELECT cgroup_scalar_text(filename);```
  * cgroup v2 examples: cgroup.type

* SETOF(BIGINT) multiline scalar values - ```SELECT * FROM cgroup_setof_bigint(filename);```
  * cgroup v2 examples: cgroup.procs, cgroup.threads
* SETOF(TEXT) multiline scalar values - ```SELECT * FROM cgroup_setof_text(filename);```
  * cgroup v2 examples: none

* ARRAY[BIGINT] space separated values - ```SELECT cgroup_array_bigint(filename);```
  * cgroup v2 examples: cpu.max
* ARRAY[TEXT] space separated values - ```SELECT cgroup_array_text(filename)```
  * cgroup v2 examples: cgroup.controllers, cgroup.subtree_control

* SETOF(TEXT, BIGINT) flat keyed - ```SELECT * FROM cgroup_setof_kv(filename);```
  * cgroup v2 examples: cgroup.events, cgroup.stat, cpu.stat, io.pressure, io.weight, memory.events, memory.events.local, memory.stat, memory.swap.events, pids.events

* SETOF(TEXT, TEXT, BIGINT) key/subkey/value space separated - ```SELECT * FROM cgroup_setof_ksv(filename);```
 * cgroup v1 examples: blkio.throttle.io_serviced and blkio.throttle.io_service_bytes

* SETOF(TEXT, TEXT, FLOAT8) nested keyed - ```SELECT * FROM cgroup_setof_nkv(filename);```
  * cgroup v2 examples: memory.pressure, cpu.pressure, io.max, io.stat

In each case, the filename must be in the form ```<controller>.<metric>```, e.g. ```memory.stat```. For more information about cgroup v2 virtual files, See https://www.kernel.org/doc/Documentation/cgroup-v2.txt.

### Get status of cgroup support

* ```SELECT current_setting('pgnodemx.cgroupfs_enabled');```
* Returns boolean result ("on"/"off").
* This value may be explicitly set in postgresql.conf
* However the extension will disable it at runtime if the location pointed to by pgnodemx.cgrouproot does not exist or is not a valid cgroup v1 or v2 mount.

### Get current cgroup mode
```
SELECT cgroup_mode();
```
* Returns the current cgroup mode. Possible values are "legacy", "unified", "hybrid", and "disabled". These correspond to cgroup v1, cgroup v2, mixed, and disabled, respectively.
* Currently "hybrid" mode is not supported; it might be in the future.

### Determine if Running Containerized
```
SELECT current_setting('pgnodemx.containerized');
```
* Returns boolean result ("on"/"off"). The extension attempts to heuristically determine whether PostgreSQL is running under a container, but this value may be explicitly set in postgresql.conf to override the heuristically determined value. The value of this setting influences the cgroup paths which are used to read the cgroup controller files.

### Get cgroup Paths
```
SELECT controller, path FROM cgroup_path();
```
* Returns the path to each supported cgroup controller.

### Get cgroup process count
```
SELECT cgroup_process_count();
```
* Returns the number of processes assigned to the cgroup
* For cgroup v1, based on the "memory" controller cgroup.procs file. For cgroup v2, based on the unified cgroup.procs file.

## Environment Variable Related Functions

### Get Environment Variable as TEXT
```
SELECT envvar_text('PGDATA');
```
* Returns the value of requested environment variable as TEXT

### Get Environment Variable as BIGINT
```
SELECT envvar_bigint('PGPORT');
```
* Returns the value of requested environment variable as BIGINT

## ```/proc``` Related Functions

### Get "/proc/meminfo" as a virtual table

* ```SELECT * FROM proc_meminfo();```

### Get "/proc/self/net/dev" as a virtual table

* ```SELECT * FROM proc_network_stats();```

## System Information Related Functions

### Get file system information as a virtual table

* ```SELECT * FROM fsinfo(path text);```
* Returns type, block_size, blocks, total_bytes, free_blocks, free_bytes, available_blocks, available_bytes, total_file_nodes, free_file_nodes, and mount_flags for the file system on which ```path``` is mounted.

## Kubernetes DownwardAPI Related Functions

### Get status of kdapi_enabled

* ```SELECT current_setting('pgnodemx.kdapi_enabled');```
* Returns boolean result ("on"/"off").
* This value may be explicitly set in postgresql.conf
* However the extension will disable it at runtime if the location pointed to by pgnodemx.kdapi_path does not exist.

### Access "key equals quoted value" files

* ```SELECT * FROM kdapi_setof_kv('filename');```

### Get scalar BIGINT from file

* ```SELECT kdapi_scalar_bigint('filename text');```

## Configuration

* Add pgnodemx to shared_preload_libraries in postgresql.conf.
```
shared_preload_libraries = 'pgnodemx'
```
* The following custom parameters may be set. The values shown are defaults. If the default values work for you, there is no need to add these to ```postgresql.conf```.
```
# enable or disable the cgroup facility
pgnodemx.cgroupfs_enabled = on
# force use of "containerized" assumptions for cgroup file paths
pgnodemx.containerized = off
# specify location of cgroup mount
pgnodemx.cgrouproot = '/sys/fs/cgroup'
# enable cgroup functions
pgnodemx.cgroupfs_enabled = on
# enable or disable the Kubernetes DownwardAPI facility
pgnodemx.kdapi_enabled = on
# specify location of Kubernetes DownwardAPI files
pgnodemx.kdapi_path = '/etc/podinfo'
```
Notes:
* If pgnodemx.cgroupfs_enabled is defined in ```postgresql.conf```, and set to ```off``` (or ```false```), then all cgroup* functions will return NULL, or zero rows, except cgroup_mode() which will return "disabled".
* If ```pgnodemx.containerized``` is defined in ```postgresql.conf```, that value will override pgnodemx heuristics. When not specified, pgnodemx heuristics will determine if the value should be ```on``` or ```off``` at runtime.
* If the location specified by ```pgnodemx.cgrouproot```, default or as set in ```postgresql.conf```, is not accessible (does not exist, or otherwise causes an error when accessed), then pgnodemx.cgroupfs_enabled is forced to ```off``` at runtime and all cgroup* functions will return NULL, or zero rows, except cgroup_mode() which will return "disabled".
* If the location specified by ```pgnodemx.kdapi_path```, default or as set in ```postgresql.conf```, is not accessible (does not exist, or otherwise causes an error when accessed), then pgnodemx.kdapi_enabled is forced to ```off``` at runtime and all kdapi* functions will return NULL, or zero rows.

## Installation

### Compatibility

PostgreSQL version 10 or newer is required.

### Compile and Install

Clone PostgreSQL repository:

```bash
$> git clone https://github.com/postgres/postgres.git
```

Checkout REL_12_STABLE (for example) branch:

```bash
$> git checkout REL_12_STABLE
```

Make PostgreSQL:

```bash
$> ./configure
$> make install -s
```

Change to the contrib directory:

```bash
$> cd contrib
```

Clone ```pgnodemx``` extension:

```bash
$> git clone https://github.com/crunchydata/pgnodemx
```

Change to ```pgnodemx``` directory:

```bash
$> cd pgnodemx
```

Build ```pgnodemx```:

```bash
$> make
```

Install ```pgnodemx```:

```bash
$> make install
```

#### Using PGXS

If an instance of PostgreSQL is already installed, then PGXS can be utilized to build and install ```pgnodemx```.  Ensure that PostgreSQL binaries are available via the ```$PATH``` environment variable then use the following commands.

```bash
$> make USE_PGXS=1
$> make USE_PGXS=1 install
```

### Configure

The following bash commands should configure your system to utilize pgnodemx. Replace all paths as appropriate. It may be prudent to visually inspect the files afterward to ensure the changes took place.

###### Initialize PostgreSQL (if needed):

```bash
$> initdb -D /path/to/data/directory
```

###### Create Target Database (if needed):

```bash
$> createdb <database>
```

###### Install ```pgnodemx``` functions:

Edit postgresql.conf and add ```pgnodemx``` to the shared_preload_libraries line, and change custom settings as mentioned above.

Finally, restart PostgreSQL (method may vary):

```
$> service postgresql restart
```
Install the extension into your database:

```bash
psql <database>
CREATE EXTENSION pgnodemx;
```
