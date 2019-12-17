# Evaluating Throttling Implementations

## Context

[`gst-plugin-threadshare`] is a framework and a collection of `GStreamer`
plugins which are developped with the Rust programming language and use the
[`gstreamer-rs`] bindings to the [`GStreamer`] C-based libraries.

The framework provides an execution evironment that allows plugins to share
threads instead of spawning dedicated threads as they would do in the regular
`GStreamer` model. This approach reduces context switches and system calls,
thus lowering CPU load.

The framework executor uses the crate [`tokio`]. Until release `0.2.0`, `tokio`
featured functions that gave control on the iterations of the executor:

  - `turn` would execute one iteration and indicate if tasks were handled by
    the executor.
  - The user could request the executor to `park` & `unpark`.

Thanks to these features the framework could benefit from the `tokio` executor
while implementing a throttling strategy. The throttling strategy consists in
grouping tasks and I/O pollings by forcing the thread to sleep during a short
duration. The benefit are a more efficient use of the CPU and a wider bandwidth.

Since the throttling strategy introduces pauses in the execution thread, timers
need a special treatment so that they are fired close enough to their target
instant. In order to avoid timers from being triggered too late,
`gst-plugin-threadshare` implements its own timers management.

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

  1. Assess the impact of recent changes in `gst-plugin-threadshare`.
  2. Support throttling as an approach to reduce CPU load for `tokio`'s executor.

## Conditions

The evaluation was conducted on the most powerful setup I could find, which is
not that powerful unfortunately:

  - CPU: Intel Core i5 - 7200U @ 2.5GHz, 2 physical cores, SMT disabled.
  - OS: Fedora 31 up to date as of 2019/12/16.
  - Kernel: `Linux 5.3.15-300.fc31.x86_64`.
  - Scaling governor: `performance`.
  - Max. file descriptors: 30,000 (`ulimit -Sn 30000`).
  - RAM: 12GB DDR4 @ 2,400 MHz.

Since the CPU provides 2 physical cores only, all tests were performed using
1 thread.

CPU load is evaluated using `top -d 20`, meaning that `top` averages
measurements on a 20s period. Measurements are taken at least one minute after
the benchmark actually starts processing data. All measurements were performed
after at least one hour after the machine was turned on with CPU loaded.

In some cases, even the 20s average was not enough to flatten variations,
in which cases a best effort average was performed. It would be interesting to
invest time in a better evaluation and provide results which would account for
the dispersion, but for the time being this method seemed good enough.

It would be interesting to conduct the tests on a more powerful setup so as to
get a better idea of the actual throttling duration that could be used on the
target machines.

## Comparison of `gst-plugin-threadshare` versions

In order to benefit from the changes in the `Future` / `async` Rust ecosystem,
`gst-plugin-threadshare` has gone through different phases corresponding to
either `tokio` version upgrades or framework reworks. In this evaluation, we
will compare the following versions which should allow assessing the impact of
the different phases:

