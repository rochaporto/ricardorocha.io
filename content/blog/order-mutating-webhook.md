+++
author = "Ricardo Rocha"
date = "2021-02-18T00:00:00+00:00"
description = ""
tags = ["kubernetes", "webhook"]
title = "Putting (Some) Order on Mutating Admission Webhooks"
+++

Use Kubernetes long enough and you will likely end up looking into
[Admision Controller Webhooks](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/#example-writing-and-deploying-an-admission-controller-webhook), a neat way to plug external plugins into the Kubernetes
API lifecycle to complement the built-in controllers that ship by default.

They can be *Mutating* or *Validating*: the first offer a way
to modify objects sent to the API server; the second a way to reject requests and
enforce custom policies. They can be extremely valuable.

### An Example 

This post focus on *Mutating Webhooks*, triggered by a recent issue while using
the [GitLab Runner Kubernetes Executor](https://docs.gitlab.com/runner/executors/kubernetes.html). This runner is a great way to run and scale CI jobs using a Kubernetes cluster,
but for some reason does not expose a generic option to pass *resources (limits
and requests)* to the job/pod definition - it instead exposes explicit fields
for *cpu*, *memory* and *storage*. No obvious way to ask for a *nvidia.com/gpu*
or *cloud-tpus.google.com/v3*.

There is of course an upstream [feature
request](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3464) but in the
meantime people looked into workarounds. Some options you might consider to
customize objects at creation time:
* A [PodPreset](https://v1-18.docs.kubernetes.io/docs/tasks/inject-data-application/podpreset/) or a [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) could do the job, but the first [is gone](https://github.com/kubernetes/kubernetes/issues/93900) starting Kubernetes 1.20 and the second applies these limits to all containers - the
executor launches pods with two containers (helper and builder) and assigning
GPUs this way would double the resource needs for no gain
* A Mutating Webhook which by now you guessed does the job

### Mutating Admission Webhooks

Admission Webhooks are called as part of the control plane, serially by the *MutatingAdmissionWebhook* controller. The configuration of the webhook is done with a bit of yaml specifying an endpoint to be called and some rules on which operations should trigger them
and any filtering for objects where the webhook should be applied:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: gitlab-resources-webhook.gitlab.com
webhooks:
  - name: gitlab-resources-webhook.gitlab.com
    clientConfig:
      service:
        name: gitlab-resources-webhook
        namespace: gitlab
        path: "/mutate"
    rules:
      - operations: [ "CREATE" ]
        apiGroups: ["apps", ""]
        apiVersions: ["v1"]
        resources: ["pods"]
        scope: "Namespaced"
    objectSelector:
      matchLabels:
        gitlab-runner-resources-webhook: "true"
    admissionReviewVersions: ["v1", "v1beta1"]
    sideEffects: None
    timeoutSeconds: 10
```

In the case above we call the webhook for *CREATE* operations on *Pod*
resources which have a label *gitlab-runner-resources-webhook: true*. We also
pass the endpoint to be called matching a service and a path in our cluster.
Behind 


### Putting Order in Webhooks

{{< figure src="/images/blog/gitlab-mutating-webhook.png"
    caption="" width="100%" >}}

