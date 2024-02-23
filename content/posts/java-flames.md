+++ 
draft = false
date = 2024-02-22T21:12:18+08:00
title = "Debugging Slow Parallel Stream (Java)"
description = ""
slug = ""
authors = []
tags = ["writeup", "linux", "concurrency"]
categories = []
externalLink = ""
series = []
+++

# Parallel Monte Carlo Method

When I was teaching CS2030S, there was a question about converting a for loop into a parallel stream in Java.

```java
  public static double estimatePi(int numOfPoints) {
    Circle c = new Circle(new Point(0.5, 0.5), 0.5);
    int n = 0;

    /* use stream instead! */
    for (int i = 0; i < numOfPoints; i++) { 
      Point p = new RandomPoint(0, 1, 0, 1);
      if (c.contains(p)) {
        n++;
      }
    }
    return 4.0 * n / numOfPoints;
  }
```

This program estimates the value of Pi using the Monte Carlo method.
> Imagine we have a square with length 2r, and within it, a circle with a radius of r.
Suppose that we randomly generate k points within the square, and then we count how many points fall inside the circle. 
Suppose n points out of k fall within the circle.
Since the area of the square is 4\*r\*r and the area of the circle is pi\*r\*r, the ratio between them is pi/4. The ratio n/k should therefore be pi/4, and pi can be estimated as 4n/k.

The (parallel) stream version is simply as follows:
```java
public static double estimatePiStream(int numOfPoints) {
  Circle c = new Circle(new Point(0.5, 0.5), 0.5);
  long n = Stream
    .generate(() -> new RandomPoint(0, 1, 0, 1))
    .limit(numOfPoints)
    .filter(p -> c.contains(p))
    .parallel() // make multiple streams to run in parallel
    .count();
  return 4.0 * n / numOfPoints;
}
```

At first glance, I thought that the parallel version will be much faster than the for loop version. After all, each stream can work independently and no synchronisation is needed between them. Theoretically, we should be able to get a speedup close to the number of processors currently available on the machine.

Let's run the full code and see if we can gain any speedup:
```java
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.ArrayList;
import java.util.Random;
import java.util.stream.Stream;

class Point {
  private double x;
  private double y;
  public Point(double x, double y) {
    this.x = x;
    this.y = y;
  }
  public double distSquare(Point p) {
    return (this.x - p.x) * (this.x - p.x) + (this.y - p.y) * (this.y - p.y);
  }
}

class RandomPoint extends Point {
  private static Random rand = new Random(0);
  public RandomPoint(double minX, double maxX, double minY, double maxY) {
    super(rand.nextDouble() * (maxX - minX) + minX,
          rand.nextDouble() * (maxY - minY) + minY);
  }
}

class Circle {
  private Point c;
  private double r;
  public Circle(Point c, double r) {
    this.c = c;
    this.r = r;
  }
  public boolean contains(Point p) {
    return c.distSquare(p) <= (r*r);
  }
}

class EstimatePi {
  public static double estimatePi(int numOfPoints) {
    Circle c = new Circle(new Point(0.5, 0.5), 0.5);
    int n = 0;
    for (int i = 0; i < numOfPoints; i++) {
      Point p = new RandomPoint(0, 1, 0, 1);
      if (c.contains(p)) {
        n++;
      }
    }
    return 4.0 * n / numOfPoints;
  }

  public static double estimatePiStream(int numOfPoints) {
    Circle c = new Circle(new Point(0.5, 0.5), 0.5);
    long n = Stream
      .generate(() -> new RandomPoint(0, 1, 0, 1))
      .limit(numOfPoints)
      .filter(p -> c.contains(p))
      // .parallel()
      .count();
    return 4.0 * n / numOfPoints;
  }

  public static double estimatePiParallelStream(int numOfPoints) {
    Circle c = new Circle(new Point(0.5, 0.5), 0.5);
    long n = Stream
      .generate(() -> new RandomPoint(0, 1, 0, 1))
      .limit(numOfPoints)
      .parallel()
      .filter(p -> c.contains(p))
      .count();
    return 4.0 * n / numOfPoints;
  }

  public static void main(String[] args) {
    final int NUM_POINTS = 5_000_000;

    Instant start;
    Instant stop;
    double res;

    /* warm up the caches */
    for (int i = 0; i < 5; ++i) {
      res = estimatePi(NUM_POINTS);
      res = estimatePiStream(NUM_POINTS);
      res = estimatePiParallelStream(NUM_POINTS);
    }

    System.out.println("=== for loop ===");
    start = Instant.now();
    res = estimatePi(NUM_POINTS);
    stop = Instant.now();
    System.out.println("res : " + res);
    System.out.println("time: " + Duration.between(start, stop).toMillis() + " ms");

    System.out.println("=== stream ===");
    start = Instant.now();
    res = estimatePiStream(NUM_POINTS);
    stop = Instant.now();
    System.out.println("res : " + res);
    System.out.println("time: " + Duration.between(start, stop).toMillis() + " ms");

    System.out.println("=== parallel stream ===");
    start = Instant.now();
    res = estimatePiParallelStream(NUM_POINTS);
    stop = Instant.now();
    System.out.println("res : " + res);
    System.out.println("time: " + Duration.between(start, stop).toMillis() + " ms");
  }
}
```

