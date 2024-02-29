---
layout: post
title: "AWS RDS deployment types"
---

I was tasked with writing AWS CDK source code for infrastructure of a project.
The specification I had available was one of those infrastructure map pictures with all the frames and icons of AWS services.
Not ideal but in some ways more descriptive then couple sentences.
Among other things it called for postgres RDS database setup with three instances.
A primary instance and two stand-by replicas.

---

Let's review what I knew about rds deployment types:

- **Single db instance** it is just what it sounds like. If you see `no` next to `multi-az` in database listing in rds web console, this is it.

- **Multi-AZ db instance** is pair of db instnaces deployed in two different availability zones.
  Primary one serves all reads and writes. The other instance is not serving any requests.
  It is kept synced and ready. If malfunction of primary is detected, AWS automatically fails over to this second instance.
  When you use `create-db-instance` aws cli rds command with `--multi-az` option, this is what you get.

  In list of databases in web rds console this shows as single database intances with `multi-az=Yes`. Also if you click through to details there is only single instance shown in replication section.

- **Multi-AZ db cluster** consists of primary and two secondary instances. They too serve as failover
  nodes. But they can also serve read traffic. Failover detection and execution should be faster as well.
  To create this using aws api you need to use `create-db-cluster` command instead.

  You can recognize this as `multi-az=Yes` and also there is additional graphics in db listing showing those reader instances.

---

Now on to the CDK side. This is my first encounter with AWS CDK. Primarily I try to use constructs classes as
that's seems to be where part of CDK advantage over CloudFormation is.
I went straight for [DatabaseCluster](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds.DatabaseCluster.html)
construct. It's initializer has `engine` parameter of `IClusterEngine` type.
Only way how the library allows me to create value of that type is through
static methods of [DatabaseClusterEngine](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds.DatabaseClusterEngine.html).
However there's only support for Aurora based clusters.

I knew Aurora is a database product by AWS but not much more. CDK api suggested that every multi-az database cluster
automatically means Aurora deployment.

---

It turns out this is not the case. One can totally have non-aurora multi-AZ postgres database cluster.
This can be seen from CloudFormation class for [DBCluster](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds.CfnDBCluster.html).
It has [engine](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_rds.CfnDBCluster.html#engine) constructor prop
which allows for these values: `aurora-mysql`, `aurora-postgresl`, `mysql` or `postgres`.
I got misled by CDK's constructs api once again.

---

## The difference

So what is a difference between `aurora-postgresql` and `postgres` multi-az clusters?
They differ in how the nodes communicate and synchronize data.

In **vanilla postgres cluster** each database node has its own local storage. Postgres takes care about all data synchronization.
It also handles about which node is primary etc. This is all happening using protocols native to Postgres.

**Aurora db clusters** on the other hand uses something AWS calls **cluster volume**. It is high-performance, distributed storage system
which grows/shrinks as needed. Distributed means that it spans multiple availability zones. It manages data replication by itself.
This custom storage system solves many problems database software otherwise needs to take care about.

Aurora multi-az db cluster uses this custom storage layer. All db instances (writer/readers) connect to this volume but not directly to each other.

In Aurora cluster all nodes (writer/readers) use this custom storage layer. All communication and data synchronization is done via
this cool storage. Database software in these nodes does not talk to each other directly any more.:0000/0000/0000/cccc

---

## Summary

Aurora and multi-az db clusters are two different concepts. The fact that in current version of aws cdk api one cannot use dbcluster construct to create non aurora cluster is just drawback of construct definition.

Using DBCluster CloudFormation class directly it was possible to create what I needed.

```
new rds.CfnDBCluster(this, 'rds', {
  engine: 'postgres',
  engineVersion: '15.5',
  ...
})
```

---

Resources
- [RDS Multi-AZ DB cluster deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html) - AWS Doc
- [Amazon Aurora DB clusters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html) - AWS Doc
- [Insightful answer about Aurora deployments](https://stackoverflow.com/a/31978929) - StackOverflow
- [High level RDS Multi-AZ comparison](https://aws.amazon.com/rds/features/multi-az/) - AWS landing page
