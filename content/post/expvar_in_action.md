+++
date = "2017-06-07T12:02:53+02:00"
draft = false
title = "expvar in action"
description = "Expose application and runtime metrics in Go using the expvar package"
tags = [
    "go",
]
categories = [
    "go",
]
+++

The Go standard library comes with the [expvar][expvar] package. This package
allows one to expose metrics about your application and the Go runtime via a
HTTP API in JSON format. I think the package is useful for everyone writing Go
code. But the data from [godoc.org][godoc] suggests that not many people are
aware of this package. The package has been imported [1761 times][godoc expvar]
by public projects. With [3300 imports][godoc image] even the [image][image]
package has more imports. With this post I want to show how the `expvar`
package works and how it can be useful for others.

## Example
Before exploring the details of this package, I want to demonstrate what you
can do with the `expvar` package. The following snippet creates a HTTP server
listening at port 1818. On each request `handler()` increases a counter before
presenting the visitor with a message.

```go
package main

import (
    "expvar"
    "fmt"
    "net/http"
)


var visits = expvar.NewInt("visits")

func handler(w http.ResponseWriter, r *http.Request) {
    visits.Add(1)
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":1818", nil)
}
```

On import the `expvar` package registers a handler function for the pattern
'/debug/vars' on the `http.DefaultServeMux`. This handler returns all metrics
that have been registered at the `expvar` package. If you run the code and
visit http://localhost:1818/debug/vars you'll see something like below. The
output has been truncated to increase readability:

```json
{
  "cmdline": [
    "/tmp/go-build872151671/command-line-arguments/_obj/exe/main"
  ],
  "memstats": {
    "Alloc": 397576,
    "TotalAlloc": 397576,
    "Sys": 3084288,
    "Lookups": 7,
    "Mallocs": 5119,
    "Frees": 167,
    "HeapAlloc": 397576,
    "HeapSys": 1769472,
    "HeapIdle": 1015808,
    "HeapInuse": 753664,
    "HeapReleased": 0,
    "HeapObjects": 4952,
    "StackInuse": 327680,
    "StackSys": 327680,
    "MSpanInuse": 14240,
    "MSpanSys": 16384,
    "MCacheInuse": 4800,
    "MCacheSys": 16384,
    "BuckHashSys": 2380,
    "GCSys": 131072,
    "OtherSys": 820916,
    "NextGC": 4194304,
    "LastGC": 0,
    "PauseTotalNs": 0,
    "PauseNs": [
      0,
      0,
    ],
    "PauseEnd": [
      0,
      0
    ],
    "GCCPUFraction": 0,
    "EnableGC": true,
    "DebugGC": false,
    "BySize": [
      {
        "Size": 16640,
        "Mallocs": 0,
        "Frees": 0
      },
      {
        "Size": 17664,
        "Mallocs": 0,
        "Frees": 0
      }
    ]
  },
  "visits": 0
}
```

It's a lot of info. That is because the metrics `os.Args` and
`runtime.Memstats` are registered by default. I want to focus at the `visits`
counter at the end of this JSON response.  Because the counter hasn't been
incremented yet it's value is still 0.  Now increment the counter by visiting
http://localhost:1818/java a few times before returning back. The counter
shouldn't be 0 anymore.

## expvar.Publish
The `expvar` package is quite small and easy to understand. It primarily
consists of two components. The first one is the function `expvar.Publish(name
string, v expvar.Var)`. This function can be used to register `v` with a
certain name at a unexported, global registry. The following snippet shows the
implementation. The next 3 code snippets are taken from [the source code of the
expvar package][source].

``` go
// Publish declares a named exported variable. This should be called from a
// package's init function when it creates its Vars. If the name is already
// registered then this will log.Panic.
func Publish(name string, v Var) {
	mutex.Lock()
	defer mutex.Unlock()

        // Check if name has been taken already. If so, panic.
	if _, existing := vars[name]; existing {
		log.Panicln("Reuse of exported var name:", name)
	}

        // vars is the global registry. It is defined somewhere else in the
        // expvar package like this:
        //
        //  vars = make(map[string]Var)
	vars[name] = v
	varKeys = append(varKeys, name)
	sort.Strings(varKeys)
}
```

## expvar.Var
The other important component is the `expvar.Var` interface. This interface has
only one method:

``` go
// Var is an abstract type for all exported variables.
type Var interface {
        // String returns a valid JSON value for the variable.
        // Types with String methods that do not return valid JSON
        // (such as time.Time) must not be used as a Var.
        String() string
}
```

So you can call `Publish()` with every type that has a method `String()`
returning a string.

## expvar.Int
The `expvar` package comes with a few types that adhere the the `expvar.Var`
interface.  One of them is `expvar.Int` and we used it already in our demo code
when we called `expvar.NewInt("visits")`. This creates a new `expvar.Int`,
registers it using `expvar.Publish` before returning a pointer to the newly
created `expvar.Int`.

``` go
func NewInt(name string) *Int {
	v := new(Int)
	Publish(name, v)
	return v
}
```

`expvar.Int` wraps around an `int64` and has two functions, `Add(delta int64)` and
`Set(value int64)`, to modify this integer in a thread safe way.

## Other types
Besides `expvar.Int` the `expvar` offers a few other types that implement the
`expvar.Var` interface:

* [expvar.Float][float]
* [expvar.String][string]
* [expvar.Map][map]
* [expvar.Func][func]

The first two types wrap around `float64` and `string`. The latter two
types require a few words of explanation.

The `expvar.Map` type can be used to make metrics appear under certain name
space. I use it often like this:

``` go
var stats = expvar.NewMap("tcp")
var requests, requestsFailed expvar.Int

func init() {
	stats.Set("requests", &requests)
	stats.Set("requests_failed", &requestsFailed)
}
```

This code registers the metrics `requests` and `requests_failed` using the
name space `tcp`. It'll appear in the JSON response like this:

``` json
{
    "tcp": {
        "request": 18,
        "requests_failed": 21
    }
}
```

When you want to use a result of a function you could use `expvar.Func`. Assume
you want you export the uptime of your application. This value must be
recalculated every time someone visits http://localhost:1818/debug/vars.

``` go
var start = time.Now()

func calculateUptime() interface {
    return  int64(time.Since(start) * time.Second)
}

expvar.Publish("uptime", expvar.Func(calculateUptime))
```

## Conclusion
Exposing application metrics is very easy with `expvar` package. I use it in
almost in every application I write to expose a few metrics indicating the
applications health. Together with a custom aggregrator, [InfluxDB][influxdb]
and [Grafana][grafana] I can monitor my applications very easy.

[expvar]: https://golang.org/pkg/expvar/
[float]: https://golang.org/pkg/expvar/#Float
[godoc]: https://godoc.org
[godoc image]: https://godoc.org/?q=image
[godoc expvar]: https://godoc.org/?q=expvar
[grafana]: https://grafana.com/
[func]: https://golang.org/pkg/expvar/#Func
[influxdb]: https://docs.influxdata.com/influxdb/v1.2/introduction/getting_started/
[image]: https://golang.org/pkg/image/
[map]: https://golang.org/pkg/expvar/#Map
[source]: https://golang.org/src/expvar/expvar.go
[string]: https://golang.org/pkg/expvar/#String
[var]: https://golang.org/pkg/expvar/#Var
