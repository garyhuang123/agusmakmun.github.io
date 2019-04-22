---
layout: post
title:  "The Analysis of Kernel & OS for running Kubernetes"
date:   2019-03-21 12:18:23 +0800
categories: [kubernetes]
---

### The Conclusion

#### Kernel

Kernel 4.x should be the must choice for Kubernetes/Docker.

#### Guest OS

- Ubuntu Server LTS, if to be conservative on stability & large developer group support.
- RancherOS, if to save operation efforts with less to patch, and more automation on creation/deletion/upgradation
- CentOS is too old to container world.

### Why Kernel 4?

#### Kernel requirement for Containerd

https://github.com/containerd/containerd/blob/master/README.md
![Kenerl requirement for containerd](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-03-21-the-analysis-of-kernel-and-os-for-running-kubernetes/kernel_requirement_for_containerd.png  "Kenerl requirement for containerd")

#### Kernel requirement for Kubernetes

Kernel 3.10 is just the minimum requirement, you can see 4+ or even 5+ is also supported by Kubernetes. The latest CentOS7.6 releases on kernel 3.10. No any plan for kernel 4. While some of the kernel feature like ‘kernel memory accounting (kmem)’ that Kubernetes enables by default is only experimental in kernel 3, but stable on kernel 4. Docker provides the kernel version check and disables kmem if kernel is 3.x. But I can’t find any flags to disable kmem from Kubernetes. One possible way to fix it is to rebuild Kubernetes from source code with build flag to disable the kmem accounting from runc.
https://github.com/kubernetes/kubernetes/blob/master/cmd/kubeadm/app/util/system/types_unix.go
![Kenerl requirement for kubernetes](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-03-21-the-analysis-of-kernel-and-os-for-running-kubernetes/kernel_requirement_for_k8s.png  "Kenerl requirement for kubernetes")

#### Kernel requirement for GKE

Take GKE as a reference, it requires kernel 4.4+
https://github.com/kubernetes/kubernetes/blob/8993fbc543c18e73668793b5d5e234c0a136735c/test/e2e_node/system/specs/gke.yaml
![Kenerl requirement for GKE](https://raw.githubusercontent.com/garyhuang123/garyhuang123.github.io/master/static/img/_posts/2019-03-21-the-analysis-of-kernel-and-os-for-running-kubernetes/kernel_requirement_for_gke.png  "Kenerl requirement for GKE")

#### Other situations

1. The kernel memory leak bug in 3.x that has been happening for quite a few weeks for us: SLUB: Unable to allocate memory on node -1, which crashes the Docker node. The kernel memory accounting is only experimental on kernel 3, but stable on kernel 4.No fix for CentOS now. See issue, https://github.com/kubernetes/kubernetes/issues/61937.
2. The kernel socket leak in 3.x I have met: unregister_netdevice: waiting for eth0 to become free. Usage count = 1. It is fixed in Ubuntu kernel 4.4.114. [net: tcp: close sock if net namespace is exiting](https://cdn.kernel.org/pub/linux/kernel/v4.x/ChangeLog-4.4.114.). See [changlog]( https://launchpad.net/ubuntu/+source/linux/4.4.0-118.142). You can also find the fix by linus torvalds in [net: tcp: close sock if net namespace is exiting](https://github.com/torvalds/linux/commit/4ee806d51176ba7b8ff1efd81f271d7252e03a1d). But no CentOS fix was ever found so far. I have rebuilt [an enhanced CentOS on kernel 3.10.0.957.10.1](https://github.com/garyhuang123/kubernetes-centos-kernel/releases).
3. Kernel 3.10 is only tech preview on overlayfs. kernel: [ 24.062493] TECH PREVIEW: Overlay filesystem may not be fully supported.
4. As you can see from the above requirement examples, seems the container ecosystem mainly develops on kernel 4.x now. It should be so in the future as well. Vendors will not prefer to provide fixes for kernel 3, take the kernel socket leak bug for example, it has been existing for several years on CentOS.
5. Even if we fix those 2 kernel bugs, how can we be sure that the future unknown potential bugs can be officially fixed in time? But the kernel 4.x can be timely patched once such critical bugs are found, as the whole ecosystem mainly plays on it.
6. Docker has the fixes for kernel memory leak, while k8s doesn’t have, seems it mainly considers & develops on kernel 4.x. Also Kubernetes is going to work without Docker, it has its own crictl tools which can replace docker client. Moreover it includes the containerd/runc as its vendor code directly. It is now possible to totally discard Docker when setting up a Kubernetes cluster, see https://aboutsimon.com/blog/2018/07/17/Kubernetes-using-containerd-1.1-without-Docker.html
7. Kubernetes only patches the last 3 minor versions. And Minor releases occur approximately every 3 months. So we are moving to the next minor version in 9 months, as Kubernetes will depend more and more heavily on kernel 4.x features. We will be in high risk on kernel 3.x when k8s version goes up. If we won’t upgrade following the pace, we can’t get the latest security patches from Kubernetes.
8. Docker now only patches 17.06+. The latest version is 1809.3. Redhat still uses Docker 1.13.1, because their kernel is too old to run newer Docker. The 1.13.1 is even older than 17.03, which is no longer in maintenance scope.
9. CoreOS adopted kernel 4 very early. In 2015
10. CentOS officially doesn’t support kernel 4.x by for now. https://en.wikipedia.org/wiki/CentOS.
11. RHEL 8 defaults on kernel 4.x, but it is beta. No timeline for GA yet. https://developers.redhat.com/rhel8/getrhel8/

### My suggestions

- Ubuntu Server LTS
- RancherOS

Those 2 are both on Rancher & Aqua support agreements. As the application may be still running on CentOS image, it should not be affected by whatever the guest OS is as long as it is Linux-kernel-based. That is also an advantage of container features, portable/consistent.

Using kernel 4.x is a long-term consideration, as container vendors would not be likely to spend time on 3.x support. The container ecosystem is moving forward so quickly on kernel 4.x. Furthermore, it is time-consuming for troubleshooting & stabilizing the cluster. To be safer, currently Ubuntu Server LTS may be the better choice as free guest OS if we want to be more conservative. Since RancherOS is still young. Actually I have tried the RancherOS 1.4 several months ago. And found a bug on dhcp/static ip assignment, then I reported to Rancher guys, they fixed it during that night. Very quick response. Now it is 1.5.1 using kernel 4.14 which directly downloaded from https://www.kernel.org/, see https://github.com/rancher/os-kernel/blob/1523fd5b73f85f1b72628a0ce62ca2900eed8948/Dockerfile.dapper. To save operation efforts & with much less attack surface, RancherOS is the choice.

#### The reasons for Ubuntu

1. Ubuntu has a wide range of kernel version support. https://wiki.ubuntu.com/Kernel/Support
2. Ubuntu responds more quickly than CentOS on fixing container related kernel bugs
3. Ubuntu is a major player in the cloud segment and provides certified images. Canonical provides certified Ubuntu images, as well as professional support for users on AWS.
4. The support for Ubuntu Server Long Term Support (LTS) lasts for 5 years 
5. Ubuntu security update notices can be timely & easily found from https://usn.ubuntu.com/
6. We can find many Docker demo/practice articles based on Ubuntu.
7. Valuable article
8. 10 Reasons Why Ubuntu Is Killing It In The Cloud 2016

#### The reasons for RancherOS

1. Small & clean & lean without useless system components, smaller attack surface
2. More automated operations with Rancher drivers
3. Easy to patch
4. Can get commercial support from Rancher directly

To dig more.

