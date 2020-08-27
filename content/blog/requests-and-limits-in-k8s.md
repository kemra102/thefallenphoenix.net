---
author: "Danny Roberts"
date: 2020-08-27
linktitle: Requests & Limits in K8s
title: Requests & Limits in K8s
highlight: true
---

# Requests & Limits in K8s

When deploying apps to Kubernetes (K8s) one thing to get right in your deployments are the requests & limits for both CPU & Memory (RAM). Understanding the differences between requests & limits, the consequences of the values you set etc are all very important things for you to understand in the K8s world.

Throughout this blog post we will use the following example app as a reference point (this isn't valid K8s yaml - just pseudo-code):

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 2
    Memory: 1Gi
```

> CPU resources are defined above using full cores as they have no suffix. Fractions can be achieved using Millicores (thousandths of a core). For example `200m` would be one fifth of a core. You can be granular down to a single millicore (`1m`) if desired.

> Memory resources as you can see are defined using "binary prefixes" instead of "decimal prefixes" (based on multiples of 1024 instead of 1000).

Let's start with some simple questions.

## What are Requests?

Requests are the guaranteed amount of resources that K8s will ensure are available for your app. So when `DemoApp1` requests 1 CPU & 512Mi, it will be scheduled by K8s to a node that has enough capacity available to meet these requests.

## What are Limits?

Limits describe the total amount of resources your app can burst up to. In our case `DemoApp1` can burst up to twice it's request values if needed. There is a very important thing to note here: K8s does **NOT** guarantee that the limit values will be available to an app on the node that it has been scheduled to.

\
No doubt all this raises several questions, so lets try to answer them.

## What happens if?

**Q.** What happens if no node has enough resources to meet my app's requests when I deploy it?\
**A.** Assuming you are running the [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (you are right?) it will spin up a new node (or multiple nodes if required) to meet the demand.

**Q.** What happens if no node has enough resources to meet my app's limits when I deploy it?\
**A.** Limits are not evaluated when K8s is looking to deploy your app to a node. K8s will only evaluate your request values.

**Q.** So what happens if my app needs to burst "into" it's limit values?\
**A1.** One of two things will happen. If there are enough resources available on the node that your app is deployed to then your app will be able to burst up to those limits as expected. It's interesting to note here that these extra resources may be "taken" from other app's request resources that they are not currently using.\
**A2.** If there are no available resources then your app will be "cut off" at that point. If some but not all the limit resources available your app can make use of them of course.

**Q.** Does anything special happen if the limit resources are not available for my app to burst into?\
**A.** For CPU: no. Your app is simply throttled at the request value or the amount it was able to burst up to. For memory however, K8s will terminate the pod your app is running in with an `OOMKilled` message.

\
In the answers above we got a couple of interesting little tidbits about how limits work. Lets explore that because it's useful to understand the mechanics at work here.

## How Limits Work

First of all as mentioned briefly above, when K8s schedules your app to a node, limits are ignored. It is scheduled based on it's request values only. So it is perfectly possible to be deployed to a node that can meet the app's request values but would never be able to meet it's limit values.

This does raise the question though: if limit values are not evaluated at deploy time, then where does K8s get the resources to allow apps to burst at all? Again, this is alluded to above. When an app runs, K8s dynamically updates the CPU/Memory resources assigned to it in order to meet it's current requirements. This means that even though `DemoApp1` is requesting 1 CPU for example, if it is currently idling at `150m` then that is all K8s will assign to that app. This means that when your app isn't using all of the resources that it requested, then another app deployed to the same node may use these currently unused resources to burst into, up to it's own limit values.

What is more interesting here is that as we know K8s guarantees that it can meet your app's request values. So when `DemoApp1` eventually gets off it's backside and does some work and starts using those additional CPU resources, what happens (assuming the node is not under subscribed of course)? Well K8s dynamically updates the resources assigned to `DemoApp1` to meet it's current needs. It does this by taking back the resources that the other app is currently using to burst. Obviously if the other app still needs those burst resources it may then encounter an error as previously discussed.

Some may assume that hitting a limit would trigger the `cluster-autoscaler` but it does **NOT**. The `cluster-autoscaler` only evaluates required resources at deployment time. This is a key reason why getting your requests & limits correct is so crucial.

This image from AWS (see original blog post containing this image [here](https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/)) really succinctly demonstrates the requests/limits relationship in a very simple way:

![©️ Amazon Web Services, Inc. or its affiliates. All rights reserved.](img/k8s-requests-limits.png)

## Slack

Not the popular chat platform. Slack is an important concept in K8s, many blog posts refer to it, and it is important to understand for some of the scenarios we'll take a look at next.

Slack space (or sometimes cost) is the amount of "wasted" resources assigned to your apps. Slack is basically the difference between the resources assigned to your app and what it is actually using. There are many tools (assuming you don't have your own custom graphs in something like Grafana) that shows this slack space such as [kube-resource-report](https://github.com/hjacobs/kube-resource-report).

Many blog posts that talk about reducing your K8s costs will talk about slack space. They will also often tell you to get rid of all this slack space without any thought as to the potential consequences of that. Yes that's right, the situation is more nuanced than many people admit to and blindly "getting rid of" all your slack space may actually be a very bad thing. We'll see how sometimes slack space is a good thing in the scenarios we look at.

## Scenarios

Now we will take this knowledge and look at some scenarios we might find ourselves in and compare the benefits & risks of each.

To get the "full picture" make sure you take a look at our values and the key facts for each scenario as these will differ in subtle ways at times.

### Scenario 1

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 2
    Memory: 1Gi
```

The key facts for this scenario:

- Request values reflect average app usage.
- Limit values are double the request values.

Benefits:

- Relatively little slack space.
- With such high limits, we can handle very large spikes.

Risks:

- If you expect to burst as high as double your requests it is more likely your app will be throttled or killed.
- If you expect to have to burst often, your app is more likely to be throttled or killed.

### Scenario 2

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 1.2
    Memory: 768Mi
```

The key facts for this scenario:

- Request values reflect average app usage.
- Limit values are slightly higher than the request values.

Benefits:

- Relatively little slack space.

Risks:

- If you expect to have to burst often, your app is more likely to be throttled or killed.

### Scenario 3

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 1
    Memory: 512Mi
```

The key facts for this scenario:

- Request values reflect average app usage.
- Limit values are set identically to the request values.

Benefits:

- Relatively little slack space.

Risks:

- If you have to burst at all your app will be throttled or killed.

### Scenario 4

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 2
    Memory: 1Gi
```

The key facts for this scenario:

- Average resource usage is far lower than request values.
- Limit values are double the request values.
- Burst usage rarely goes as high as the request values.
- App has never been known to go as high as limit values.

Benefits:

- You are extremely unlikely to ever be throttled or killed due to resource utilisation as a result of spikes.

Risks:

- You have significant slack space.

### Scenario 5

```
DemoApp1:
  Requests:
    CPU: 1
    Memory: 512Mi
  Limits:
    CPU: 1
    Memory: 512Mi
```

The key facts for this scenario:

- Average resource usage is far lower than request values.
- Limit values are set identically to the request values.
- Burst usage rarely goes as high as the request values.
- App has never been known to go as high as limit values.

Benefits:

- You are extremely unlikely to ever be throttled or killed due to resource utilisation as a result of spikes.

Risks:

- You have significant slack space.
- You essentially don't have any limit values (due to them being identical to the request values) so if you ever have an unexpected spike it is highly likely that your app will be throttled or killed.

## Conclusion

As we can see from the scenarios, getting the requests & limits right for your app is as crucial as it is subtle and dare I say, just a little complicated. We saw that slack space, so looked down upon by many can have the benefit of helping prevent situations where your app is throttled or killed at the cost of consuming extra resources (and upping your AWS bill!).

Undoubtedly getting the right values will take a little trial and error. The TL;DR of this for me is:

- Have a basic fundamental knowledge of how requests & limits work.
- Understand your app's usage profile (things like metrics being fed into a solution like Grafana are gold here).
- Evaluate & balance the benefits/risks you are willing to take.
- Test, update values, test, etc.

Of course this doesn't cover other situations that will influence how your app responds to traffic. Your app's deployment is likely to be autoscaled via the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) for example. This increases the number of replicas of your app across the cluster based on certain pre-defined resource thresholds. Obviously this impacts how your app behaves in terms of resource usage and the consequences of bursting into your limit values etc. This is why testing these values is key.

Hopefully though this has been useful food for thought and will enable you to make more informed decisions about your resource settings from now on.
