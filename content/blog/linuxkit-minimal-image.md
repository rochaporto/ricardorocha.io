+++
author = "Ricardo Rocha"
date = "2021-01-11T14:00:00+00:00"
description = "A simple and declarative way to maintain minimal and reproducible cluster node images"
tags = ["kubernetes", "docker", "containerd", "linuxkit"]
title = "Building Immutable Cluster Images with LinuxKit"
+++

Probably the most common way 

Tradeoffs (state, local data ... needs to be externalized or properly
replicated)

Containerization improves the situation as nodes do not necessarily need to be
recreated, as long as only the containers should be upgraded.

Levels:
* Node dependencies
* Containers / applications



https://martinfowler.com/bliki/PhoenixServer.html
https://martinfowler.com/bliki/ImmutableServer.html

Immutable vs configuration drift

Immutable usage in other places: rancherOs, CoreOS, Fedora Atomic, Fedora Core,
COS in GCP


While many applications and services already expose internal metrics in
[Prometheus](https://prometheus.io/) format - especially the ones often
deployed on Kubernetes - many others keep this information only inside log files
and require extra tooling to parse and expose the data.

Recently i worked on a distributed setup with Kubernetes clusters running in
multiple clouds, each with a Prometheus instance collecting local metrics which 
then get aggregated centrally. One of the use cases involves evaluating the
performance of batch like workloads, which also tend to keep performance data in the logs
for end user evaluation after the jobs are finished.

Looking for a good option to expose these metrics live in the Prometheus setup
above (including the central aggregation) i found [mtail](https://github.com/google/mtail)
which made my day. It's a simple tool that does the job - i rely on other
solutions for other projects but they tend to be generic enough to require
multiple pieces and complex configurations.

#### An mtail Example

> [mtail](https://github.com/google/mtail) is a tool for extracting metrics from application logs to be exported into a timeseries database or timeseries calculator for alerting and dashboarding

It supports collectd, StatsD and Graphite as well, but in this case i care
about Prometheus. You can define [counters and
gauges](https://github.com/google/mtail/blob/master/docs/Metrics.md) and attach
labels that end up being multiple dimensions on the metrics. 

Here's an example of the output from one of the workloads being submitted.
```bash
$ cat sample.log
running with 1 200 100 2000
platform: gpu
1.8687622547149658
(200, 100, 1, 2000) gpu:0
running with 1 200 100 2000
platform: gpu
1.9642915725708008
(200, 100, 1, 2000) gpu:0
...
```

We can create a gauge with the workload duration (the line with the floating point
value) with a very simple regex.
```bash
$ cat config.mtail
gauge duration

/(?P<dur>[0-9]+[.][0-9]+)/ {
    duration = float($dur)
}
```

And we validate locally before deploying.
```bash
$ mtail -one_shot -logs sample.log -progs config.mtail
Metrics store:{
  "duration": [
    {
      "Name": "duration",
      "Program": "log.mtail",
      "Kind": 2,
      "Type": 1,
      "LabelValues": [
        {
          "Value": {
            "Value": 1.9642915725708008,
            "Time": 1608122868426470655
          }
        }
      ]
    }
  ]
```

In this mode mtail is dumping the last metric parsed, but by default it would
start a HTTP listener on port 3903 that can be scraped periodically.

#### Deployment as a Sidecar

The next step is to deploy this along the workload in our Kubernetes clusters,
with the goal of not having to change the original application. To generate the
metrics mtail needs access to the log file being populated and
this is where the [sidecar
pattern](https://kubernetes.io/docs/concepts/workloads/pods/#using-pods) fits perfectly - not sure who first came
up with the `Pod` concept, but kudos. 

We need a `ConfigMap` to hold the mtail config.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myworkload
data:
  log.mtail: |-
    gauge duration
    
    /(?P<dur>[0-9]+[.][0-9]+)/ {
        duration = float($dur)
    }
```

And in this case we're deploying the workload using a `CronJob`.
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: myworkload
spec:
  schedule: ...
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            workload: myworkload
            cluster: clustername
            cloud: gcp
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "3903"
        spec:
          containers:
          - name: myworkload
            image: rochaporto/myworkload
            command: ["bash", "-c", "python3 /lowlevel.py 2>&1 | tee -a /log/output.log"]
            volumeMounts:
            - name: log
              mountPath: /log
          - name: mtail
            image: rochaporto/mtail
            command: ["bash", "-c", "while [[ ! -f /log/output.log ]]; do sleep 1; done && /bin/mtail -logtostderr -progs /etc/mtail/log.mtail -logs /log/output.log"]
            ports:
            - containerPort: 3903
            volumeMounts:
            - name: log
              mountPath: /log
            - name: mtail
              mountPath: /etc/mtail
          volumes:
          - name: log
            emptyDir: {}
          - name: mtail
            configMap:
              name: myworkload 
```

There are some things that are interesting in this definition.

* The **log volume** is mounted on both containers so that mtail can see the
log file being populated, and the funny while loop before launching mtail
waits that it gets created. Also **tee** is used to also keep the logs in the
main container's stdout - so that `kubectl get logs myworkload -c workload` is
also possible
* The **mtail container** exposes port 3903 which is the default port for
mtail's HTTP listener. Our Prometheus instance has rules to take as scrape
targets all Pods with **prometheus.io/scrape=true**, you might be doing
discovery differently