Running the code with k = 5000:
```bash
$ java EstimatePI
=== for loop ===
res : 3.1552
time: 2 ms
=== stream ===
res : 3.1552
time: 9 ms
=== parallel stream ===
res : 3.136
time: 13 ms  
```

Hmm, okay. Since the workload is quite light, maybe the overhead of creating multiple threads overwhelms the potential gain. Let's try that again using k = 5,000,000.
```bash
$ java EstimatePI
=== for loop ===
res : 3.1416976
time: 61 ms
=== stream ===
res : 3.1416976
time: 96 ms
=== parallel stream ===
res : 3.1408872
time: 3304 ms 
```

Okay, I guess the parallel version is just slower. But why?

## Profiling using Lightweight Java Profiler + Flamegraph
Instead of just randomly guessing, let's just profile our code and see what is happening during runtime. [Following Brendan Gregg's tutorial](https://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html), we will be using the [lightweight-java-profiler](https://code.google.com/archive/p/lightweight-java-profiler/wikis/GettingStarted.wiki) and visualise the result using [FlameGraph](https://github.com/brendangregg/FlameGraph).

```bash
$ wget https://storage.googleapis.com/google-code-archive-source/v2/code.google.com/lightweight-java-profiler/source-archive.zip
$ unzip source-archive.zip
$ cd lightweight-java-profiler
```
We need to make some changes and set the architecture to 64 bits:
```bash
4c4
< BITS?=32
---
> BITS?=64
49c49
< INCLUDES=-I$(JAVA_HOME)/$(HEADERS) -I$(JAVA_HOME)/$(HEADERS)/$(UNAME)
---
> INCLUDES=-I$(JAVA_HOME)/$(HEADERS) -I$(JAVA_HOME)/$(HEADERS)/$(UNAME) -I/usr/include/x86_64-linux-gnu
```

After we are done, we can start building the binary and use it when we run our pi estimator.

```bash
$ make all
$ java -agentpath:/tmp/lightweight-java-profiler/trunk/build-64/liblagent.so EstimatePi
```

The report will be written to `traces.txt`, which can be read and parsed to by FlameGraph.
```bash
$ git clone http://github.com/brendangregg/FlameGraph
$ cd FlameGraph
$ ./stackcollapse-ljp.awk < ../traces.txt | ./flamegraph.pl > ../traces.svg
```

Let's see what happens when we run the for parallel stream version:

![parallel](/images/java-flames/parallel.svg)

Oh? 

We spent most of our time calling `java.util.Random.next`. Ermm, okay... Let's consult the docs and see what actually happens when you call `next()`.
> ... The method next is implemented by class Random by **atomically** updating the seed to (seed * 0x5DEECE66DL + 0xBL) & ((1L << 48) - 1) and returning (int)(seed >>> (48 - bits)).

[source](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Random.html#next(int))

I see. It seems that the `Random` object maintains an internal state and the `next()` method updates it atomically. No wonder there's contention.

In that case, let's try to create a new `Random` object in each iteration. See if that solves the issue.
```java
  public RandomPoint(double minX, double maxX, double minY, double maxY) {
    super(new Random().nextDouble() * (maxX - minX) + minX,
          new Random().nextDouble() * (maxY - minY) + minY);
  }
```

```bash
➜  java EstimatePi
=== for loop ===
res : 3.1414016
time: 314 ms
=== stream ===
res : 3.1421856
time: 356 ms
=== parallel stream ===
res : 3.141012
time: 1115 ms  
```

The performance of the normal for loop and single-threaded stream degraded, but that's to be expected. After all, we are creating a new object on each iteration. We see some improvement on the parallel version, but seems that it's still a bit slow. Let's profile it once more and see what's happening.

![parallel_new](/images/java-flames/parallel-newRandom.svg)

Uhh, what the hell is `seedUniquifier`? Must be a private method in the library, since it's not documented anywhere.

After searching here and there, I finally found jdk's github repo and managed to find the method in question.

https://github.com/openjdk/jdk17/blob/4afbcaf55383ec2f5da53282a1547bac3d099e9d/src/java.base/share/classes/java/util/Random.java#L112

```java
    /**
     * Creates a new random number generator. This constructor sets
     * the seed of the random number generator to a value very likely
     * to be distinct from any other invocation of this constructor.
     */
    public Random() {
        this(seedUniquifier() ^ System.nanoTime());
    }

    private static long seedUniquifier() {
        // L'Ecuyer, "Tables of Linear Congruential Generators of
        // Different Sizes and Good Lattice Structure", 1999
        for (;;) {
            long current = seedUniquifier.get();
            long next = current * 1181783497276652981L;
            if (seedUniquifier.compareAndSet(current, next))
                return next;
        }
    }

    private static final AtomicLong seedUniquifier
            = new AtomicLong(8682522807148012L);
```

Right. We are updating a static atomic variable each time a new instance is created. I guess there's no way we can make the parallel version faster, unless we create all the random points beforehand.

Time measurement for each methods, iterating through a pre-allocated `ArrayList` of `RandomPoint`:
```java
$ java EstimatePi
=== for loop ===
res : 3.14144152
time: 116 ms
=== stream ===
res : 3.14144152
time: 180 ms
=== parallel stream ===
res : 3.14144152
time: 54 ms  
```

## Conclusion
Thanks for staying till the end. Remember, when in doubt, profile and read the docs!