# Evaluating Throttling Implementations

## Context

[`gst-plugin-threadshare`] is a framework and a collection of `GStreamer`
elements which are developped with the Rust programming language and use the
[`gstreamer-rs`] bindings to the [`GStreamer`] C-based libraries.

In the regular `GStreamer` model, pipeline elements spawn there own threads to
deal with asynchronous processing. Most of the time, these threads wait for
something to happen which causes multiple overheads. The [`gst-plugin-threadshare`]
model allows elements to share threads which reduces context switches and system
calls, thus lowering CPU load.

The framework executor uses the crate [`tokio`]. Until release `0.2.0`, `tokio`
featured functions that gave control on the iterations of the executor:

  - `turn` would execute one iteration and indicate if tasks were handled by
    the executor.
  - The user could request the executor to `park` & `unpark`.

Thanks to these features the framework could benefit from the `tokio` executor
while implementing a throttling strategy. The throttling strategy consists in
grouping tasks, timers and I/O handling by forcing the thread to sleep during a
short duration. For use cases where a large number of events are to be handled
randomly (e.g. 2,000+ live multimedia streams), this induces an even more
efficient use of the CPU and wider bandwidth.

Since the throttling strategy introduces pauses in the execution thread, timers
need a special treatment so that they are fired close enough to their target
instant. In order to avoid timers from being triggered too late,
`gst-plugin-threadshare` implemented its own timers management.