| Name                | tokio         | Futures        | Framework         | Repo. |
| ------------------- | ------------- | -------------- | ----------------- | ----- |
| 0.1 (master)        | 0.1.22        | 0.1            | Initial model     | [link](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/f638b0eef7155e9bf40fb315284920b3d5eb034e) | 
| 0.2.0.alpha.6 prev. | 0.2.0.alpha.6 | 0.3.0-alpha.19 | Initial model     | [link](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/908fa74132dc7935e40a55473ab24298a066c259) |
| 0.2.0.alpha.6 Pad   | 0.2.0.alpha.6 | 0.3.0-alpha.19 | Pad wrapper model | [link](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/80a01f4754a0ec7d1b4756d7755fcdeff6d50928) |
| 0.2.2 + throttling  | 0.2.2 + thrtl | 0.3.1          | Pad wrapper model | [link](https://gitlab.freedesktop.org/fengalin/gst-plugins-rs/tree/acce02073550ec4e7f065e48e13511ee1cc6db26) |

The Pad wrapper model introduces an API similar to `GStreamer`'s [`GstPad`] type,
except that it uses `Future`s whenever possible.

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

### Notable consequence of the upgrade to `tokio-0.2.x`

Before `tokio-0.2.0`, the `UdpSocket` was built from a `Handle` to the I/O
`Reactor` (see [this code](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/blob/master/gst-plugin-threadshare/src/udpsrc.rs#L574)).
Since `tokio-0.2.0`, the `Reactor` is no longer passed as an argument, however
the `UdpSocket` must be created in the thread where the executor is running.
In order to achieve this, a `Future` is spawned for the `UdpSocket` creation and
we wait for the `Future` completion before continuing with the initialization.
(see [this code](https://gitlab.freedesktop.org/fengalin/gst-plugins-rs/blob/acce02073550ec4e7f065e48e13511ee1cc6db26/gst-plugin-threadshare/src/udpsrc.rs#L639)).

In the benchmark receiver, all the elements for all the streams are added to the
same `GStreamer` `pipeline`. This is convenient because all the elements can
be started from one instruction and only one bus is needed to monitor errors.
Each `ts-udpsrc` element creates its own `UdpSocket` by spawning a `Future` on
the throttling executor. As a consequence, starting the `0.2.2 + throttling`
benchmark with 4,000 streams and a throttling duration of 50ms takes about 200s
for `ts-udpsrc`.

### Results

FIXME figure out why the sweet spot is not around 20ms.

The following plots show the CPU load induced by the `gst-plugin-threadshare`
versions at different throttling duration depending on the streams number.

Note the dots at throttling duration 0. They represent the CPU load when no
throttling is applied. For the `0.2.2 + throttling` version, the regular `tokio`
`BasicScheduler` implementation is used, for the other versions the thread is
not forced to sleep.

The black line indicates the CPU load for the `udpsrc` element, so this line is
the threshold under which throttling actually reduces CPU load compare to the
element threaded implementation.

![`gst-plugin-threadshare` 1,500 streams](plots/gst-plugin-threadshare/cpu_load_1500.png?raw=true "`gst-plugin-threadshare` 1,500 streams")

At 1,500 streams, `0.1 (master)` reaches the threshold with a shorter throttling
duration than `0.2.2 + throttling`. However, as throttling duration increases,
CPU load tends to fluctuate (more than a 10% in relative value), even for a
given throttling duration (not represented here). `0.2.2 + throttling` shows a
more predictable behavior both as throttling duration increases and for a given
operating point.

The upgrade to `tokio-0.2.0.alpha.6` induces a slight increase in CPU load,
but quite insignificant when compared to the 10% increase (in relative value)
due to the `Pad` wrapper model as `0.2.0.alpha.6 Pad` seems to show.
The throttling implemented on `tokio-0.2.2` compensates the overhead at the
interesting operating points though.

![`gst-plugin-threadshare` 2,000 streams](plots/gst-plugin-threadshare/cpu_load_2000.png?raw=true "`gst-plugin-threadshare` 2,000 streams")

At 2,000 streams, `0.2.2 + throttling` reaches the threshold faster than any
other implementations.

The upgrade to `tokio-0.2.0.alpha.6` shows a more significant increase in CPU
load compared to the 1,500 streams test.

`0.2.0.alpha.6 Pad` overhead seems to prevent this version from taking advantage
of the throttling strategy.

![`gst-plugin-threadshare` 2,500 streams](plots/gst-plugin-threadshare/cpu_load_2500.png?raw=true "`gst-plugin-threadshare` 2,500 streams")

At 2,500 streams, only `0.2.2 + throttling` is able to benefit from the
throttling strategy on this configuration.

## Testing `tokio` throttling

`tokio-udp-receiver-benchmark` was created to demonstrate the benefit of the
throttling strategy on `tokio` without the `GStreamer` overhead. This project
uses the same approach as the tests above: a set of UDP sender & receiver with
control on the streams number and throttling duration.

The tests were conducted on [`tokio` `0.2.2 + throttling`] branch with a
[modified version of `tokio-udp-receiver-benchmark`]. The modified version
prints the throughput reached by a single port. Only one port was used in order
to avoid the need for synchronization primitives on a shared counter. The counter
is incremented upon receipt of a new packet during 20s after which the throughput
is printed on screen.

### Results

The following plot shows the CPU load induced by the non-throttling and
throttling `BasicScheduler` of [`tokio` `0.2.2 + throttling`].

![`tokio` only CPU load](plots/tokio/tokio_only_cpu_load_throuput_cpu_load.png?raw=true "`tokio` only CPU load")

With this tests, the benefit on CPU load of throttling appears immediately. But,
it would be interesting to check that the throughput doesn't suffer from the
thread forced pauses.

![`tokio` only throughput](plots/tokio/tokio_only_throuput_throughput.png?raw=true "`tokio` only throughput")

The sender emits one buffer on each stream every 20ms. This means that one port
of the receiver can receive at most 50 buffers per second.

Despite the reduced CPU load, the throttling version can handle significantly
more streams than the non-throttling version before the throughput starts to
decline.

## Next steps

The following next steps are identified:

  - Cleanup the implementation of [`tokio` `0.2.2 + throttling`]
    and rebase on master.
  - Propose [`tokio` `0.2.2 + throttling`] for inclusion in upstream `tokio`.
  - Perform similar tests on `gst-plugin-threadshare` with additional
    `{ts-}Queue`s in the pipeline so that the evaluation doesn't rely mostly
    on I/O.
  - Check the benefit of using `lto` (link time optimization) when building
    tools.
  - Analyze and reduce the overhead induced by the `Pad` wrapper model in
    `gst-plugin-threadshare`.

[`gst-plugin-threadshare`]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/tree/master/gst-plugin-threadshare
[`gstreamer-rs`]: https://gitlab.freedesktop.org/gstreamer/gstreamer-rs
[`GStreamer`]: https://gstreamer.freedesktop.org/
[`tokio`]: https://github.com/tokio-rs/tokio
[`GstPad`]: https://gstreamer.freedesktop.org/documentation/gstreamer/gstpad.html?gi-language=c
[`tokio` `0.2.2 + throttling`]: https://github.com/fengalin/tokio/tree/330dd3a55d4d09548492ebe2b5fdd473b4842cb2
[`tokio-udp-receiver-benchmark`]: https://github.com/sdroege/tokio-udp-receiver-benchmark
[modified version of `tokio-udp-receiver-benchmark`]: https://github.com/fengalin/tokio-udp-receiver-benchmark/tree/5e3a6ba2d81dda6a0f470eee1c46d380f6bf1a50
