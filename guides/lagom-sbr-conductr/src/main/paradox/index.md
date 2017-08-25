# Deploying Lagom Microservices with Akka's Split Brain Resolver

[Lagom](http://www.lagomframework.com/) is an opinionated microservices framework that makes it quick and easy to
build, test, and deploy your systems with confidence. [Akka SBR](http://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html) handles network partitions automatically and is a feature of the Lightbend Reactive Platform exclusively available for Lightbend customers.

## The Challenge

A fundamental and common problem in distributed systems is that network partitions (split brain scenarios) and machine crashes are indistinguishable for the observer, i.e. a node can observe that there is a problem with another node, but it cannot tell if it has crashed and will never be available again or if there is a network issue that might or might not heal again after a while. Temporary and permanent failures are indistinguishable because decisions must be made in finite time, and there always exists a temporary failure that lasts longer than the time limit for the decision.

## The Solution

Lagom services combined with Akka SBR can help avoid the consequences of network partitions thereby adding further resilience to your system. This guide shows how to add Akka SBR to your Lagom service. As a follow on, the guide then demonstrates a network partition in action using ConductR, the [application management component of the Lightbend Enterprise Suite](https://conductr.lightbend.com/docs/2.1.x/Home).

## 1. Build your "Lagom with Scala" service

Follow the steps at [Using Lagom with Scala](https://www.lagomframework.com/get-started-scala.html)

## 2. Enable Akka Cluster & Akka SBR

There are various reasons why you may decide to use Akka Cluster with Lagom, including persistence and publish-subscribe.

> See the [Lagom documentation for more information on Lagom with Akka Cluster](https://www.lagomframework.com/documentation/1.3.x/scala/Cluster.html#Cluster).

In the following `build.sbt` of your service, Akka clustering is already added given the dependency on `lagomScaladslPersistenceCassandra`. Akka SBR is added via the `akka-split-brain-resolver` dependency.

@@snip [build.sbt](../../../lagom-scala-sbt/build.sbt) { #hello-lagom-impl-build }

> In order for the build to download Akka SBR you will require credentials. You find your credentials at https://www.lightbend.com/product/lightbend-reactive-platform/credentials including links to instructions of how to add the credentials to your build.

The next step is to enable the Split Brain Resolver by configuring it with akka.cluster.downing-provider-class in the configuration of the ActorSystem (*application.conf*):

@@snip [application.conf](../../../lagom-scala-sbt/hello-lagom-impl/src/main/resources/application.conf) { #hello-lagom-impl-conf }

The above configuration declares that Akka SBR will be used and it will use a `keep-majority` strategy. Akka SBR has a number of strategies and "keep majority" will down the unreachable nodes if the current node is in the majority part based on the last known membership information. Otherwise, "keep majority" will down the reachable nodes, i.e. the own part. If the parts are of equal size the part containing the node with the lowest address is kept.

The configuration also declares that Lagom should exit when Akka SBR makes a decision to down a node. We do this so that any orchestrator in use (such as ConductR, or Kubernetes) will restart the service elsewhere given a non-zero exit value.

> Consult the [Akka SBR documentation for more information on its available strategies](http://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html#strategies).

## 3. Deploy your service to ConductR

At this stage, we have a service that is ready to be deployed. Simply add [ConductR's plugin](https://github.com/typesafehub/sbt-conductr#sbt-conductr) to your build's `plugins.sbt` so that we can package and deploy your service:

@@snip [plugins.sbt](../../../lagom-scala-sbt/project/plugins.sbt) { #hello-lagom-plugins }

From the `sbt` project (press RETURN if you're still running the `runAll` command from having built "Lagom with Scala"), reload your project in order to realize its new settings:

```
sbt> reload
```

...and then build your a package of your service for ConductR:

```
sbt> hello-lagom-impl/bundle:dist
```