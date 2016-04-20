---
layout: nav
---

[![Codacy Badge](https://api.codacy.com/project/badge/grade/83c6250bd9fc45a98c12c191af710754)](https://www.codacy.com/app/bluestreak/nfsdb)
[![Build Status](https://semaphoreci.com/api/v1/appsicle/questdb/branches/master/badge.svg)](https://semaphoreci.com/appsicle/questdb)
[![Gitter](https://badges.gitter.im/Join Chat.svg)](https://gitter.im/NFSdb/nfsdb?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## What is NFSdb?

NFSdb is a java library that lets you easily persist huge volumes of POJOs on disk with almost zero GC overhead and minimal latency (millions of writes in a second). With NFSdb you can also query these objects and replicate them over the network. Fast. Very, very fast.

---

## Why?

Storing and querying data for Java developer is always pain in the neck. JDBC requires ORM tools, which is always a maintenance nightmare and performance hog. NoSQL databases are better but come with tricky installation and integration procedures and are not maintenance free either. We wanted to create a library that would help us to:

- linear scalability in response to data volume increase.
- throw away boilerplate persistence layer.
- have clean, minimalistic API.
- throw away caching because our database would be fast enough!
- have minimal heap footprint.
- leverage all of the available memory without using it for heap.
- handle time series queries efficiently.
- provide out of box support for temporal data.
- scale processing out to multiple servers

---

## How?

NFSdb provides automatic serialization for primitive types of POJOs to Memory Mapped Files. Files organised on disk in directories per class and files per attributes, providing column-based data store. String values can be indexed for fast searches and if your data has timestamp - it can be partitioned by DAY, MONTH or YEAR. Memory Mapped Files are managed by Operating System, which provides resilience in case of JVM crash and also a way of inter-process communication as data written by one process is immediately available to all other processes.

NFSdb also provides easy to setup data replication over TCP/IP with automatic service discovery via multicast.

This is an example of fully functional data _publisher/server_:

```java
public class SimpleReplicationServerMain {

    private final String location;

    public SimpleReplicationServerMain(String location) {
        this.location = location;
    }

    public static void main(String[] args) throws Exception {
        new SimpleReplicationServerMain(args[0]).start();
    }

    public void start() throws Exception {
        JournalFactory factory = new JournalFactory(location);
        JournalServer server = new JournalServer(factory);

        JournalWriter<Price> writer = factory.writer(Price.class);
        server.publish(writer);

        server.start();

        System.out.print("Publishing: ");
        for (int i = 0; i < 10; i++) {
            publishPrice(writer, 1000000);
            Thread.sleep(TimeUnit.SECONDS.toMillis(2));
            System.out.print(".");
        }
        System.out.println(" [Done]");
    }

    private void publishPrice(JournalWriter<Price> writer, int count) 
            throws JournalException {
        long tZero = System.currentTimeMillis();
        Price p = new Price();
        for (int i = 0; i < count; i++) {
            p.setTimestamp(tZero + i);
            p.setSym(String.valueOf(i % 20));
            p.setPrice(i * 1.04598 + i);
            writer.append(p);
        }
        // commit triggers network publishing
        writer.commit();
    }
}
```

And this is fully functional _client_ measuring transaction latency:

```java
public class SimpleReplicationClientMain {
    public static void main(String[] args) throws Exception {
        JournalFactory factory = new JournalFactory(args[0]);
        final JournalClient client = new JournalClient(factory);

        final Journal<Price> reader 
                         = factory.bulkReader(Price.class, "price-cpy");

        client.subscribe(Price.class, null, "price-cpy", new TxListener() {
            @Override
            public void onCommit() {
                int count = 0;
                long t = 0;
                for (Price p : reader.incrementBuffered()) {
                    if (count == 0) {
                        t = p.getTimestamp();
                    }
                    count++;
                }
                System.out.println("took: "
                                + (System.currentTimeMillis() - t) 
                                + ", count=" + count);
            }
        });
        client.start();
        System.out.println("Client started");
    }
}
```
---

### More examples?

We have growing collection of examples in our [git repository](https://github.com/bluestreak01/questdb/tree/master/examples/src/main/java/org/nfsdb/examples).

---

### Performance

On test rig (Intel i7-920 @ 4Ghz) NFSdb shows average read latency of 20-30ns and write latency of 60ns per column of data. Read and write do not have any GC overhead (excluding reading strings - they are immutable).

Above example takes ~500ms to produce and fully consume 1 million objects on localhost.

---

![YourKit Logo](http://www.questdb.org/images/yklogo.png) 

This project is supported by YourKit LLC, creator of powerful [YourKit Java Profiler](http://www.yourkit.com/java/profiler/index.jsp) and [YourKit .NET profiler](http://www.yourkit.com/.net/profiler/index.jsp). It would not be possible to make NFSdb as fast as it is without YourKit.

---

## License

Latest release of NFSdb is available under [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0.txt)

The following releases will be available under [GNU Affero GPLv3](https://github.com/bluestreak01/questdb/blob/master/LICENSE.txt)

---

## Support

We actively respond to all [issues](https://github.com/bluestreak01/questdb/issues) raised via GitHub. Please do not hesitate to ask questions or request features.
