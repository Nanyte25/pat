```
 ____   _  _____ 
|  _ \ / \|_   _|
| |_) / _ \ | |  
|  __/ ___ \| |  
|_| /_/   \_\_|  
                 
```

Prometheus Alert Testing tool

[![CircleCI](https://circleci.com/gh/kevinjqiu/pat.svg?style=svg)](https://circleci.com/gh/kevinjqiu/pat)


You may also be interested in [PromCLI](https://github.com/kevinjqiu/promcli)

Build & Install
===============

    go get github.com/kevinjqiu/pat

You must have golang 1.9+ and [`dep`](https://github.com/golang/dep) installed.

Build from source
-----------------

Check out this repo to $GOPATH/src/github.com/kevinjqiu/pat

and then:

    cd $GOPATH/src/github.com/kevinjqiu/pat && make build

Usage
=====

    pat [options] <test_yaml_file_glob>

e.g.,

    pat test/*.yaml

Test File Format
================

Test files are written in yaml format. For a complete schema definition (in jsonschema format), see [here](https://github.com/kevinjqiu/pat/blob/master/pkg/schema/schema.yaml).

Top level attributes
--------------------

* `name` - The name of the test case
* [`rules`](#rules) - The rule definitions that are under test
* [`fixtures`](#fixtures) - The fixture setup for the tests
* [`assertions`](#assertions) - The test assertions

Rules
-----

The `rules` section defines how the rules-under-test should be loaded.
Currently, two rules loading strategies are supported:

* fromFile - load the rules from a .rules yaml file. If the path specified is not an absolute path, the rule file path will be relative to the test file.
* fromLiteral - embed the rules under test right inside the test file.

### Example

```yaml
rules:
  fromFile: http-rules.yaml
```

or

```yaml
rules:
  fromLiteral: |-
    groups:
      - name: prometheus.rules
        rules:
          - alert: HTTPRequestRateLow
            expr: http_requests{group="canary", job="app-server"} < 100
            for: 1m
            labels:
              severity: critical
```

Fixtures
--------

The `fixtures` section defines a list of metrics fixtures that the tests will be using.
Each item in the list has the following attributes:

* `duration` - How long these metrics will be set to the specified value. The duration must be acceptable by Golang's [`time.ParseDuration()`](https://golang.org/pkg/time/#ParseDuration), e.g., `5m` (5 minutes), `1h` (1 hour), etc.
* `metrics` - The metrics and their values

### Example

```yaml
fixtures:
  5m:
    - http_requests{job="app-server", instance="0", group="blue"}	75
    - http_requests{job="app-server", instance="1", group="blue"}	120
```

This will create these two metrics, with the values last for 5 minutes.

You are also able to specify multiple metrics values:

```yaml
  5m:
    - http_requests{job="app-server", instance="0", group="blue"}	75 100 200
```

In this case, the metric `http_requests{job="app-server", instance="0", group="blue"}` will be set to `75` for the first 5 minutes, `100` for the next 5 minutes and `200` for the next 5 minutes. You can use this form to easily setup long running time series.

Assertions
----------

The `assertions` section contains a list of expectations when the alert rules are evaluated at certain time.

* `at` - The instant when the rules are being evaluated
* `expected` - The list of expected alert properties

### Example

```yaml
assertions:
  - at: 0m
    expected:
      - alertname: HTTPRequestRateLow
        alertstate: pending
        job: app-server
        severity: critical
  - at: 5m
    expected:
      - alertname: HTTPRequestRateLow
        alertstate: firing
        job: app-server
        severity: critical
  - at: 10m
    expected: []
```

In this example, we're asserting that when the alert rules are evaluated at `0m`, with the given fixtures, we should get `HTTPRequestRateLow` alert in `pending` state, and when evaluated at `5m`, the alert should be in `firing` state. When evaluated at `10m`, we shouldn't get any alert.

A Complete Example
==================

Suppose you have the following rule file that you want to be tested:

```yaml
groups:
  - name: prometheus.rules
    rules:
      - alert: HTTPRequestRateLow
        expr: http_requests{group="canary", job="app-server"} < 100
        for: 1m
        labels:
          severity: critical
```

Write a yaml file with your test cases:

```yaml
name: Test HTTP Requests too low alert
rules:
  fromFile: rules.yaml
fixtures:
  - duration: 5m
    metrics:
      - http_requests{job="app-server", instance="0", group="canary", severity="overwrite-me"}	75 85  95 105 105  95  85
      - http_requests{job="app-server", instance="1", group="canary", severity="overwrite-me"}	80 90 100 110 120 130 140
assertions:
  - at: 0m
    expected:
      - alertname: HTTPRequestRateLow
        alertstate: pending
        group: canary
        instance: "0"
        job: app-server
        severity: critical
      - alertname: HTTPRequestRateLow
        alertstate: pending
        group: canary
        instance: "1"
        job: app-server
        severity: critical
    comment: |-
      At 0m, the alerts met the threshold but has not met the duration requirement. Expect the alert to be pending
  - at: 5m
    expected:
      - alertname: HTTPRequestRateLow
        alertstate: firing
        group: canary
        instance: "0"
        job: app-server
        severity: critical
      - alertname: HTTPRequestRateLow
        alertstate: firing
        group: canary
        instance: "1"
        job: app-server
        severity: critical
    comment: |-
      At 5m, the alerts should be firing because the duration requirement is met.
  - at: 10m
    expected:
      - alertname: HTTPRequestRateLow
        alertstate: firing
        group: canary
        instance: "0"
        job: app-server
        severity: critical
    comment: |-
      At 10m, the alert should be firing only for instance 0 because instance 1 is >= 100.
  - at: 15m
    expected: []
    comment: |-
      At 15m, both instances are back to normal, therefore we expect no alert.
```

Run the test:

```bash
$ ./pat examples/test.yaml
=== RUN   Test_HTTP_Requests_too_low_alert_at_0m
--- PASS: Test_HTTP_Requests_too_low_alert_at_0m (0.00s)
=== RUN   Test_HTTP_Requests_too_low_alert_at_5m
--- PASS: Test_HTTP_Requests_too_low_alert_at_5m (0.00s)
PASS
```
