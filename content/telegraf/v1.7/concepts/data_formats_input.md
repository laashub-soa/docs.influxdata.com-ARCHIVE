---
title: Telegraf input data formats
description: Telegraf, the plugin-driven server agent component of the InfluxData time series platform, supports parsing input data formats into metrics for InfluxDB Line Protocol, JSON, Graphite, Value, Nagios, Collectd, and Dropwizard.
menu:
  telegraf_1_7:
    name: Input data formats
    weight: 20
    parent: Concepts
---

Telegraf is able to parse the following input data formats into metrics:

1. [InfluxDB Line Protocol](#influxdb-line-protocol)
2. [JSON](#json-data-format)
3. [Graphite](#graphite-data-format)
4. [Value](#value), ie: 45 or "booyah"
5. [Nagios](#nagios-data-format)
6. [Collectd](#collectd-data-format)
7. [Dropwizard](#dropwizard-data-format)

Telegraf metrics, like InfluxDB
[points](/{{< latest "influxdb" >}}/write_protocols/line_protocol_tutorial/),
are a combination of four basic parts:

1. Measurement name
2. Tags
3. Fields
4. Timestamp

These four parts are easily defined when using the [InfluxDB Line Protocol](/{{< latest "influxdb" >}}/write_protocols/line_protocol_reference/) as a data format.
Other data formats may require more advanced configuration to create usable Telegraf metrics.

Plugins such as the Exec (`exec`) input plugin and the Kafka Consumer (`kafka_consumer`) input plugin parse textual data. Up until now,
these plugins were statically configured to parse just a single
data format. The Exec (`exec`) input plugin mostly only supported parsing JSON, and the Kafka Consumer (`kafka_consumer`) only
supported data in InfluxDB line protocol.

Now, we are normalizing the parsing of various data formats across all
plugins that can support it. You will be able to identify a plugin that supports
different data formats by the presence of a `data_format` config option, for
example, in the Exec (`exec`) input plugin:

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/tmp/test.sh", "/usr/bin/mycollector --foo=bar"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "json"

  ## Additional configuration options go here
```

Each `data_format` has an additional set of configuration options available, which
are discussed below.

# InfluxDB Line Protocol

There are no additional configuration options for InfluxDB line protocol. The
metrics are parsed directly into Telegraf metrics.

#### Influx configuration

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/tmp/test.sh", "/usr/bin/mycollector --foo=bar"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "influx"
```

# JSON data format

The JSON data format flattens JSON into metric _fields_.
NOTE: Only numerical values are converted to fields, and they are converted
into a float. Strings are ignored unless specified as a tag_key (see below).

So for example, this JSON:

```json
{
    "a": 5,
    "b": {
        "c": 6
    },
    "ignored": "I'm a string"
}
```

Would get translated into _fields_ of a measurement:

```
myjsonmetric a=5,b_c=6
```

The _measurement name_ is usually the name of the plugin,
but can be overridden using the `name_override` configuration option.

#### JSON configuration

The JSON data format supports specifying "tag keys". If specified, keys
will be searched for in the root-level of the JSON blob. If the key(s) exist,
they will be applied as tags to the Telegraf metrics.

For example, if you had this configuration:

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/tmp/test.sh", "/usr/bin/mycollector --foo=bar"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "json"

  ## List of tag names to extract from top-level of JSON server response
  tag_keys = [
    "my_tag_1",
    "my_tag_2"
  ]
```

with this JSON output from a command:

```json
{
    "a": 5,
    "b": {
        "c": 6
    },
    "my_tag_1": "foo"
}
```

Your Telegraf metrics would get tagged with `my_tag_1`

```
exec_mycollector,my_tag_1=foo a=5,b_c=6
```

If the JSON data is an array, then each element of the array is parsed with the configured settings.
Each resulting metric will be output with the same timestamp.

For example, if the following configuration:

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/usr/bin/mycollector --foo=bar"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "json"

  ## List of tag names to extract from top-level of JSON server response
  tag_keys = [
    "my_tag_1",
    "my_tag_2"
  ]
```

with this JSON output from a command:

```json
[
    {
        "a": 5,
        "b": {
            "c": 6
        },
        "my_tag_1": "foo",
        "my_tag_2": "baz"
    },
    {
        "a": 7,
        "b": {
            "c": 8
        },
        "my_tag_1": "bar",
        "my_tag_2": "baz"
    }
]
```

Your Telegraf metrics would get tagged with `my_tag_1` and `my_tag_2`.

```
exec_mycollector,my_tag_1=foo,my_tag_2=baz a=5,b_c=6
exec_mycollector,my_tag_1=bar,my_tag_2=baz a=7,b_c=8
```

# Value

The "value" data format translates single values into Telegraf metrics. This
is done by assigning a measurement name and setting a single field ("value")
as the parsed metric.

#### Value configuration

You **must** tell Telegraf what type of metric to collect by using the
`data_type` configuration option. Available options are:

1. integer
2. float or long
3. string
4. boolean

The default measurement name is the name of the plugin. You can use the `name_override` option to rename the metric.

**Example: Renaming the measurement name using `name_override`**

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["cat /proc/sys/kernel/random/entropy_avail"]

  ## Override the default measurement name of "exec"
  name_override = "entropy_available"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "value"
  data_type = "integer" # required
```

# Graphite data format

The Graphite data format translates Graphite _dot_ buckets directly into
Telegraf measurement names, with a single value field, and without any tags.
By default, the separator is left as `.`, but this can be changed using the
`separator` argument.
For more advanced options, Telegraf supports specifying "templates" to translate
Graphite buckets into Telegraf metrics.

Templates are of the form:

```
"host.mytag.mytag.measurement.measurement.field*"
```

Where the following keywords exist:

1. `measurement`: specifies that this section of the graphite bucket corresponds
to the measurement name. This can be specified multiple times.
2. `field`: specifies that this section of the graphite bucket corresponds
to the field name. This can be specified multiple times.
3. `measurement*`: specifies that all remaining elements of the graphite bucket
correspond to the measurement name.
4. `field*`: specifies that all remaining elements of the graphite bucket
correspond to the field name.

Any part of the template that is not a keyword is treated as a tag key. This
can also be specified multiple times.

NOTE: `field*` cannot be used in conjunction with `measurement*`!

#### Measurement and tag templates

The most basic template is to specify a single transformation to apply to all
incoming metrics. So the following template:

```toml
templates = [
    "region.region.measurement*"
]
```

would result in the following Graphite -> Telegraf transformation.

```
us.west.cpu.load 100
=> cpu.load,region=us.west value=100
```

Multiple templates can also be specified, but these should be differentiated
using _filters_ (see below for more details)

```toml
templates = [
    "*.*.* region.region.measurement", # <- all 3-part measurements will match this one.
    "*.*.*.* region.region.host.measurement", # <- all 4-part measurements will match this one.
]
```

#### Field templates

The field keyword tells Telegraf to give the metric that field name.
So the following template:

```toml
separator = "_"
templates = [
    "measurement.measurement.field.field.region"
]
```

would result in the following Graphite to Telegraf transformation.

```
cpu.usage.idle.percent.eu-east 100
=> cpu_usage,region=eu-east idle_percent=100
```

The field key can also be derived from all remaining elements of the graphite
bucket by specifying `field*`:

```toml
separator = "_"
templates = [
    "measurement.measurement.region.field*"
]
```

which would result in the following Graphite to Telegraf transformation.

```
cpu.usage.eu-east.idle.percentage 100
=> cpu_usage,region=eu-east idle_percentage=100
```

#### Filter templates

Users can also filter the template(s) to use based on the name of the bucket,
using _glob matching_, like so:

```toml
templates = [
    "cpu.* measurement.measurement.region",
    "mem.* measurement.measurement.host"
]
```

which would result in the following transformation:

```
cpu.load.eu-east 100
=> cpu_load,region=eu-east value=100

mem.cached.localhost 256
=> mem_cached,host=localhost value=256
```

#### Adding tags

Additional tags can be added to a metric that don't exist on the received metric.
You can add additional tags by specifying them after the pattern.
Tags have the same format as the line protocol.
Multiple tags are separated by commas.

```toml
templates = [
    "measurement.measurement.field.region datacenter=1a"
]
```

would result in the following Graphite to Telegraf transformation.

```
cpu.usage.idle.eu-east 100
=> cpu_usage,region=eu-east,datacenter=1a idle=100
```

Many more [template options](https://github.com/influxdata/influxdb/tree/master/services/graphite#templates) are available.

#### Graphite configuration

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/tmp/test.sh", "/usr/bin/mycollector --foo=bar"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "graphite"

  ## This string will be used to join the matched values.
  separator = "_"

  ## Each template line requires a template pattern. It can have an optional
  ## filter before the template and separated by spaces. It can also have optional extra
  ## tags following the template. Multiple tags should be separated by commas and no spaces
  ## similar to the line protocol format. There can be only one default template.
  ## Templates support below format:
  ## 1. filter + template
  ## 2. filter + template + extra tag(s)
  ## 3. filter + template with field key
  ## 4. default template
  templates = [
    "*.app env.service.resource.measurement",
    "stats.* .host.measurement* region=eu-east,agent=sensu",
    "stats2.* .host.measurement.field",
    "measurement*"
  ]
```

# Nagios data format

There are no additional configuration options for Nagios line protocol. The
metrics are parsed directly into Telegraf metrics.

Note: Nagios input data formats are only supported in the [Exec (`exec`) input plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/exec).

#### Nagios configuration

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["/usr/lib/nagios/plugins/check_load -w 5,6,7 -c 7,8,9"]

  ## measurement name suffix (for separating different commands)
  name_suffix = "_mycollector"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "nagios"
```

# Collectd data format

The collectd data format parses the collectd binary network protocol.  Tags are
created for host, instance, type, and type instance.  All collectd values are
added as float64 fields.

For more information, see
[Binary protocol](https://collectd.org/wiki/index.php/Binary_protocol) at the collectd Wiki.

You can control the cryptographic settings with parser options.  Create an
authentication file and set `collectd_auth_file` to the path of the file, then
set the desired security level in `collectd_security_level`.

Additional information, including client setup, can be found
in [Cryptographic setup](https://collectd.org/wiki/index.php/Networking_introduction#Cryptographic_setup) section of the collectd Wiki.

You can also change the path to the `typesdb` or add additional `typesdb` using
`collectd_typesdb`.

#### Collectd Configuration:

```toml
[[inputs.socket_listener]]
  service_address = "udp://127.0.0.1:25826"
  name_prefix = "collectd_"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "collectd"

  ## Authentication file for cryptographic security levels
  collectd_auth_file = "/etc/collectd/auth_file"
  ## One of none (default), sign, or encrypt
  collectd_security_level = "encrypt"
  ## Path of to TypesDB specifications
  collectd_typesdb = ["/usr/share/collectd/types.db"]
```

# Dropwizard data format

The Dropwizard format can parse the JSON representation of a single Dropwizard metric registry. By default, tags are parsed from metric names as if they were actual influxdb line protocol keys (`measurement<,tag_set>`) which can be overridden by defining custom [measurement & tag templates](#measurement-and-tag-templates). All field value types are supported, `string`, `number` and `boolean`.

A typical JSON of a Dropwizard metric registry:

```json
{
	"version": "3.0.0",
	"counters" : {
		"measurement,tag1=green" : {
			"count" : 1
		}
	},
	"meters" : {
		"measurement" : {
			"count" : 1,
			"m15_rate" : 1.0,
			"m1_rate" : 1.0,
			"m5_rate" : 1.0,
			"mean_rate" : 1.0,
			"units" : "events/second"
		}
	},
	"gauges" : {
		"measurement" : {
			"value" : 1
		}
	},
	"histograms" : {
		"measurement" : {
			"count" : 1,
			"max" : 1.0,
			"mean" : 1.0,
			"min" : 1.0,
			"p50" : 1.0,
			"p75" : 1.0,
			"p95" : 1.0,
			"p98" : 1.0,
			"p99" : 1.0,
			"p999" : 1.0,
			"stddev" : 1.0
		}
	},
	"timers" : {
		"measurement" : {
			"count" : 1,
			"max" : 1.0,
			"mean" : 1.0,
			"min" : 1.0,
			"p50" : 1.0,
			"p75" : 1.0,
			"p95" : 1.0,
			"p98" : 1.0,
			"p99" : 1.0,
			"p999" : 1.0,
			"stddev" : 1.0,
			"m15_rate" : 1.0,
			"m1_rate" : 1.0,
			"m5_rate" : 1.0,
			"mean_rate" : 1.0,
			"duration_units" : "seconds",
			"rate_units" : "calls/second"
		}
	}
}
```

Would get translated into four different measurements:

```
measurement,metric_type=counter,tag1=green count=1
measurement,metric_type=meter count=1,m15_rate=1.0,m1_rate=1.0,m5_rate=1.0,mean_rate=1.0
measurement,metric_type=gauge value=1
measurement,metric_type=histogram count=1,max=1.0,mean=1.0,min=1.0,p50=1.0,p75=1.0,p95=1.0,p98=1.0,p99=1.0,p999=1.0
measurement,metric_type=timer count=1,max=1.0,mean=1.0,min=1.0,p50=1.0,p75=1.0,p95=1.0,p98=1.0,p99=1.0,p999=1.0,stddev=1.0,m15_rate=1.0,m1_rate=1.0,m5_rate=1.0,mean_rate=1.0
```

You may also parse a Dropwizard registry from any JSON document which contains a Dropwizard registry in some inner field.
For example, to parse the following JSON document:

```json
{
	"time" : "2017-02-22T14:33:03.662+02:00",
	"tags" : {
		"tag1" : "green",
		"tag2" : "yellow"
	},
	"metrics" : {
		"counters" : 	{
			"measurement" : {
				"count" : 1
			}
		},
		"meters" : {},
		"gauges" : {},
		"histograms" : {},
		"timers" : {}
	}
}
```
and translate it into:

```
measurement,metric_type=counter,tag1=green,tag2=yellow count=1 1487766783662000000
```

you simply need to use the following additional configuration properties:

```toml
dropwizard_metric_registry_path = "metrics"
dropwizard_time_path = "time"
dropwizard_time_format = "2006-01-02T15:04:05Z07:00"
dropwizard_tags_path = "tags"
## tag paths per tag are supported too, eg.
#[inputs.yourinput.dropwizard_tag_paths]
#  tag1 = "tags.tag1"
#  tag2 = "tags.tag2"
```


For more information on the Dropwizard JSON format, see
[JSON Support](http://metrics.dropwizard.io/3.1.0/manual/json/) in the Dropwizard documentation.

#### Dropwizard configuration

```toml
[[inputs.exec]]
  ## Commands array
  commands = ["curl http://localhost:8080/sys/metrics"]
  timeout = "5s"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "dropwizard"

  ## Used by the templating engine to join matched values when cardinality is > 1
  separator = "_"

  ## Each template line requires a template pattern. It can have an optional
  ## filter before the template and separated by spaces. It can also have optional extra
  ## tags following the template. Multiple tags should be separated by commas and no spaces
  ## similar to the line protocol format. There can be only one default template.
  ## Templates support below format:
  ## 1. filter + template
  ## 2. filter + template + extra tag(s)
  ## 3. filter + template with field key
  ## 4. default template
  ## By providing an empty template array, templating is disabled and measurements are parsed as influxdb line protocol keys (measurement<,tag_set>)
  templates = []

  ## You may use an appropriate [gjson path](https://github.com/tidwall/gjson#path-syntax)
  ## to locate the metric registry within the JSON document
  # dropwizard_metric_registry_path = "metrics"

  ## You may use an appropriate [gjson path](https://github.com/tidwall/gjson#path-syntax)
  ## to locate the default time of the measurements within the JSON document
  # dropwizard_time_path = "time"
  # dropwizard_time_format = "2006-01-02T15:04:05Z07:00"

  ## You may use an appropriate [gjson path](https://github.com/tidwall/gjson#path-syntax)
  ## to locate the tags map within the JSON document
  # dropwizard_tags_path = "tags"

  ## You may even use tag paths per tag
  # [inputs.exec.dropwizard_tag_paths]
  #   tag1 = "tags.tag1"
  #   tag2 = "tags.tag2"

```
