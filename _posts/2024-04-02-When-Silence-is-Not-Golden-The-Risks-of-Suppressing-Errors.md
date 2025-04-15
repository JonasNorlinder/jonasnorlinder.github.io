---
title: "When Silence is Not Golden: The Risks of Suppressing Errors"
layout: post
hidden: true
sitemap: false
redirect_from:
  - /When-Silence-is-Not-Golden-The-Risks-of-Ignoring-Exceptions/
  - /When-Silence-is-Not-Golden-The-Risks-of-Supressing-Exceptions/
  - /When-Silence-is-Not-Golden-The-Risks-of-Supressing-Errors/
  - /When-Silence-is-Not-Golden-The-Risks-of-Ignoring-Errors/
---

[DaCapo](https://github.com/dacapobench/dacapobench/) is a widely used
benchmarking tool in the Java community and the original paper has been cited
over 1,400 times. It was originally released in 2006 and after 14 years, got a new
major release in 2023. The development team deserves credit for their diligent
efforts in revamping DaCapo. However, upon reviewing the benchmarks, I noticed
that none have been verified for correctness through functional testing. While
there are lightweight checksums in place to ensure consistency in output across
individual benchmarks, no automatic analysis is performed to validate the
accuracy of these outputs.

# Examining Output Suppression in Cassandra

Cassandra is a new addition in the latest major release of DaCapo. It is a NoSQL
distributed database used by many, so it is certainly a welcomed addition to the
benchmarking suite. DaCapo uses [YCSB](https://github.com/brianfrankcooper/YCSB)
as the driver for the workload.

In version `v23.11` the large workload uses the following configuration
(omitted intricate details, see full version
[here](https://github.com/dacapobench/dacapobench/blob/v23.11-chopin/benchmarks/bms/cassandra/workload/ycsb/workload-huge).
Yes, the file is called huge but it is indeed [what is configured to run for large](https://github.com/dacapobench/dacapobench/blob/v23.11-chopin/benchmarks/bms/cassandra/cassandra.cnf)):
{% highlight configuration %}
workload=site.ycsb.workloads.TimeSeriesWorkload
operationcount=1179648

readproportion=0.10
insertproportion=0.90
{% endhighlight %}

whereas small and default are configured to use `CoreWorkload`
with vastly different read and insert proportions

{% highlight configuration %}
workload=site.ycsb.workloads.CoreWorkload
operationcount=200000 # default
operationcount=2000   # small

readproportion=0.5
updateproportion=0.5
{% endhighlight %}

The logging configuration of DaCapo's driver for Cassandra sets the default log
level for `org.apache.cassandra` to DEBUG, resulting in a vast amount of logged
information that significantly impacts performance. However, this went unnoticed
since DaCapo suppresses all output from the benchmark. This issue was addressed
in `v23.11-MR1`, released in November 2024, as mentioned in the release notes,
which point to an [issue](https://github.com/dacapobench/dacapobench/issues/272)
that revealed excessive allocation due to debug logging rather than Cassandra
itself.

Upon removing output suppression during iteration and running `java -jar
dacapo-evaluation-git-fd292e92.jar cassandra -s large -n 1 &>
workload_large.log`, I obtained a 1.6 GB log file filled with exceptions.
Notably, `com.datastax.driver.core.exceptions.InvalidQueryException` are thrown
118,206 times, accounting for approximately 10% of all operations, indicating
that nearly all reads failed. Consequently, DaCapo `v23.11` using Cassandra with
a large workload essentially measure the error processing rate of a faulty
application, rather than providing a representative workload. Surprisingly,
neither the release notes nor the linked issue mention this critical problem.

This error might explain why the workload was entirely revamped in `v23.11-MR1` to resemble the small/default configuration:

{% highlight configuration %}
workload=site.ycsb.workloads.CoreWorkload
operationcount=2000000

readproportion=0.5
updateproportion=0.5
{% endhighlight %}

I believe highlighting this issue is crucial, given that published research has utilized `v23.11` with its flawed configuration.

Key takeaways:

* Cassandra with a large workload in `v23.11` are measuring a buggy application
  where all reads fail

* DaCapo "resolved" the issue in `v23.11-MR1` by changing the workload to something completely
  different with vastly different read/insert proportions.

# Examining Exception Suppression in H2

One of the benchmarks offered by DaCapo is H2, a Java-based SQL database. To
evaluate H2's performance, the TPC-C benchmark is utilized. TPC-C is an
industry-standard benchmark specifically designed for assessing the performance
of Online Transaction Processing (OLTP) systems.

While DaCapo relies on Apache's implementation of TPC-C they have modified the
implementation to some degree to fit with their framework.

On a high level the DaCapo driver for H2 looks something like this (I have
simplified the code for readability):

{% highlight java linenos %}
// DaCapo main driver
public void run() {
    while (iterations < MAX_ITERATIONS) {
        final long start = System.currentTimeMillis();
        iterate();
        final long duration = System.currentTimeMillis() - start;
        System.out.print("===== DaCapo completed warmup in "
            + elapsed + " msec");

        iterations++;
    }
}

// DaCapo's H2 driver
public void iterate() {
    Thread[] threads = new Thread[submitters.length];
    for (int i = 0; i < submitters.length; i++) {
      submitters[i].clearTransactionCount();
      threads[i] = newThread(i, submitters[i], transactionsPerTerminal[i]);
    }

    for (int i = 0; i < threads.length; i++) {
      threads[i].start();
    }

    for (int i = 0; i < threads.length; i++) {
      threads[i].join();
    }
}
{% endhighlight %}

Each thread will then run the following method, where count corresponds to the
number of transactions that DaCapo has defined for each workload size: 400
(small), 100000 (default), 2500000 (large).

{% highlight java linenos %}
public void runTransactions(final int count) {
    for (int i = 0; i < count; i++) {
      int txType = getTransactionType();
      boolean success = false;
      while (!success) {
        try {
          success = runTransaction(txType, displayData);
        } catch (Exception e) {
        }
      }
    }
}
{% endhighlight %}

The above method is part of DaCapo's driver and not part of Apache's
implementation of TPC-C. Have you noticed anything unusual? Specifically, if a
transaction fails, we retry it until it succeeds. Any exceptions that occur are
suppressed. As a result, we disregard all attempted transactions and only count
those that complete without throwing an exception. While this approach may be
problematic if many transactions fail, it may be less concerning if failures are
infrequent. It's possible that this behavior aligns with real-world
applications, where occasional failures can occur. Although I'm not familiar
with the intricacies of the TPC-C specification, it's conceivable that it
permits some degree of transaction failure.

I set out to measure how frequent we observe successes and failures for small,
default and large workload.

```plain
                              Large      Default    Small
Total fails:              6 691 994      349 111    2 792
Total successes:          2 500 000      100 000      400
Total transactions:       9 191 994      449 111    3 192
% error of total:                73           78       87
```

We can observe that transactions that fails and throw an exception are between
more than 2.7 to 7 times more common than normal transactions! As a consequence,
73-87% of the transactions are errors and hence the benchmark mostly measures the
rate of error processing of a buggy application, which is not very meaningful.
What's even more concerning is that the proportion of failed transactions
varies, making it impossible to determine the impact of changing workload sizes
without explicit measurement. Unless you specifically track how the workload
characteristics change with different sizes, it will remain unclear.

There are 8 types of different transactions and measured sucess and failures for
each:

![H2 large](/images/dacapoException/large.png)

It is clear that `PAYMENT_BY_NAME` and `PAYMENT_BY_ID` have a really high
failure rate. The above plot depicts the large workload but
[default](/images/dacapoException/default.png) and
[small](/images/dacapoException/small.png) looks very similar.

I believe this investigation into DaCapo's driver of the TPC-C workload for H2
reveals:

* We should verify that the workload behaves as expected, going beyond mere
  checksum validation of the output to ensure its correctness

* Silently suppressing exceptions poses significant risks, as these tend to be
  overlooked over time. It is likely that the driver was correct initially, but
  became faulty at some point during development.

* Unfortunately, my analysis indicates that the current implementation of TPC-C
  in the latest DaCapo release contains errors.
