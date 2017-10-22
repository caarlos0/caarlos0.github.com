---
layout: post
title: "Docker, Java and cgroups"
---

So, you're running Java microservices in Docker, huh?
Maybe in a Kubernetes cluster? What happens if you don't
set a `Xmx`?

Before all, let's understand how containers limit their
resources.

When you `docker run --memory=10m`, for example, a file
`/sys/fs/cgroup/memory/memory.limit_in_bytes` is created
containing the amount of bytes the contaienr is allowed to
use.

Let's check that:

```console
$ docker run --memory=10m busybox free
             total       used       free     shared    buffers     cached
Mem:       1530936    1068796     462140     168252      32120     879304
-/+ buffers/cache:     157372    1373564
Swap:      1048572        116    1048456
```

WTF? This is not the ram I said it must have, this is
my Docker for Mac memory limit!

In practice, a given process will see more memory
as available than it actually have, which can lead
processes to use more memory than allowed, which will
cause linux have them killed.

We can easily reproduce this:

```java
import java.util.Vector;

public class Main {
	private static final int MB = 1024 * 1024;
	public static void main(String[] args) {
		System.out.println("available processors: " + Runtime.getRuntime().availableProcessors());
		System.out.println("max memory: " + (Runtime.getRuntime().maxMemory() / MB));
		System.out.println("total memory: " + (Runtime.getRuntime().totalMemory() / MB));
		Vector v = new Vector();
		while (true) {
			byte b[] = new byte[1048576];
			v.add(b);
			System.out.println("free memory: " + (Runtime.getRuntime().freeMemory() / MB));
		}
	}
}
```

We can then compile this and run it:

```console
$ javac Main.java
$ java -Xmx10M Main
...
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at Main.main(Main.java:11)
```

OK, we wrote a program that uses all memory, let's put it inside a container:

```Dockerfile
FROM java:openjdk-8-jre-alpine
COPY Main.class /
ENTRYPOINT ["java", "Main"]
```

Build and run it:

```console
$ docker build -t example1 -f Dockerfile.1 .
Sending build context to Docker daemon  92.16kB
Step 1/3 : FROM java:openjdk-8-jre-alpine
 ---> fdc893b19a14
Step 2/3 : COPY Main.class /
 ---> 7fd64ef1012d
Step 3/3 : ENTRYPOINT java Main
 ---> Running in 71e5148c8593
 ---> 756ee8ab6448
Removing intermediate container 71e5148c8593
Successfully built 756ee8ab6448
Successfully tagged example1:latest

$ docker run --memory=10m example1
available processors: 2
max memory: 361
total memory: 23
free memory: 21
free memory: 20
free memory: 19
free memory: 18
free memory: 17
free memory: 16
free memory: 15
free memory: 14
free memory: 13
free memory: 12
free memory: 11
free memory: 10
# container died
```

OK, what happened here?

Poking with `docker inspect`, we can find this piece of information:

```json
{
  "OOMKilled": true
}
```

Hmm, let's try this differently:

```Dockerfile
FROM java:openjdk-8-jre-alpine
COPY Main.class /
ENTRYPOINT ["java", "-Xmx10M", "Main"]
```

Build and run it:


```console
$ docker run --memory=10m example2
available processors: 2
max memory: 9
total memory: 9
free memory: 8
free memory: 7
free memory: 6
free memory: 5
free memory: 4
free memory: 3
free memory: 2
free memory: 1
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at Main.main(Main.java:11)
```

OK, so now we got a heap space error again.

## How does all this affects me?

Well, if you test your apps and set a good memory limit via `-Xmx` flag and
friends, it might not affect you as much.

If you don't, and run your apps in a kubernetes cluster, you might start to
get into trouble.

Since the JVM don't know it is running in a resource-limited container,
it will assume it can use all available memory (which is all the host memory).
This can lead to a lot of OOMKills, which can negatively affect your users.
Since app's warm up usually is an expensive operation, unecessary warm ups
can also hypothetically degrade your cluster's performance.

## How can I fix this?

Well, lucky for us, since JRE 1.8.0_u131 there is a new flag to enable
cgroups limit for the heap: `-XX:+UseCGroupMemoryLimitForHeap`.

Unfortunatelly, there is no official OpenJDK image for update 131+ as the time
of writing, so, let's use the Oracle one:

```Dockerfile
FROM store/oracle/serverjre:8
COPY Main.class /
ENTRYPOINT ["java", "-XX:+UnlockExperimentalVMOptions", "-XX:+UseCGroupMemoryLimitForHeap", "Main"]
```

Build and run it:

```console
$ docker run --memory=10m example3
available processors: 2
max memory: 7
total memory: 5
free memory: 4
free memory: 3
free memory: 2
free memory: 1
free memory: 2
free memory: 1
free memory: 0
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at Main.main(Main.java:11)
```

As you can see, the heap was limited based on the limits passed down to cgroups
by docker! Sweet!

We can also hack around this on old JREs by using the `-XX:MaxRAM` flag.
You can either set a value or get it directly from cgroups:

```sh
java -XX:MaxRAM=`cat /sys/fs/cgroup/memory/memory.limit_in_bytes` -jar app.jar
```

## Summing up

None of this will solve the "i do not know how much memory my app needs" issue,
but can at least prevent repetitive OOMKills.

Limiting resources is important in a cluster because one app with a really
bad behaviour can cause downtimes on other apps. Your chances to have
bad luck on this are increased if you run few replicas of each pod.

You can alert on OOMKills on a kubernetes cluster by using the
upcoming [kube-state-metrics], which will contain a [patch][p1] adding the
`kube_pod_container_status_terminated_reason` metric.

FWIW, such an alert will look like:

```
ALERT TooManyOOMKills
  IF kube_pod_container_status_terminated_reason{reason="OOMKilled"} != 0
  FOR 5m
```

[p1]: https://github.com/kubernetes/kube-state-metrics/pull/276


## Useful links

- https://developers.redhat.com/blog/2017/04/04/openjdk-and-containers/
- https://bugs.openjdk.java.net/browse/JDK-8170888
- https://store.docker.com/images/oracle-serverjre-8
- https://store.docker.com/images/java
- https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gc-ergonomics.html
- https://fabiokung.com/2014/03/13/memory-inside-linux-containers/
- https://github.com/caarlos0/java-docker-cgroups
