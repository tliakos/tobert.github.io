---
id: "2014-04-09-fio-output-explained"
title: "Fio Output Explained"
abstract: "Fio packs a lot of information into its output. This is a section-by-section breakdown of what it's telling you."
tags: ["fio", "benchmark", "documentation"]
pubdate: 2014-04-09T00:00:00Z
draft: false
---

Previously, I blogged about setting up my [benchmarking machine](/post/2014-03-29-benchmarking-disk-latency-setup.html). Now
that it's up and running, I've started exploring the fio benchmarking tool. In doing so, I've had to learn all of the
abbreviations and terms that fio uses for the kinds of latency it measures. I didn't find the documentation for the
default output very helpful, so here's a line-by-line breakdown that I hope is useful to new users. Most of the data
being displayed was generated on a Samsung 840 Pro SSD.

      read : io=10240MB, bw=63317KB/s, iops=15829, runt=165607msec


The first line is pretty easy to read. fio did a total of 10GB of IO at 63.317MB/s for a total of 15829 IOPS (at the
default 4k block size), and ran for 2 minutes and 45 seconds.

The first latency metric you'll see is the 'slat' or submission latency. It is pretty much what it sounds like, meaning
"how long did it take to submit this IO to the kernel for processing?"

        slat (usec): min=3, max=335, avg= 9.73, stdev= 5.76

I originally thought that submission latency would be useless for tuning, but the numbers below changed my mind. 269usec
or 1/4 of a millisecond seems to be noise, but check it out. I haven't tuned anything yet, so I suspect that changing
the scheduler and telling the kernel it's not a rotating device will help.

        slat (usec): min=5, max=68,  avg=26.21, stdev= 5.97 (SAS 7200)
        slat (usec): min=5, max=63,  avg=25.86, stdev= 6.12 (SATA 7200)
        slat (usec): min=3, max=269, avg= 9.78, stdev= 2.85 (SATA SSD)
        slat (usec): min=6, max=66,  avg=27.74, stdev= 6.12 (MDRAID0/SAS)

The next metric is completion latency. This is the time that passes between submission to the kernel and when the IO is
complete, not including submission latency. In older versions of fio, this was the best metric for approximating
application-level latency.

        clat (usec): min=1, max=18600, avg=51.29, stdev=16.79

From what I can see, the 'lat' metric is fairly new. It's not documented in the man page or docs. Looking at the C
code, it seems that this metric starts the moment the IO struct is created in fio and is completed right after clat,
making this the one that best represents what applications will experience.  This is the one that I will graph.

         lat (usec): min=44, max=18627, avg=61.33, stdev=17.91

Completion latency percentiles are fairly self-explanatory and probably the most useful bit of info in the output. I
looked at the source code and this is not slat + clat; it is tracked in its own struct.

The buckets are configurable in the config file. In the terse output, this is 20 fields of %f=%d;%f=%d;... which makes
parsing more fun than it should be.

        clat percentiles (usec):
         |  1.00th=[   42],  5.00th=[   45], 10.00th=[   45], 20.00th=[   46],
         | 30.00th=[   47], 40.00th=[   47], 50.00th=[   49], 60.00th=[   51],
         | 70.00th=[   53], 80.00th=[   56], 90.00th=[   60], 95.00th=[   67],
         | 99.00th=[   78], 99.50th=[   81], 99.90th=[   94], 99.95th=[  101],
         | 99.99th=[  112]

For comparison, here's the same section from a 7200 RPM SAS drive running the exact same load.

        clat percentiles (usec):
         |  1.00th=[ 3952],  5.00th=[ 5792], 10.00th=[ 7200], 20.00th=[ 8896],
         | 30.00th=[10304], 40.00th=[11456], 50.00th=[12608], 60.00th=[13760],
         | 70.00th=[15168], 80.00th=[16768], 90.00th=[18816], 95.00th=[20608],
         | 99.00th=[23424], 99.50th=[24192], 99.90th=[26752], 99.95th=[28032],
         | 99.99th=[30080]

Bandwidth is pretty self-explanatory except for the per= part. The docs say it's meant for testing a single device
with multiple workloads, so you can see how much of the IO was consumed by each process. When fio is run against
multiple devices, as I did for this output, it doesn't provide much meaning but is amusing when SSDs are mixed with
spinning rust.

        bw (KB  /s): min=52536, max=75504, per=67.14%, avg=63316.81, stdev=4057.09