See [this blog post](https://coaxion.net/blog/2018/04/improving-gstreamer-performance-on-a-high-number-of-network-streams-by-sharing-threads-between-elements-with-rusts-tokio-crate)
for a more in-depth explanation of the throttling strategy.

Two solutions are considered for `gst-plugin-threadshare` to keep up with
`tokio` new versions (see [this thread](https://github.com/tokio-rs/tokio/issues/1887)):

  1. Introduce the throttling strategy as an option in `tokio`'s executor.
  2. Re-introduce an API in `tokio`'s executor so that `gst-plugin-threadshare`
     can keep its throttling strategy.

Solution (1) presents the following benefits:

  - Other projects could activate the throttling strategy depending on their
    use cases.
  - The timers special handling with regard to throttling would be implemented
    in `tokio`, meaning `gst-plugin-threadshare` would not implement its own
    timers management.

This evaluation reports measurements that were conducted while testing
implementations for solution (1).

## Goal

The goal of this evaluation is twofold:

  1. Support throttling as an approach to reduce CPU load for `tokio`'s executor.
  2. Assess the impact of recent changes in `gst-plugin-threadshare`.

## Testing `tokio` throttling

The evaluation was conducted on a machine with the following characteristics:

  - CPU: Intel Core i5 - 7200U @ 2.5GHz, 2 physical cores, SMT disabled.
  - OS: Fedora 31 up to date as of 2019/12/17.
  - Kernel: Linux `5.3.16-300.fc31.x86_64`.
  - Scaling governor: `performance`.
  - Max. file descriptors: 30,000 (`ulimit -Sn 30000`).
  - rustc stable 1.39.0.
  - RAM: 12GB DDR4 @ 2,133 MHz.

Since the CPU provides 2 physical cores only, all tests were performed using
1 thread.

CPU load is evaluated using `top -d 20`, meaning that `top` averages
measurements on a 20s period. Measurements are taken at least one minute after
the benchmark actually starts processing data. All measurements were performed
after at least one hour after the machine was turned on with CPU loaded.

[`tokio-udp-receiver-benchmark`] was created to demonstrate the benefit of the
throttling strategy on `tokio` without `GStreamer` overhead. This project uses a
set of UDP sender & receiver with control on the streams number and throttling
duration.

The tests were conducted on [`tokio` `0.2.5 + throttling`] branch with a
[modified version of `tokio-udp-receiver-benchmark`]. The modified version
prints the throughput reached by a single port. Only one port was used in order
to avoid the need for synchronization primitives on a shared counter.
The counter is incremented upon receipt of a new packet during 20s after which
the throughput is printed on screen.

All executables are built before-hand in `release` version.

### Results

The following plot shows the CPU load induced by the non-throttling and
throttling `BasicScheduler` of [`tokio` `0.2.5 + throttling`].

![`tokio` only CPU load](plots/tokio/tokio_only_cpu_load_throuput_cpu_load.png?raw=true "`tokio` only CPU load")

With this tests, the benefit on CPU load of throttling appears immediately.
Note that increasing the throttling duration reduces CPU load, but also adds
latency.

![`tokio` only throughput](plots/tokio/tokio_only_throuput_throughput.png?raw=true "`tokio` only throughput")

The sender emits one buffer on each stream every 20ms. This means that one port
of the receiver can receive at most 50 buffers per second.

Despite the reduced CPU load, the throttling version can handle significantly
more streams than the non-throttling version before the bandwidth starts
narrowing.

## Comparison of `gst-plugin-threadshare` versions

In order to benefit from the changes in the `Future` / `async` Rust ecosystem,
`gst-plugin-threadshare` has gone through different phases corresponding to
either `tokio` version upgrades or framework reworks. In this evaluation, we
will compare the following versions which should allow assessing the impact of
the different phases:

| Name                | tokio         | Futures        | Repo. |
| ------------------- | ------------- | -------------- | ----- |
| 0.1.x intial design | 0.1.22        | 0.1            | [link](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/f638b0eef7155e9bf40fb315284920b3d5eb034e) | 
| 0.2.0.alpha.6 Pad   | 0.2.0.alpha.6 | 0.3.0-alpha.19 | [link](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/80a01f4754a0ec7d1b4756d7755fcdeff6d50928) |
| 0.2.5 + throttling  | 0.2.5 + thrtl | 0.3.1          | [link](https://gitlab.freedesktop.org/fengalin/gst-plugins-rs/tree/0221524a1091cfea688f95805e37ad71b7f4778d) |

The Pad wrapper design used in `0.2.0.alpha.6 Pad` & `0.2.5 + throttling`
introduces an API similar to `GStreamer`'s [`GstPad`] type, except that it uses
`Future`s whenever possible.

The evaluation was conducted on a machine with the following characteristics:

  - CPU: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz (+ all the microcode updates).
  - Kernel: Linux 5.3.9 (Debian unstable).
  - rustc stable 1.39.0.
  - Memory: 4x8GB DDR3 @ 1,600MHz.

### Tools

`gst-plugin-threadshare` features benchmark tools to compare the behavior of a
threadshare-based implementation of a UDP client source element to its regular
`GStreamer` model counterpart. The benchmark consists in a sender and a receiver.
The user can configure the receiver with the following parameters:

| Parameter  | Description              | Value                   | Effect           |
| ---------- | ------------------------ | ----------------------- | ---------------- |
| Source     | Implementation variant   | `ts-udpsrc` or `udpsrc` |                  | 
| Streams    | UDP streams to handle    | number (0..)            |                  |
| Groups     | Threads to spawn         | number (0..)            | `ts-udpsrc` only |
| Throttling | Max. throttling duration | duration (ms)           | `ts-udpsrc` only |

When using e.g. 2 groups, the streams are distributed on 2 threads, which means
that the load can be balanced on 2 CPU cores.

Example (last parameter is not used):

```
benchmark 2000 ts-udpsrc 1 20 X
```

The sender sends one buffer on each configured streams every 20ms. The user can
configure the sender with the number of UDP streams to send:

```
udpsrc-benchmark-sender 2000
```

All executables are built before-hand in `release` version.

### Results

The following plots show the CPU load induced by the `gst-plugin-threadshare`
versions with and without throttling. The black line indicates the CPU load for
the non-threadshare element.

![`gst-plugin-threadshare` 2,000 streams](plots/gst-plugin-threadshare/cpu_load_2000.png?raw=true "`gst-plugin-threadshare` 2,000 streams")

At 2,000 streams, all `gst-plugin-threadshare` versions behave approximately the
same, regardless of the `tokio` version used or the framework design.

## Next steps

The following next steps are identified:

  - Propose the throttling modifications for inclusion in upstream `tokio`.
  - Perform similar tests on `gst-plugin-threadshare` with additional
    `{ts-}Queue`s in the pipeline so that the evaluation doesn't rely mostly
    on I/O.
  - Check the benefit of using `lto` (link time optimization) when building
    tools.

[`gst-plugin-threadshare`]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/master/gst-plugin-threadshare
[`gstreamer-rs`]: https://gitlab.freedesktop.org/gstreamer/gstreamer-rs
[`GStreamer`]: https://gstreamer.freedesktop.org/
[`tokio`]: https://github.com/tokio-rs/tokio
[`GstPad`]: https://gstreamer.freedesktop.org/documentation/gstreamer/gstpad.html?gi-language=c
[`tokio` `0.2.5 + throttling`]: https://github.com/fengalin/tokio/tree/fengalin/0.2.5-throttling
[`tokio-udp-receiver-benchmark`]: https://github.com/sdroege/tokio-udp-receiver-benchmark
[modified version of `tokio-udp-receiver-benchmark`]: https://github.com/fengalin/tokio-udp-receiver-benchmark/tree/ccee51f336baf77367a9a4b4e99458289820b45a
