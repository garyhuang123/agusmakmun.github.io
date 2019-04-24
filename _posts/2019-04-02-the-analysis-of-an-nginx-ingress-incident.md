---
layout: post
title:  "The Analysis of An Nginx Ingress Incident"
date:   2019-04-02 10:10:03 +0800
categories: [kubernetes]
---
### The incident
All the nginx controllers crash by a bad ingress annotation consisting of $ sign.

### An short conclusion
It is a defect of nginx-controller. When it is at the first time of reloading configuration, say nginx-controller starts up, it fails to reload the healthy rules if there are bad ones (like, with ${variable}) in there. The /etc/nginx/nginx.conf then comes with no upstreams and /healthz location. After a while, Kubernetes probes the pod is unhealthy because no /healthz location is served. So it kills nginx-contoller and tries to restart it. But that repeats because of the continuous conf syntax check failure.

### How to reproduce it?
Put ${variable} into the Ingress annotations, like,

'nginx.ingress.kubernetes.io/upstream-vhost: mc.${teamServer}.dev.com'

![Invalid annotation](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/annotation.png)

And delete nginx-controller pods, then the new pods will fail

The above will not fail an already-running nginx-controller. As an already-running nginx-controller rejects to reload with bad rules. But once nginx-controller is restarted, it fails to retrieve all the rules, as a result, the /etc/nginx/nginx.conf always keeps empty with upstreams and /healthz location. With no /healthz location served, Kubernetes can’t detect a healthy probe, then it signals SIGTERM to nginx-controller to shut it down. Since pod can be self-healed in K8s, nginx-controller will be restarted again. But still fail with bad rules. This procedure loops a few times, and finally falls into the crash state.

### Some evidences if interested
Those can be proven by nginx-controller code and nginx-controller log

[https://github.com/kubernetes/ingress-nginx/blob/nginx-0.16.2/internal/ingress/controller/controller.go](https://github.com/kubernetes/ingress-nginx/blob/nginx-0.16.2/internal/ingress/controller/controller.go)

![Controller logic](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/controller.png)

It just returns on error if OnUpdate fails. The OnUpdate is responsible to validate the syntax of new nginx rules and decides whether to reload new conf or not. You can find it skips reloading bad rules once the testTemplate fails. The testTemplate is responsible to validate nginx configuration rules.

[https://github.com/kubernetes/ingress-nginx/blob/nginx-0.16.2/internal/ingress/controller/nginx.go](https://github.com/kubernetes/ingress-nginx/blob/nginx-0.16.2/internal/ingress/controller/nginx.go) 

![Nginx logic1](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/nginx.png)

![Nginx logic2](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/nginx2.png)

![Nginx logic3](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/nginx3.png)

nginx-controller log

![Nginx-controller log](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-04-02-the-analysis-of-an-nginx-ingress-incident/nginx-controller-log.png)

### My Suggestion
Checked with nginx-controller features and code for options/annotations if there is another way to figure it out, but no solution found by for now yet. The nginx-controller should work as stable as a gateway. Suggesting to discard bad rules and keep healthy ones when OnUpdate is hit with bad rules, at the meanwhile, log down what rules are discarded for track, which is some kind of fixing it fundamentally but may cost lots of efforts. So the lightweight enhancement should be,
1.	Validate ingress annotation syntax before applying into Kubernetes, which is some kind of preventive action.
2.	Employ ingress-class to isolate different applications' ingresses, which has the cons of requiring more nginx-controllers even with no fix in actual, merely narrow down the impact scope.
