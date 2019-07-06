Performance comparisons
=======================

I've listed the steps for the following four comparisons below:

1. The CPU governors all set to performance and the GPU governor left at its default value, i.e. `nvhost_podgov`.
2. The CPU governors and the GPU governor all set to performance.
3. The CPU governors left set to performance and the GPU governors set to back to `nvhost_podgov` and the GPU `min_freq` set to its maximum value.
4. The CPU and GPU clocks (and other values) set to maximum with `jetson_clocks`.

If I run through these steps repeatedly (doing the four comparisons and then rebooting the system to start afresh) I get the following median times for each of the above:

1. 4m40.759s
2. 3m56.635s
3. 3m45.352s
4. 3m37.201s

The numbers are pretty much as you'd expect with the slowest performance being when the GPU governor is left as its default value. For 1, 3 and 4. the results are fairly consistent, while 2 shows the most variability - perhaps because this governor setting allow the GPU to run very fast but can also adjust down performance when it gets too hot.

Steps
-----

**A.** Reset all clocks etc. back to their default values by rebooting:

    $ sudo reboot now

**B.** Check the default GPU governor and check the available GPU frequencies:

    $ cat /sys/devices/57000000.gpu/devfreq/57000000.gpu/governor
    nvhost_podgov
    $ cat /sys/devices/57000000.gpu/devfreq/57000000.gpu/available_frequencies
    76800000 153600000 230400000 307200000 384000000 460800000 537600000 614400000 691200000 768000000 844800000 921600000

So `nvhost_podgov` is the default governor and 921.6 MHz is the maximum frequency.

**C.** Check that we're in MAXN mode:

    $ sudo nvpmodel -q
    NV Power Mode: MAXN
    0

**D.** Set all the four CPU governors to performance:

    $ for gov in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    do
        echo performance | sudo tee -a $gov
    done

**E.** Run our **first** test:

    $ cd /usr/src/tensorrt/bin
    $ time sudo ./sample_uff_ssd_rect
    ...
    Begin building engine...
    Time lapsed to create an engine: 247348ms
    ...
    Average time spent per iteration is 28.1992 ms.
    Time taken for inference is 27.9786 ms.
    ...
    real    4m40.103s

So a total run time of 4m 40s and an inference time of 28ms.

**F.** Set the GPU governor to performance:

    $ echo performance | sudo tee -a /sys/devices/57000000.gpu/devfreq/57000000.gpu/governor

**G.** Run our **second** test:

    $ time sudo ./sample_uff_ssd_rect
    ...
    Begin building engine...
    Time lapsed to create an engine: 195383ms
    ...
    Average time spent per iteration is 26.3252 ms.
    Time taken for inference is 26.1633 ms.
    ...
    real    3m44.310s

This time things ran well with the GPU governor set to performance and we got a total run time of 3m 44s and an inference time of 26ms.

**H.** Set the GPU governor back to `nvhost_podgov` and set the `min_freq` to the maximum value:

    $ echo nvhost_podgov | sudo tee -a /sys/devices/57000000.gpu/devfreq/57000000.gpu/governor
    $ echo 921600000 | sudo tee -a /sys/devices/57000000.gpu/devfreq/57000000.gpu/min_freq

**I.** Run our **third** test:

    time sudo ./sample_uff_ssd_rect
    ...
    Begin building engine...
    Time lapsed to create an engine: 196295ms
    ...
    Average time spent per iteration is 26.2475 ms.
    Time taken for inference is 26.2237 ms.
    ...
    real    3m45.130s

So a total run time of 3m 45s and an inference time of 26ms.

**J.** Set all clocks etc. to their maxes with `jetson_clocks`:

    $ sudo jetson_clocks

**K.** Run our **forth** test:

    $ time sudo ./sample_uff_ssd_rect
    ...
    Begin building engine...
    Time lapsed to create an engine: 188786ms
    ...
    Average time spent per iteration is 26.0683 ms.
    Time taken for inference is 26.0673 ms.
    ...
    real    3m37.309s

So a total run time of 3m 37s and an inference time of 26ms.
