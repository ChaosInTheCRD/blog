---
title: "When disaster strikes, count on... Velero?"
date: 2020-04-16T16:16:01+01:00
draft: false
tags: ["Kubernetes", "Tools", "Disaster Recovery"]
categories: ["Tools and Platforms"]
description: "How can Velero save your bacon?"
featured_image: "/img/05-Velero/veleros.jpg"
featured_image_preview: "/img/05-Velero/veleros.jpg"
---


The open source tool that makes backing up your Kubernetes Cluster plain sailing!
<!--more-->

Well surprise surprise, it looks like I'm back! Not at all the length of time that I was planning to be away for, but certain events (that I'm sure you will not want to spend another minute hearing about) led me to put our curious voyage together through the mystical universe of cloud-native, on the back burner for a little while. Now now, of course that does not mean that I haven't been exploring further; in fact, quite the contrary.

### So what have I been up to?
That's a great question! The past few weeks has actually given me a considerable amount of time to play around with a selection of different projects; some of them more productive than others; but alas, many of them could very happily overviewed on this blog, without having me veer too far off topic. Hopefully over the next few weeks, I will take strides to write some chapters on WTH explaining them. To tease your taste buds, here is a sneak preview of what I have been up to:

- Migration of Kubernetes workloads away from GCP, onto a freshly squeezed kubeadm-based home server 🍋
- The curious case of the kettle in the kitchen that corrupted the single-node etcd cluster 🔌⚡️
- Redesigning Kubernetes to be a Highly available, medusa-like god figure 🦹🏻‍♀️
- Securing Dashboards with OAuth2 Proxy magic 🐇


## Enough chit chat, tell me about Velero
Okay, okay I hear you, listening to my invalid excuses isn't much fun for anyone. But funnily enough, neither is system failure. And what's worse than system failure? I'll tell you for free; system failure with no disaster recovery solution! No disaster recovery solution, mitigation, or protection, for the kubernetes empire that you have worked tirelessly to setup over many months, if not years. But this begs the question; if your workloads are running in ephemeral containers, why not treat your cluster as an ephemeral beast as well?

