---
title: "Recap: Multicluster Kubernetes at Kubecon EU '18"
date: "2018-05-16"
---

This month, I attended [Kubecon + CloudNativeCon EU](https://events.linuxfoundation.org/events/kubecon-cloudnativecon-europe-2018/), in Copenhagen, Denmark. This conference covers not only [Kubernetes](https://kubernetes.io/) but also the growing number of open source projects within the [Cloud Native Computing Foundation](https://www.cncf.io/projects/).

It was an exciting conference, with lots of buzz especially around [Serverless](https://www.youtube.com/watch?v=_1-5YFfJCqM&index=216&list=PLj6h78yzYM2N8GdbjmhVU65KYm_68qBmo&t=0s) and Machine Learning with [Kubeflow](https://www.youtube.com/watch?v=I6iMznIYwM8). I think we are also seeing that as more enterprises move to Kubernetes, the more stories they have to tell about their use cases, deployments, and challenges; overall, my favorite talk, [from Sarah Wells](https://www.youtube.com/watch?v=H06qrNmGqyE&t=0s&index=144&list=PLj6h78yzYM2N8GdbjmhVU65KYm_68qBmo) of the London Financial Times, included some awesome insights about adopting Kubernetes as an organization.

But for me, most relevant were the talks about multicluster Kubernetes. In my current role, I think a lot about: how to lifecycle and manage lots of clusters from one touchpoint? How to orchestrate applications across clusters?

There is a Special Interest Group within the Kubernetes organization [dedicated to multicluster](https://github.com/kubernetes/community/blob/master/sig-multicluster/README.md). Formerly called SIG Federation, this group has shifted their focus from “one size fits all” solutions, like the [Federation v1](https://kubernetes.io/docs/concepts/cluster-administration/federation/) API, to a set of smaller, flexible tools that can fit a variety of use cases. With that new goal, some exciting projects (such as [cluster-registry](https://github.com/kubernetes/cluster-registry) and [kubemci](https://github.com/GoogleCloudPlatform/k8s-multicluster-ingress)) have emerged from that community.

With all these changes afoot, it can be hard to step back and think about larger trends. Here is my attempt to tease out a few observations about the latest thinking for multicluster Kubernetes.

## Trend #1 - A Move to Disaggregation 🌌

The most common pain point I’ve observed around K8s Federation is that when you deploy the Federation API Server into one cluster, that cluster becomes a single point of failure. Rob Szumski’s [talk](https://www.youtube.com/watch?v=zVOIk7nO_ts), “Kubernetes Multi-cluster Without Federation,” touched on this. He also brought up an important issue around security: to work, the Federation API running on one cluster must be root, able to access all the namespaces in all the clusters.

I liked Szumski’s talk because he spoke to the big picture of what exactly operators need to do with multiple clusters. He made a key distinction between the infrastructure admins (who must connect, track, and secure the clusters) and the application owners (who must be able to execute [CI/CD](https://www.atlassian.com/continuous-delivery/ci-vs-ci-vs-cd) pipelines across those clusters, work with secrets, failover policies, and set up cross-cluster [service discovery](https://www.datawire.io/guide/traffic/service-discovery-microservices/)).

Szumski used the common ground between those two user groups – the need for a clusters “source of truth,” and for security – to explore alternatives to Federation that promote a more disaggregated approach. In this approach, each cluster is autonomous, and no single cluster provides the entry point into all the others.

One option, he said, is to deploy [cluster-registry](https://github.com/kubernetes/cluster-registry) onto all the clusters, and then to write a small agent (also running on each cluster) to sync the `/cluster` resources. Users can then define policy by selecting clusters from the registry and creating a `ClusterPolicy`. A key advantage of this approach is that unlike Federation, clusters never have full write access to other clusters; for example, one app owner would not be able to modify a different application running another cluster in the group.

Disaggregated multicluster intelligence, then, can help promote security best practices for multicluster in addition to promoting resilience. If one of Szumski’s cluster-registry deployments goes down, for instance, `n-1` will still be running.

## Trend #2 - Hybrid Cloud -> Service Mesh 🕸

Sometimes, clusters live in the same Cloud - for instance, a user could have Dev, Build, and Prod clusters, all in AWS. But to an increasing degree, users are dealing with clusters that span different locations (on prem, public cloud), and are running applications where [microservices](https://en.wikipedia.org/wiki/Microservices) span these different locations. These users need a way to handle load balancing across clusters, routing policies that span multiple clusters, cross-cluster service discovery, and failover policy.

[Istio](https://istio.io/) aims to fill these gaps in the future. One cool Google/Cisco [talk](https://www.youtube.com/watch?v=bLJL53UIcqI) explored how users can make use of new Istio multicluster features to run applications that span multiple clouds. In a multicluster Istio setup, one cluster runs a primary [Pilot](https://istio.io/docs/concepts/what-is-istio/overview.html#pilot), through which users can define cross-cluster policy. This primary Pilot knows about all the [Envoy sidecars](https://istio.io/docs/concepts/what-is-istio/overview.html#envoy) running across all the clusters. In their demo, they showed different versions of the same microservice running in two different GKE clusters, then demonstrated that by setting up an Istio rule in the primary pilot, they could send all traffic to the latest version running in one cluster.

I think the idea of a multicluster [service mesh](https://thenewstack.io/history-service-mesh/) is really exciting, and also reflects the disaggregation trend for multicluster; in Istio, by relying on lots of Envoy proxies to intelligently handle traffic into your applications, you are offloading control to each of your multiple clusters.

## Trend #3 - Cluster Management at Scale 🔁

Szumski ran a survey before his “Multicluster Without Federation” talk, and found that on average, enterprise Kubernetes customers have 5-10 clusters total. But I was at Kubecon to give [a talk](https://www.youtube.com/watch?v=UR8N6mIAFlM) on Edge Computing, where operators could be dealing with many _thousands_ of clusters. In this case, cluster operators need both disaggregated _and_ aggregated control. For instance, they must be able to manage namespaces across clusters, but then allow each cluster to make its own decisions about (for instance) request routing.

I think the community is in the early stages of thinking about how this could actually work. For instance, one great Heptio [talk](https://www.youtube.com/watch?v=5kMz2oJgV0A) (“Clusters as Cattle”) stressed the importance of treating clusters as disposable infrastructure, where cluster [install](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) and upgrade must be automated and seamless. This is especially true in an at-scale multicluster use case where manual cluster lifecycling could be unrealistic. That talk introduced a multicluster app migration tool called [Heptio Ark](https://github.com/heptio/ark), a CLI that allows you to back up Kubernetes objects, then restore them from `.tar` files onto new clusters. I think this is definitely a step in the right direction for at-scale cluster management.

## Looking Ahead

If I had to boil all these observations into one, it would be that there is no single solution or tool that will satisfy all the multicluster use cases. Example: [for CERN](https://www.youtube.com/watch?v=2PRGUOxL36M&t=0s&index=75&list=PLj6h78yzYM2N8GdbjmhVU65KYm_68qBmo), traditional Federation v1 actually works really well. Federation allowed them to deploy a Daemonset to 200+ clusters at once, in order to execute distributed scientific workloads.

Beyond v1, work is in progress on [Federation v2](https://github.com/kubernetes-sigs/federation-v2), which provides not simply a replica of the traditional K8s API (as was the case in v1), but instead a toolkit for developers to define their own federated workload types by [implementing](https://github.com/kubernetes-sigs/federation-v2#concepts) Templates, Placements, and Overrides. This would support not only a manual workload placement paradigm, but also an [automated system](https://docs.google.com/document/d/1ihWETo-zE8U_QNuzw5ECxOWX0Df_2BVfO3lC4OesKRQ/edit#heading=h.cjzhckjkyyjf) where the Scheduler places workloads based on user-defined policy.

Overall, I think that the multicluster Kubernetes space is changing rapidly as the community learns from mistakes, learns about new user needs, and aligns under a set of new tools— both within and outside the SIG. [Operators](https://coreos.com/operators/), for instance, were something I had never heard of prior to Kubecon, but in the future could be an exciting way for app owners to handle upgrades across multiple clusters.

I think that as this year goes on, we will start to see more consensus around which tools
work best for large swaths of multicluster users. I’m excited to see what’s next!
