---
title: "The right way to do Databases in Kubernetes"
date: 2025-01-26T13:19:16-07:00
tags: [Kubernetes, AWS, CloudNativePG]
---

There's this weird stigma with running databases in Kubernetes.

Here's the gist- <!--more--> It's due to a combination of Kubernetes being stateless, having ephemeral workloads, complications with storage, and requiring additional overhead that just makes it difficult to maintain on a cluster.

## Complications with Kubernetes

Contrary to typical Kubernetes workloads, databases are inherently stateful. 

A stateful application usually interacts with a database, where prior information influences the app's behavior. 

A stateless application typically does short term tasks and thus, therefore doesn't need any prior information, nor does it need to store its state anywhere.

In short, this is where the Kubernetes database dilemma stems from. Stateful holds data that must be preserved. Stateless does not store persistent data.

However, there are special operators that solve these problems, and automate most of the overhead. [CloudNativePG](https://cloudnative-pg.io/) (CNPG) is a fantastic example.

## The CloudNativePG Operator

The reason a databases needs to be stateful boils down to a few things. A few of which are needing to maintain identity between replicas, deployment order, and keeping one instance per volume.

CNPG maintains these necessities for us. Today, I'll install a database cluster into my [homelab](https://github.com/clumpspirit/homelab) for my bookmark manager, [linkding](https://github.com/sissbruecker/linkding). I'll include a backup to an AWS S3 bucket, and how to setup monitoring.

I should probably note that I chose linkding because it integrates well with an external database, providing environment variables to specify cluster information.

### Installing the Operator

Everything here will be deployed through GitOps. You can also check out my homelab [repository](https://github.com/clumpspirit/homelab) to see my directory structure.

The operator itself installs all the necessary custom resource definitions for the databases to run off of.

```
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cnpg
  namespace: cnpg-system
spec:
  interval: 1h
  chart:
    spec:
      chart: cloudnative-pg
      version: "0.23.0"
      sourceRef:
        kind: HelmRepository
        name: cnpg
        namespace: cnpg-system
      interval: 12h
```

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cnpg
  namespace: cnpg-system
spec:
  interval: 24h
  url: https://cloudnative-pg.github.io/charts
```

These are some pretty basic Helm manifests. I avoid using the 'latest' release tag because I have [renovate](https://docs.renovatebot.com/modules/manager/flux/) set up to look for updates automatically.

After you apply these manifests, it'll install all the necessary CRDs to create a database cluster.

### Creating a Cluster

One of the CRDs it defines is a *Cluster*, which is what this section will setup.

To deploy a database with 3 replicas, I created a *Cluster* manifest in my linkding cluster directory.

```
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: linkding-database-cluster
spec:
  description: Postgres database cluster for linkding
  imageName: ghcr.io/cloudnative-pg/postgresql:16.6-30-bookworm
  instances: 3

  inheritedMetadata:
    labels:
      app: linkding-database

  bootstrap:
    recovery:
      source: clusterBackup
      database: linkding
      owner: linkding

  externalClusters:
    - name: clusterBackup
      barmanObjectStore:
        destinationPath: "s3://hl-database-clump/linkding/"
        serverName: linkding-backup
        s3Credentials:
          accessKeyId:
            name: aws-creds
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: aws-creds
            key: ACCESS_SECRET_KEY

  backup:
    barmanObjectStore:
      destinationPath: s3://hl-database-clump/linkding/
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
    retentionPolicy: 14d

  monitoring:
    enablePodMonitor: true

  storage:
    size: 1Gi
```

It's kind of long, but there's a lot of important things here.

I'm bootstrapping (creating) the cluster based on a previous backup that I had in my S3 bucket in the cloud. The bootstrap reference is the *externalClusters* object directly below it.

**note:** I was stuck on this for a while, but you need to make sure the name of the backup directory you're bootstrapping from is DIFFERENT than the new one the cluster will create. The new one's name will default to the name of the cluster (linkding-database-cluster in this case).

Furthermore, I'm creating a new place to back the data up in the *backup* object further down. It has the same destination path as the bootstrap object, but there will be a new file created inside of it. I specified the use of the *gzip* compression method to save space in the cloud.

**note:** ensure that you have the proper AWS IAM role permissions for the credentials you provide it. These permissions must allow CNPG to create, delete & get objects from the bucket. Also, make sure the bucket actually exists, create one if necessary.

Moreover, I made sure that I enabled pod monitoring. This will create a *PodMonitor* resource that will export metrics to Prometheus. I'll touch more in that in the monitoring section.

Finally, I set the size of each of the 3 instances to 1Gi. This is plenty for my use case, which is just to store my bookmarks.

### Configuring Linkding

Linkding, of course, needs to be made aware of the new database and start putting its data there. Ensure you create a proper backup of your data if you haven't already. I found that the easiest way to do it in this case was to export my bookmarks in .html format through the interface.

```
apiVersion: v1
data:
    LD_DB_ENGINE: postgres
    LD_DB_HOST: linkding-database-cluster-rw
    LD_DB_PORT: "5432"
kind: ConfigMap
metadata:
    creationTimestamp: null
    name: linkding-configmap
```

Create this *ConfigMap* manifest. 

These are environment variables from linkding.

- LD_DB_ENGINE - we're telling linkding this is a postgres database.
- LD_DB_HOST - this is the name of the service created by the cluster manifest from above.
- LD_DB_PORT - the default port for most Postgres databases.

Additionally, you have to tell linkding the credentials for the database. I will be using the credentials that are automatically generated by CNPG in the secret it creates:

```
          # these envs are managed by CloudNativePG
          env:
          - name: LD_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: linkding-database-cluster-app
                key: password
          - name: LD_DB_USER
            valueFrom:
              secretKeyRef:
                name: linkding-database-cluster-app
                key: user
```

This is what I have in my linkding deployment manifest. Unfortunately, I really don't like this solution because I want to keep things to a single Secret or ConfigMap object. But, due to the nature of the situation, I ended up just referencing the Secret that is generated by the cluster manifest. If this is confusing to you, it's because this method sucks, which is why I'll be moving to an External Secrets Operator soon. Stay tuned!

### Scheduling a Backup

The cluster manifest points the backup to the right place, but it doesn't actually create a backup automatically. You can trigger a manual backup on the CLI with the Kubectl CNPG plugin, but that's way too much work to keep track of. It's cool if you want an initial backup, which is how I did it while testing out the operator.

CNPG defines a resource we can use called the *ScheduledBackup*. Here's what mine looks like:

```
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: linkding-database-cluster
spec:
  immediate: true
  schedule: "0 0 3 * * *"
  backupOwnerReference: cluster
  cluster:
    name: linkding-database-cluster
```

It's a simple manifest. With the *immediate* key, it'll create a backup to S3 as soon as this manifest is deployed.

The *schedule* is basically CronJob format, but the first number is seconds, which CronJob doesn't include. With that said, I have scheduled a backup every single day at 03:00 in the morning.

### Setting up Monitoring

In the cluster manifest, pod monitoring is enabled, so once the change is pushed, it should create a *PodMonitor* resource in the same namespace as the cluster that will export the database metrics to Prometheus.

But that's not all. Technically, if all you want are the metrics, then that's actually all you would need to do. We, however, also want the beautiful Grafana dashboards.

Luckily, CNPG also provides a Helm chart for the dashboards. In the monitoring directory and namespace of my cluster, I added these two manifests:

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cnpg-grafana
spec:
  interval: 24h
  url: https://cloudnative-pg.github.io/grafana-dashboards
```

```
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cnpg-grafana
spec:
  interval: 1h
  chart:
    spec:
      chart: cluster
      version: "0.0.3"
      sourceRef:
        kind: HelmRepository
        name: cnpg-grafana
        namespace: monitoring
      interval: 12h
```

With these manifests, Grafana should have automatically detected and created a *CloudNativePG* dashboard.

**note:** if your dashboard is void of data at this point, it's probably your Prometheus configuration. Make sure you have this configuration in your release in order to detect metrics in all namespaces:

```
    prometheus:
      prometheusSpec:
        # allows servicemonitor and podmonitor discovery across all namespaces
        serviceMonitorSelectorNilUsesHelmValues: false
        podMonitorSelectorNilUsesHelmValues: false
```

After that, slowly but surely, the CNPG dashboard will become populated with the beautiful data we all crave.

I love that the dashboard contains backup information. Feels great knowing my data is sitting cozy in the cloud.

## Conclusion & Takeaways

With this setup, you should have a CNPG database cluster with 3 instances backed up to the cloud and with monitoring.

The idea that you shouldn't run databases in Kubernetes is a bunch of baloney. So long as you are using all the tools Kubernetes is capable of, a database solution works smoothly in your cluster. This means working with custom resource definitions, custom operators, controllers, etc.

One of the key takeaways I had from this project was to TRUST the logs. When setting up recovery, I was getting a constant error failing to bootstrap the database from recovery. The log said it was expecting an empty archive, which I misinterpreted as the backup needing to be empty. This made no sense to me. I read the docs and found out that I was pointing the new backup to the existing backup. This was a no no. This is a **reminder** to READ the friendly manual! 

Finally, if any of the steps in this process were confusing, I welcome you to explore my [homelab](https://github.com/clumpspirit/homelab) repository and find all of the configurations & their respective locations.
