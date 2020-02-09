---
title: "A Kubernetes Developer Workflow for MacOS"
date: "2019-01-24"
---

[Kubernetes](https://kubernetes.io/) development is not one-size-fits-all. Maybe
you’re learning Kubernetes with
[Minikube](https://github.com/kubernetes/minikube) on your local machine; maybe
you’re part of a large organization with many clusters; maybe your cluster is an
on-prem lab, or lives in the cloud.

But whether you’re a cluster operator managing policies, an app developer
test-driving a new service, or a data scientist running
[Kubeflow](https://www.kubeflow.org/docs/about/kubeflow), chances are you are
doing some (or all) of: connecting to a cluster, inspecting its state, creating
resources, and debugging those resources.

As a [developer relations engineer](https://twitter.com/mo_keefe) for
Kubernetes, I work a lot with demo code, samples, and sandbox clusters. This can
get interesting to keep track of (read: total chaos). So in this post I’ll show
some of the tools that make my Kubernetes life a lot better.

This environment can work no matter what flavor of Kubernetes you’re running,
and all these tools are made possible by the amazing open source community.

*****

### 💻 terminal

I use [iterm2](https://www.iterm2.com/) with the
[palenight](https://github.com/JonathanSpeek/palenight-iterm2) color scheme. On
top of that, I’m running [zsh](http://www.zsh.org/) and
[oh_my_zsh](https://github.com/robbyrussell/oh-my-zsh) with the default
`robby-russell` theme.

This theme has basic Git support but is otherwise minimal. If you’re interested
in displaying your current Kubernetes context in your shell prompt, check out
[kube-ps1](https://github.com/jonmosco/kube-ps1) or the
[spaceship](https://github.com/denysdovhan/spaceship-prompt#features) prompt.

![](https://cdn-images-1.medium.com/max/1600/1*JSiYljTJt2LyffjZ9_Irag.png)

Second, my `~/.zshrc` file has an essential line:

    source <(kubectl completion zsh)

This
[enables](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)
tab completion for kubectl commands. No more copy-pasting pod names!

### 🛶 navigating clusters

On any given day, I might switch between three clusters. This might describe you
too! Do you get annoyed anytime you have to open your `kubeconfig`?Same!
Luckily, there is [kubectx](https://github.com/ahmetb/kubectx) for exactly this
purpose:

<img src="https://cdn-images-1.medium.com/max/1600/1*vt7On1gLVnZyFKDScL-GJw.gif">


kubectx lets you navigate between cluster
[contexts](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#define-clusters-users-and-contexts)
easily. My favorite thing is running `kubectx -`, which takes you to the cluster
you were using last.

### ⚡️ superpowered kubectl

Now that we’ve got a cluster to work with, let’s do stuff.

Maybe you’ve felt that kubectl commands can get very long, with lots of
command-line flags. I have found that kubectl tab completion, plus a
comprehensive set of aliases (command shortcuts), helps a lot.

Here is a [great list of kubectl
aliases](https://github.com/ahmetb/kubectl-aliases), that lets you run things
like:

<img src="https://cdn-images-1.medium.com/max/1600/1*8j4WgwFfK_9wOHq5K2TUtg.png">
<span class="figcaption_hack">get pods</span>


<img src="https://cdn-images-1.medium.com/max/1600/1*trS4MZoird8kgIaM618eig.png">
<span class="figcaption_hack">describe pod</span>

Finally, I use a few [kubectl
plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/). But
manually setting these up can get annoying. So I use
[krew](https://github.com/GoogleContainerTools/krew), [an open-source plugin
manager ](https://medium.com/@ahmetb/kubectl-plugin-management-21dcd93e3721)for
kubectl:


<img src="https://cdn-images-1.medium.com/max/1600/1*1TjlIdS2DfKVabZ5ldCLCg.gif">

krew lets you browse, install, and use kubectl plugins, so that you can run
custom commands.

### 📜 wrangling yaml

Now that we have a cluster ready to go, let’s deploy something.

Developing on Kubernetes means writing, managing, updating, and deploying lots
of YAML files. I keep all my YAML files in
[Git](https://github.com/m-okeefe/helloworld). Adopting
[GitOps](https://www.weave.works/technologies/gitops/) early (rather than
keeping files locally) lets me see revision history as I’m doing early
debugging, and sets me up for success later if/when I start formalizing a
pipeline for the application I’m working on. (example: a Github webhook for a
CI/CD pipeline.)

I use [VSCode](https://code.visualstudio.com/) as my text editor, plus the
[Moonlight](https://marketplace.visualstudio.com/items?itemName=atomiks.moonlight)
theme. And while VSCode has a ton of great features on its own, Red Hat’s [YAML
Suppor](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)t
plugin is very handy for validation, autocompletion, and formatting.

<img src="https://cdn-images-1.medium.com/max/1600/1*ZGdvQwvn6_8WavC8L3XJ5Q.png">

Right now my process of writing Kubernetes YAML is fairly manual. But usually
for every new project, I’m writing the same Kubernetes specs: ConfigMap, Secret,
Volume, Deployment, Service.

I’m actively looking into ways to streamline this process, whether through text
editor extensions, [templating](https://github.com/kubernetes-sigs/kustomize),
or other tools. If you have tools that you use to help write and manage YAML,
comment down below! ⬇️

### ✨ deploying

We have our YAML files. Now we can deploy the resources! Given my souped-up
`kubectl` environment, it’s tempting to do this manually.

But this can be a struggle road, dragging you into the same `docker build`,
`docker push`, `kubectl apply`, and`kubectl delete pod` commands. No fun.

A tool called [skaffold](https://skaffold.dev/) can automate away much of this
pain. skaffold is magical: it watches your codebase for changes. When you save
changes locally, skaffold will automatically docker build, push a new image tag,
and redeploy into your cluster.

<img src="https://cdn-images-1.medium.com/max/1600/1*rF-csS89udEJ6w-O9KFEbg.gif">

One really cool thing skaffold does is auto-generate image tags. So in your
actual YAML, you just list the image repo, not the tag, and skaffold will
populate the new tag(s) on deploy.

    spec:
      containers:
      - name: helloworld
        image:  gcr.io/megangcp/helloworld
        imagePullPolicy: Always
       ports:
         - containerPort: 8080

All that skaffold needs is a (yes, YAML) configuration file:

    apiVersion: skaffold/v1beta3
    kind: Config
    build:
      artifacts:
      - image: gcr.io/megangcp/helloworld
    deploy:
      kubectl:
        manifests:
          - kubernetes/*

This is a minimal config where I specify my image repo (in this case, in [Google
Container Registry](https://cloud.google.com/container-registry/), but any image
registry like [DockerHub](https://hub.docker.com/) also works). I also specify
directory where my manifests live.

skaffold [is highly
customizable](https://github.com/GoogleContainerTools/skaffold/tree/master/examples),
and can work with deploy tools like Helm in addition to kubectl.

### 🐳 inspecting docker images

skaffold abstracts the docker build process, but sometimes I want to look into
my newly-built images to inspect things like: what is the image size compared to
previous versions? what are the contents of each of my image layers?

[dive](https://github.com/wagoodman/dive) is an amazing tool for inspecting
Docker images.

<img src="https://cdn-images-1.medium.com/max/1600/1*zLIgc-0sHlfLAH0eJ06S6g.gif">

With dive, I can examine filesystem changes between different layers. This is
super helpful if something in my Docker build has gone wrong.

### 😱 debugging

Now we’ve got pods running Kubernetes. What next?

Every so often (read: all the time) something goes wrong — with my spec, or with
my application code.

My kubernetes debugging workflow is usually:

1.  **Describe the pod** (`kdpo` alias). Is it my spec’s fault? (example: is the
Deployment trying to mount a Secret I accidentally put in a different
Namespace?) If not…
1.  **Get the pod logs**. the [skaffold
dev](https://skaffold.dev/docs/getting-started/#skaffold-dev-build-and-deploy-your-app-every-time-your-code-changes)
command will combine all the logs for every container deployed and stream all of
it to stdout. But I’ve found that when I have two or more pods running, that
format gets noisy. At the same time, the usual `kubectl logs` command can result
in an endless cycle of copy-pasting new pod names.

[stern](https://github.com/wercker/stern) is a great alternative for tailing
logs in a more customizable way. stern uses regular expressions to select on
pods — and given that all pods start with their deployment name, you can follow
logs for all the pods in a deployment, without having to know the exact pod
name. Super helpful:

<img src="https://cdn-images-1.medium.com/max/1600/1*WqZ3eEhhbt8Wau_G6DWDyQ.png">

If the logs aren’t giving me clues for what’s gone wrong, usually I’ll…

3. **Exec into the pod** (`kex` alias with tab completion):

<img src="https://cdn-images-1.medium.com/max/1600/1*RnF4LX330A4slnax3cvt3A.png">

### that’s a wrap

Kubernetes is a big, complex piece of software with a large configuration model.
I hope sharing some of these tools might help you, wherever you’re at in your
k8s journey.

Here’s the full list of tools and plugins mentioned in this post:

* [iterm2](https://www.iterm2.com/) /
[palenight](https://github.com/JonathanSpeek/palenight-iterm2) /
[oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
* [kubectl tab
completion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)
* [kubectx](https://github.com/ahmetb/kubectx)
* [kubectl aliases](https://github.com/ahmetb/kubectl-aliases)
* [krew](https://github.com/GoogleContainerTools/krew)
* [VSCode](https://code.visualstudio.com/) with
[GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
* [skaffold](https://github.com/GoogleContainerTools/skaffold/tree/master/examples/getting-started)
* [dive](https://github.com/wagoodman/dive)
* [stern](https://github.com/wercker/stern)
