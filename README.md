mesos-distcc: distcc on Mesos.
==============================

Usage:

```
mesos-distcc [pump] make -jN CC='distcc gcc' CXX='distcc g++'
```

Alternatively:

```
./configure CC='distcc gcc' CXX='distcc g++'
mesos-distcc [pump] make -jN
```

How does this work?
-------------------
`mesos-distcc` will introspect the `make` command to determine the number of `distccd` workers that need to be run to acheive the desired build parallelism.

Then, the `DistCCScheduler` is started and connects to a Mesos master specified in your environment or your `mesos-distcc.rc` file, it will attempt to run distccd servers on Mesos. Once these are running, the make command is run with `DISTCC_HOSTS` and `DISTCC_POTENTIAL_HOSTS` set in its environment.

Once compilation is complete, the scheduler tears down the temporary `distccd` cluster.

Installation
------------
* Install distcc on your build machine and on all slave machines.
* Make the Python Mesos eggs available on your build machine and ensure your `PYTHONPATH` is pointing to them (the easiest way to do this is by running `make` in the Mesos project). As an example:

```
PYTHONPATH=/home/bmahler/mesos-0.17.0/build/src/python/dist/mesos-0.17.0-py2.6-linux-x86_64.egg:/home/bmahler/mesos-0.17.0/build/3rdparty/libprocess/3rdparty/protobuf-2.4.1/python/dist/protobuf-2.4.1-py2.6.egg
```

* Modify the `mesos-distcc.rc` file to point to the Mesos master and the location of the distccd binary on your slave machines. Alternatively, these can be set in your environment as `MESOS_MASTER` and `DISTCCD_PATH`.
* Now you can run mesos-distcc:

```
./configure CC='distcc gcc' CXX='distcc g++'
./mesos-distcc make -j200
```


FAQ:
----
#### Q. mesos-distcc fails with ImportError: No module named mesos.

You'll need to ensure your `PYTHONPATH` is pointing to the Mesos eggs, for example:

```
PYTHONPATH=/home/bmahler/mesos-0.17.0/build/src/python/dist/mesos-0.17.0-py2.6-linux-x86_64.egg:/home/bmahler/mesos-0.17.0/build/3rdparty/libprocess/3rdparty/protobuf-2.4.1/python/dist/protobuf-2.4.1-py2.6.egg /home/bmahler/mesos-distcc/mesos-distcc make -j200'
```

Unfortunately, `make install` of Mesos does not install these eggs to be detected automatically by Python, see: [MESOS-899](https://issues.apache.org/jira/browse/MESOS-899).

#### Q. Why can't I run a long-lived distcc cluster?

See the TODOs below. Starting a cluster for each make invocation was a simpler first approach, and more elastic (it avoids keeping `distccd` daemons running when not in use). A long-lived distcc cluster would either be statically sized, or would need to introspect into the distcc state to determine how to size the cluster.

One downside of the current approach is that there is no sharing of the `distccd` servers across different builds.

#### Q. Distcc does not seem to be communicating with the distccd servers.
You will need network access from your client machine to the slave machines (specifically, to the ports being offered as slave resources).

TODOs:
------
* Provide a standalone `mesos-distcc` executable (via [PEX](https://gist.github.com/wickman/2371638)?) that does not require the eggs to be available via `PYTHONPATH`.
* Do the right thing for large N in `make -jN`.
* Allow distcc to be downloaded via a URI rather than requiring installation.
* Add the ability to run a distcc cluster without invoking `make`. This would require the ability to query the scheduler for the current set of `distccd` endpoints.

Benchmark:
----------
I ran a quick performance test to observe the benefits for those
without access to server-grade machines. To measure this, I built
the Mesos project itself off of sha de4b104c.

This test was done using distcc-3.2rc1.

On a 4 core machine:

```
$ ../configure
$ time make check -j4 GTEST_FILTER=""
real	13m36.824s
user	43m52.899s
sys		1m46.021s
```

Using a 4 core machine with mesos-distcc with -j200:

```
$ ../configure CC=distcc CXX=distcc
$ time mesos-distcc make check -j200 GTEST_FILTER=""
real	7m01.801s
user	244.640s
sys		32.403s
```

On a 16 core machine:

```
$ ../configure
$ time make check -j16 GTEST_FILTER=""
real	9m31.985s
user	63m40.918s
sys		4m16.105s
```

Using `mesos-distcc` can provide substantial improvements when one
does not have access to server-grade machines with a high core count.

The Mesos project is not wide enough to benefit substantially from
using `distcc`. It appears the build is ~30 jobs wide at the _widest_
point. Also, `pump` mode does not work for Mesos due to:

https://code.google.com/p/distcc/issues/detail?id=40
https://code.google.com/p/distcc/issues/detail?id=16

Projects that are immune to these `distcc` issues would benefit further
when using `pump` mode. See the [distcc documentation](http://distcc.googlecode.com/svn/trunk/doc/web/benchmark.html) for typical project benchmarks.