### Ephemeral?
In the world of cloud native, containers by and large are treated to be ephemeral entities. In essence, this means their lifetime is viewed to be very short, before dying a valiant death. For this reason, data that we want to persist after a said death, to be used by the next container that comes along, is given its own dedicated resource to live in. For Kubernetes, this resource is called a persistent volume (PV), and solves many headaches. Try deploying [Grafana](https://blog.chaosinthe.dev/posts/04-prometheus/) dashboards on Kubernetes without any persistent storage; you'll see what I mean!


### And my Kubernetes Manifests... Where are they stored? 💾
Aha! now you're asking some important questions. The Kubernetes [manifests](https://devspace.cloud/docs/cli/deployment/kubernetes-manifests/what-are-manifests), are a big stack of yaml files that are presented to the API server. They describe the what, when, how and why of every application running on the cluster, and are stored on a database called [etcd](https://etcd.io/). Without going into this too deeply (I shall save it for another instalment), this a distributed, reliable key-value store; that is accessible cluster-wide. Due to its distributed nature, etcd usually runs in a cluster model of multiple nodes; just like kubernetes. Most commonly, this cluster is set up as a selection of docker containers. Containers everywhere!

So what happens when you have a single node etcd cluster that has its data corrupted? What happens when the power goes out in your house, and you power your cluster back on to find that your etcd node isn't responding? Well, of course i'm talking from experience, so I can tell you. You cry for a while, and then get up and say; looks like we'll have to just build all over again.

![sand castle](/img/05-Velero/sandcastle.gif "what happened to my old cluster? 😭")

### Planning for the inevitable
Well guess what, in the world of Kubernetes, disaster recovery is not something you should think about on day 2, or day 1... it should be thought out and implemented on day 0. The beauty of treating your containers as ephemeral, is that they can fail relatively frequently, and kubernetes will spin up new ones to take their place; mounting the persistent data in the appropriate places. But as highlighted by the etcd anecdote... my cluster would maybe benefit from taking some inspiration from this 'live fast, die young' attitude that the containers it runs have adopted. Its about time we designed our cluster with failure in mind. Wouldn't it be great to have our Kubernetes resources declaratively backed up in an automated fashion. Sounds great! Well... I hear you thinking "How do I do that?"... Velero, take centre stage.


## Please don't you rock my boat ⛵️

{{< figure src="/img/05-Velero/velero.png">}}

So we have deployed our highly-available kubernetes cluster; and we stand triumphantly over our containerised splendor... 3 master nodes, ready to serve our army of workers deployed beneath them, whatever the weather. But just in case our cluster can't quite handle the storm, let [Velero](https://velero.io) be your vessel to guide you through.

Giving you the capability to schedule automatic backups at recurring intervals, you can direct this tool to store a copy of whatever Kubernetes resource you desire. Whether it be persistent volumes, all namespaces, specific namespaces, the whole cluster with every nook and cranny; you name it, you can do it. The tool then enables the user to restore these resources back to the cluster at any time. The tool also supports the migration of these resources to other clusters, so you can save yourself the headache of starting from scratch with every new kubernetes instance. 

### But where are those backups going? 🤔
Good question. These backups are capable of going to a selection of supported object stores, from an array of public cloud and on-premise storage providers. These include AWS S3, Google Cloud Storage, Portworx and OpenEBS. What intrigued me most is, some third-party S3-compatible object store providers also work. So I was able to set this up to for my [Minio](https://min.io/), and avoid those nasty public cloud prices 🤭.

### So...
It sounds like this is the solution to all our problems! So how do we use it? Does it work as expected?... Lets find out.

## Time to give it a whirl 🌀
So if you have a look at the [overview](https://velero.io/docs/v1.3.2/index.html) documentation, we can see that Velero uses a client-server architecture, deploying a server that runs on the cluster upon install, after a client binary has been setup on the users local machine.

The basic install shows us that we can simply go ahead and install Velero on any machine that has kubectl access to the cluster you want to backup. I used MacOS, so installed the Velero client tool with the [brew](https://brew.sh/) package manager. Otherwise, the [latest release](https://github.com/vmware-tanzu/velero/releases/latest) tarball for your platform can be downloaded from Github.

From there, you need to setup and configure the object store, choosing from the list of providers mentioned above. I used minio, which provides AWS S3 compatible storage.

You will then need to perform a `velero install` command from your chosen client, with added parameters that describe your object store location. In my case the `install` command with flags looked like the following:

```console
chaointhecrd@home:~$ velero install \    
    --provider aws \ # name of provider
    --plugins velero/velero-plugin-for-aws:v1.0.0 \ # the plugin velero needs to use for minio
    --bucket velero \ # the name of the minio bucket
    --secret-file ./credentials-velero \ # the credentials for the minio server to be stored in a secret
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.168.0.210:9000 # specifying the location of the bucket (region and IP)
```

### Success! Now what to backup first...

So now there should be a namespace on the cluster called `velero`, and the command `velero get backup-locations` should list the object store of your choosing 😎.

Now this is where things get really slick. So we need something to backup that will test the power this tool really wields, right? Well I thought to myself; how about that [Weave Scope](https://www.weave.works/oss/scope/) namespace I have lying on my cluster, that I haven't really had the time to mess with yet. Sounds good. worst case; no important config lost 🙂.

#### 1. Creating the backup before wreaking havoc 🧨
As seen in the animation below, creating the backup is a simple case of telling the client: the name of the backup, and what resources should be included within it. Once this has been carried out, you should be notified that your backup request has been submitted, hurrah! You can then call `velero get backups` to check that it has completed successfully. Bare in mind that the time to backup after submission will depend on the size of the resources being backed up, as well as the network connection to the object store.

Once the backup is shown as `Completed`, you can go ahead and start creating problems for yourself. `kubectl scale deployment chaos-monkeys --replicas=1000` begins to ring in your eardrums 😼.

{{< figure src="/img/05-Velero/velero-backup.gif">}}


```console
chaosinthecrd@home:~$ velero create backup weave --include-namespaces weave 
# creates the backup
chaosinthecrd@home:~$ velero get backups
# fetches the backups, and shows their status
chaosinthecrd@home:~$ kubectl delete ns weave
# deletes the namespace... let the chaos begin
```
#### 2. Don't panic, and fall back on Velero 🙆‍♂️
So we've deleted the namespace, its time to panic. In fact no, it certainly is not time to panic. You were cunning, and planned for disaster on day 0, rather than day 2! Lets make the magic happen.

As shown in the animation below, I have lost the namespace `weave`, and if I don't get this sorted now; I will be crying myself to sleep, knowing that my users are unable to view the cluster resources zipping around in real time (don't worry, i'll do a post on Scope at some point; it's really cool).

Fear not! We created our backup with Velero, and we're not afraid ot use it. A simple `velero restore create` command is all it takes; and before you know it... look! Your namespace is right back where you left it! No `CrashLoopBackoffs`,`Pending` or `Error` pod statuses to worry about here 👌. 

{{< figure src="/img/05-Velero/velero-restore.gif">}}

```console
chaosinthecrd@home:~$ velero restore create --from-backup weave
# All we need to get 'backup' and running 😏
```

## Wrapping up

So, thank you for following along my anecdotal overview of Velero, and how I managed to stumble upon a truly magnificent tool, that has already saved my bacon a couple of times. The best part is that I have only scratched the surface of what Velero is capable of, and I will leave it to you with regards to taking your disaster recovery journey further into the abyss ✨.

As is tradition with these posts, I have chosen a video that would be worth watching if you want to dive further into the topic in a more practical fashion - check out the TGI Kubernetes Episode at the bottom of the page. It's definitely worth a watch.

Finally, a big thank you to members of the Velero team that helped me with some of the finer details within this post. I would also like to shout out a podcast hosted by one of the Engineers for Velero, Carlisia Campos. [The Podlets](https://www.youtube.com/playlist?list=PL7bmigfV0EqSh-btGOy8BLG3lsF0ylfZ-) is a weekly video podcast, where Carlisia and guests explore cloud native, one buzzword at a time. 

{{< youtube tj5Ey2bHsfM >}}
