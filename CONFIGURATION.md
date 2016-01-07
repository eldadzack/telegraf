# Telegraf Configuration

## Generating a config file

A default Telegraf config file can be generated using the `-sample-config` flag,
like this: `telegraf -sample-config`

To generate a file with specific inputs and outputs, you can use the
`-input-filter` and `-output-filter` flags, like this:
`telegraf -sample-config -input-filter cpu:mem:net:swap -output-filter influxdb:kafka`

## Plugin Configuration

There are some configuration options that are configurable per plugin:

* **name_override**: Override the base name of the measurement.
(Default is the name of the plugin).
* **name_prefix**: Specifies a prefix to attach to the measurement name.
* **name_suffix**: Specifies a suffix to attach to the measurement name.
* **tags**: A map of tags to apply to a specific plugin's measurements.
* **interval**: How often to gather this metric. Normal plugins use a single
global interval, but if one particular plugin should be run less or more often,
you can configure that here.

### Plugin Filters

There are also filters that can be configured per plugin:

* **pass**: An array of strings that is used to filter metrics generated by the
current plugin. Each string in the array is tested as a glob match against field names
and if it matches, the field is emitted.
* **drop**: The inverse of pass, if a field name matches, it is not emitted.
* **tagpass**: tag names and arrays of strings that are used to filter
measurements by the current plugin. Each string in the array is tested as a glob
match against the tag name, and if it matches the measurement is emitted.
* **tagdrop**: The inverse of tagpass. If a tag matches, the measurement is not
emitted. This is tested on measurements that have passed the tagpass test.

### Plugin Configuration Examples

This is a full working config that will output CPU data to an InfluxDB instance
at 192.168.59.103:8086, tagging measurements with dc="denver-1". It will output
measurements at a 10s interval and will collect per-cpu data, dropping any
fields which begin with `time_`.

```toml
[tags]
  dc = "denver-1"

[agent]
  interval = "10s"

# OUTPUTS
[outputs]
[[outputs.influxdb]]
  url = "http://192.168.59.103:8086" # required.
  database = "telegraf" # required.
  precision = "s"

# PLUGINS
[plugins]
[[inputs.cpu]]
  percpu = true
  totalcpu = false
  # filter all fields beginning with 'time_'
  drop = ["time_*"]
```

### Plugin Config: tagpass and tagdrop

```toml
[plugins]
[[inputs.cpu]]
  percpu = true
  totalcpu = false
  drop = ["cpu_time"]
  # Don't collect CPU data for cpu6 & cpu7
  [inputs.cpu.tagdrop]
    cpu = [ "cpu6", "cpu7" ]

[[inputs.disk]]
  [inputs.disk.tagpass]
    # tagpass conditions are OR, not AND.
    # If the (filesystem is ext4 or xfs) OR (the path is /opt or /home)
    # then the metric passes
    fstype = [ "ext4", "xfs" ]
    # Globs can also be used on the tag values
    path = [ "/opt", "/home*" ]
```

### Plugin Config: pass and drop

```toml
# Drop all metrics for guest & steal CPU usage
[[inputs.cpu]]
  percpu = false
  totalcpu = true
  drop = ["usage_guest", "usage_steal"]

# Only store inode related metrics for disks
[[inputs.disk]]
  pass = ["inodes*"]
```

### Plugin config: prefix, suffix, and override

This plugin will emit measurements with the name `cpu_total`

```toml
[[inputs.cpu]]
  name_suffix = "_total"
  percpu = false
  totalcpu = true
```

This will emit measurements with the name `foobar`

```toml
[[inputs.cpu]]
  name_override = "foobar"
  percpu = false
  totalcpu = true
```

### Plugin config: tags

This plugin will emit measurements with two additional tags: `tag1=foo` and
`tag2=bar`

```toml
[[inputs.cpu]]
  percpu = false
  totalcpu = true
  [inputs.cpu.tags]
    tag1 = "foo"
    tag2 = "bar"
```

### Multiple plugins of the same type

Additional plugins (or outputs) of the same type can be specified,
just define more instances in the config file:

```toml
[[inputs.cpu]]
  percpu = false
  totalcpu = true

[[inputs.cpu]]
  percpu = true
  totalcpu = false
  drop = ["cpu_time*"]
```

## Output Configuration

Telegraf also supports specifying multiple output sinks to send data to,
configuring each output sink is different, but examples can be
found by running `telegraf -sample-config`.

Outputs also support the same configurable options as plugins
(pass, drop, tagpass, tagdrop), added in 0.2.4

```toml
[[outputs.influxdb]]
  urls = [ "http://localhost:8086" ]
  database = "telegraf"
  precision = "s"
  # Drop all measurements that start with "aerospike"
  drop = ["aerospike*"]

[[outputs.influxdb]]
  urls = [ "http://localhost:8086" ]
  database = "telegraf-aerospike-data"
  precision = "s"
  # Only accept aerospike data:
  pass = ["aerospike*"]

[[outputs.influxdb]]
  urls = [ "http://localhost:8086" ]
  database = "telegraf-cpu0-data"
  precision = "s"
  # Only store measurements where the tag "cpu" matches the value "cpu0"
  [outputs.influxdb.tagpass]
    cpu = ["cpu0"]
```