And here's the SAS drive again with 0.36% of the total IO out of 4 devices being tested.

        bw (KB  /s): min=   71, max=  251, per=0.36%, avg=154.84, stdev=18.29

The latency distribution section took me a couple passes to understand. This is one series of metrics. Instead of using
the same units for all three lines, the third line switches to milliseconds to keep the text width under control. Read
the last line as 2000, 4000, 10,000, and 20,000usec and it makes more sense.

As this is a latency distribution, it's saying that 51.41% of requests took less than 50usec, 48.53% took less than
100usec and so on.

        lat (usec) :   2= 0.01%,   4=0.01%,  10=0.01%,   20=0.01%, 50=51.41%
        lat (usec) : 100=48.53%, 250=0.06%, 500=0.01%, 1000=0.01%
        lat (msec) :   2= 0.01%,   4=0.01%,  10=0.01%,   20=0.01%

In case you were thinking of parsing this madness with a quick script, you might want to know that this section will
omit entries and whole lines if there is no data. For example, the SAS drive I've been referencing didn't manage to do
any IO faster than a millisecond, so this is the only line.

        lat (msec) : 4=1.07%, 10=27.04%, 20=65.43%, 50=6.46%, 100=0.01%

Here's the user/system CPU percentages followed by context switches then major and minor [page
faults](http://en.wikipedia.org/wiki/Page_fault).  Since the test is
[configured](https://gist.github.com/tobert/10685735) to use direct IO, there should be very few page faults.

      cpu          : usr=5.32%, sys=21.95%, ctx=2829095, majf=0, minf=21

Fio has an iodepth setting that controls how many IOs it issues to the OS at any given time. This is entirely
application-side, meaning it is not the same thing as the device's IO queue. In this case, iodepth was set to 1 so the
IO depth was always 1 100% of the time.

      IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, &gt;=64=0.0%

These next two are the number of submitted IOs at a time by fio and the number completed at a time. In the case of the
thrashing test used to generate this output, the iodepth is at the default value of 1, so 100% of IOs were submitted 1
at a time placing the results in the 1-4 bucket. Basically these only matter if iodepth is greater than 1.  These will
get much more interesting when I get around to testing the various schedulers.

         submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, &gt;=64=0.0%
         complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, &gt;=64=0.0%

The number of IOs issued. Something is weird here since this was a 50/50 read/write load, so there should have been an
equal number of writes. I suspect having
[unified_rw_reporting](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L380)
enabled is making fio count all IOs as reads.

If you see short IOs in a direct IO test something has probably gone wrong. The reference I found in the
[Linux kernel](https://github.com/torvalds/linux/blob/v3.14/fs/direct-io.c#L1323)
indicates that this happens at EOF and likely end of device.

         issued    : total=r=2621440/w=0/d=0, short=r=0/w=0/d=0

Fio can be [configured](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L904)
 with a latency target, which will cause it to adjust throughput until it can consistently hit the
configured latency target. I haven't messed with this much yet. In time or size-based tests, this line will always look
the same. All four of these values represent the configuration settings
[latency_target](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L904),
[latency_window](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L909),
[latency_percentile](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L913),
and [iodepth](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L680).

         latency   : target=0, window=0, percentile=100.00%, depth=1

fio supports grouping different tests for aggregation. For example, I can have one config for SSDs and HDDs mixed in the
same file, but set up groups to report the IO separately. I'm not doing this for now, but future configs will need this
functionality.

    Run status group 0 (all jobs):

And finally, the total throughput and time. io= indicates the amount of IO done in total. It will be variable for timed
tests and should match the [size](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L421)
parameter for sized tests. aggrb is the aggregate bandwidth across all processes / devices. minb/maxb show
minimum/maximum observed bandwidth. mint/maxt show the shortest & longest times for tests.
Similar to the io= parameter, these should match the
[runtime](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L973) parameter for
time-based tests and will vary in size-based tests.

Since I ran this test with
[unified_rw_reporting](https://github.com/axboe/fio/blob/046395d7ab181288d14737c1d0041e98328f473f/HOWTO#L380) enabled,
we only see a line for MIXED. If it's disabled there will be separate lines for READ and WRITE.

      MIXED: io=12497MB, aggrb=42653KB/s, minb=277KB/s, maxb=41711KB/s, mint=300000msec, maxt=300012msec

Simple, right?