# go_expvar

The `go_expvar` module can monitor any Go application that exposes its metrics with the use of
`expvar` package from the Go standard library.

`go_expvar` produces charts for Go runtime memory statistics and optionally any number of custom charts.

For the memory statistics, it produces the following charts:

1. **Heap allocations** in kB
 * alloc: size of objects allocated on the heap
 * inuse: size of allocated heap spans

2. **Stack allocations** in kB
 * inuse: size of allocated stack spans

3. **MSpan allocations** in kB
 * inuse: size of allocated mspan structures

4. **MCache allocations** in kB
 * inuse: size of allocated mcache structures

5. **Virtual memory** in kB
 * sys: size of reserved virtual address space

6. **Live objects**
 * live: number of live objects in memory

7. **GC pauses average** in ns
 * avg: average duration of all GC stop-the-world pauses


## Monitoring Go Applications

Netdata can be used to monitor running Go applications that expose their metrics with
the use of the [expvar package](https://golang.org/pkg/expvar/) included in Go standard library.

The `expvar` package exposes these metrics over HTTP and is very easy to use.
Consider this minimal sample below:

```go
package main

import (
        _ "expvar"
        "net/http"
)

func main() {
        http.ListenAndServe("127.0.0.1:8080", nil)
}
```

When imported this way, the `expvar` package registers a HTTP handler at `/debug/vars` that
exposes Go runtime's memory statistics in JSON format. You can inspect the output by opening
the URL in your browser (or by using `wget` or `curl`).

Sample output:

```json
{
"cmdline": ["./expvar-demo-binary"],
"memstats": {"Alloc":630856,"TotalAlloc":630856,"Sys":3346432,"Lookups":27, <ommited for brevity>}
}
```

You can of course expose and monitor your own variables as well.
Here is a sample Go application that exposes a few custom variables:

```go
package main

import (
    "expvar"
    "net/http"
    "runtime"
    "time"
)

func main() {

    tick := time.NewTicker(1 * time.Second)
    num_go := expvar.NewInt("runtime.goroutines")
    counters := expvar.NewMap("counters")
    counters.Set("cnt1", new(expvar.Int))
    counters.Set("cnt2", new(expvar.Float))

    go http.ListenAndServe(":8080", nil)

    for {
        select {
        case <- tick.C:
            num_go.Set(int64(runtime.NumGoroutine()))
            counters.Add("cnt1", 1)
            counters.AddFloat("cnt2", 1.452)
        }
    }
}
```

Apart from the runtime memory stats, this application publishes two counters and the
number of currently running Goroutines and updates these stats every second.

In the next section, we will cover how to monitor and chart these exposed stats with
the use of `netdata`s ```go_expvar``` module.

### Using netdata go_expvar module

The `go_expvar` module is disabled by default. To enable it, edit [`python.d.conf`](https://github.com/netdata/netdata/tree/master/collectors/python.d.plugin/python.d.conf)
(to edit it on your system run `/etc/netdata/edit-config python.d.conf`), and change the `go_expvar`
variable to `yes`:

```
# Enable / Disable python.d.plugin modules
#default_run: yes
#
# If "default_run" = "yes" the default for all modules is enabled (yes).
# Setting any of these to "no" will disable it.
# 
# If "default_run" = "no" the default for all modules is disabled (no).
# Setting any of these to "yes" will enable it.
...
go_expvar: yes
...
```

Next, we need to edit the module configuration file (found at [`/etc/netdata/python.d/go_expvar.conf`](https://github.com/netdata/netdata/tree/master/collectors/python.d.plugin/go_expvar/go_expvar.conf) by default)
(to edit it on your system run `/etc/netdata/edit-config python.d/go_expvar.conf`).
The module configuration consists of jobs, where each job can be used to monitor a separate Go application.
Let's see a sample job configuration:

```
# /etc/netdata/python.d/go_expvar.conf

app1:
  name : 'app1'
  url  : 'http://127.0.0.1:8080/debug/vars'
  collect_memstats: true
  extra_charts: {}
```

Let's go over each of the defined options:

    name: 'app1'

This is the job name that will appear at the netdata dashboard.
If not defined, the job_name (top level key) will be used.

    url: 'http://127.0.0.1:8080/debug/vars'

This is the URL of the expvar endpoint. As the expvar handler can be installed
in a custom path, the whole URL has to be specified. This value is mandatory.

    collect_memstats: true

Whether to enable collecting stats about Go runtime's memory. You can find more
information about the exposed values at the [runtime package docs](https://golang.org/pkg/runtime/#MemStats).

    extra_charts: {}

Enables the user to specify custom expvars to monitor and chart.
Will be explained in more detail below.

**Note: if `collect_memstats` is disabled and no `extra_charts` are defined, the plugin will
disable itself, as there will be no data to collect!**

Apart from these options, each job supports options inherited from netdata's `python.d.plugin`
and its base `UrlService` class. These are:

    update_every: 1          # the job's data collection frequency
    priority:     60000      # the job's order on the dashboard
    retries:      60         # the job's number of restoration attempts
    user:         admin      # use when the expvar endpoint is protected by HTTP Basic Auth
    password:     sekret     # use when the expvar endpoint is protected by HTTP Basic Auth

### Monitoring custom vars with go_expvar

Now, memory stats might be useful, but what if you want netdata to monitor some custom values
that your Go application exposes? The `go_expvar` module can do that as well with the use of
the `extra_charts` configuration variable.

The `extra_charts` variable is a YaML list of netdata chart definitions.
Each chart definition has the following keys:

    id:         netdata chart ID
    options:    a key-value mapping of chart options
    lines:      a list of line definitions

**Note: please do not use dots in the chart or line ID field.
See [this issue](https://github.com/netdata/netdata/pull/1902#issuecomment-284494195) for explanation.**

Please see these two links to the official netdata documentation for more information about the values:

- [External plugins - charts](../../plugins.d/#chart)
- [Chart variables](https://github.com/netdata/netdata/wiki/How-to-write-new-module#global-variables-order-and-chart)

**Line definitions**

Each chart can define multiple lines (dimensions).
A line definition is a key-value mapping of line options.
Each line can have the following options:
    
    # mandatory
    expvar_key: the name of the expvar as present in the JSON output of /debug/vars endpoint
    expvar_type: value type; supported are "float" or "int"
    id: the id of this line/dimension in netdata

    # optional - netdata defaults are used if these options are not defined
    name: ''
    algorithm: absolute
    multiplier: 1
    divisor: 100 if expvar_type == float, 1 if expvar_type == int
    hidden: False

Please see the following link for more information about the options and their default values:
[External plugins - dimensions](../../plugins.d/#dimension)

Apart from top-level expvars, this plugin can also parse expvars stored in a multi-level map;
All dicts in the resulting JSON document are then flattened to one level.
Expvar names are joined together with '.' when flattening.

Example:
```
{
    "counters": {"cnt1": 1042, "cnt2": 1512.9839999999983},
    "runtime.goroutines": 5
}
```

In the above case, the exported variables will be available under `runtime.goroutines`,
`counters.cnt1` and `counters.cnt2` expvar_keys. If the flattening results in a key collision,
the first defined key wins and all subsequent keys with the same name are ignored.

**Configuration example**

The configuration below matches the second Go application described above.
Netdata will monitor and chart memory stats for the application, as well as a custom chart of
running goroutines and two dummy counters.

```
app1:
  name : 'app1'
  url  : 'http://127.0.0.1:8080/debug/vars'
  collect_memstats: true
  extra_charts:
    - id: "runtime_goroutines"
      options:
        name: num_goroutines
        title: "runtime: number of goroutines"
        units: goroutines
        family: runtime
        context: expvar.runtime.goroutines
        chart_type: line
      lines:
        - {expvar_key: 'runtime.goroutines', expvar_type: int, id: runtime_goroutines}
    - id: "foo_counters"
      options:
        name: counters
        title: "some random counters"
        units: awesomeness
        family: counters
        context: expvar.foo.counters
        chart_type: line
      lines:
        - {expvar_key: 'counters.cnt1', expvar_type: int, id: counters_cnt1}
        - {expvar_key: 'counters.cnt2', expvar_type: float, id: counters_cnt2}
```

**Netdata charts example**

The images below show how do the final charts in netdata look.

![Memory stats charts](https://cloud.githubusercontent.com/assets/15180106/26762052/62b4af58-493b-11e7-9e69-146705acfc2c.png)

![Custom charts](https://cloud.githubusercontent.com/assets/15180106/26762051/62ae915e-493b-11e7-8518-bd25a3886650.png)

