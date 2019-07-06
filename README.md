Setting the Jetson Nano GPU governor to performance can be consistently shown to perform terribly
=================================================================================================

You'd expect that setting the GPU governor to performance would consistently result in improved performance compared to the default. And indeed it does _sometimes_ - but I can also consistently demonstrate it having a terrible affect on performance - increasing the run time of a standard GPU benchmark by an order of magnitude.

This is **not** the result of thermal throttling - in fact on the bad runs things run so slowly that the heatsink is noticeably cooler to touch than normal.

I installed the SSD-Mobilenet-V2 benchmark (as described in the Nvidia [deep learning inference benchmarking instructions](https://devtalk.nvidia.com/default/topic/1050377/jetson-nano/deep-learning-inference-benchmarking-instructions/) page) and then ran the benchmark with various governor and clock settings.

The key factor seems to be starting from a just rebooted system. If I reboot the system, set the CPU and GPU governors to performance and then run the SSD-Mobilenet-V2 benchmark, it consistently shows a terrible run time of around 12 minutes. If I rerun the benchmark multiple times it makes no difference - once it has run slowly first time, it consistently runs slowly.

However if I do something slightly different then things perform more as expected. If I rebook the system, set _just_ the **CPU** governors to performance and then run the benchmark, it gives an acceptable time of around 4m 40s. But what's interesting is that if I now set the **GPU** governor to performance and rerun the benchmark then, as expected and desired, I get an improved time. Rather than the terrible time of 12 minutes it runs in about 3m 50s.

So it seems if, after boot, I immediately set the GPU governor to performance then I get terrible times. However if I "exercise" the GPU first and only then set the GPU governor to performance, I get the improved performance I expect.

Notes:

* My system is set not to boot into graphical mode, so the GPU is used purely for benchmark work.
* I want to use the CPU and GPU governors in performance mode, rather than using `jetson_clocks`, as I still want thermal throttling when necessary. With the governors set to performance this is still possible but it is essentially disabled if `jetson_clocks` has been used to prevent all the clocks from going to anything below their the maximum frequency.
* Nvpmodel is set to MAXN.
* I'm running using the recommended micro-USB power supply.

Testing
-------

This all sounds so unlikely that I'm guessing you think I'm making some kind of mistake here - but I can consistently show this behavior and have rechecked my steps each time. I've outlined these steps below and also include links to two scripts on GitHub that I used to repeatedly restart my Nano and run these steps.

The first script sets the CPU and GPU governors to performance and runs the SSD-Mobilenet-V2 benchmark three times (with pauses to avoid any heat related issues) and then reboots the system. I've left it running over night and consistently get a run time of around 12 minutes per run of the benchmark.

The second script sets the CPU governors to performance, runs the SSD-Mobilenet-V2 benchmark once, then additionally sets the GPU governor to performance, runs the benchmark again and then reboots the system. I've left it running over night and consistently get a run time of around 3m 50s for the second run of the benchmark.

If you try these scripts make sure you've just rebooted your Nano before the first run.

Prerequisites
-------------

I conducted all my tests on a Nano that has been setup:

* Not to boot into graphical mode.
* Not to require a password for `sudo`.
* To allow password-less SSH login using SSH keys.

I always used WiFi, rather than wired Ethernet, to connect to the Nano.

**1.** Turn off requiring a password for sudo:

    $ sudo visudo

And change the following line:

    %sudo   ALL=(ALL:ALL) ALL

To:

    %sudo   ALL=(ALL:ALL) NOPASSWD: ALL

**2.** By default the Nano boots into graphical mode:

    $ systemctl get-default
    graphical.target

Disable this as we want to reserve the GPU completely for benchmarking:

    $ sudo systemctl set-default multi-user.target

We want WiFi to start automatically but by default it only starts if someone has logged in (typically via the GNOME display manager). So change this:

    $ cd /etc/NetworkManager/system-connections
    $ ls

Assuming there's just one file here (named the same as your WiFi network), edit this file:

    $ sudo vim *

And find the line starting `permissions`, it should be of the form:

    permissions=user:my-user-name:;

Remove everything _after_ the `=` leaving just:

    permissions=

**3.** Setting up SSH keys to allow password-less SSH login to the Nano.

This is slightly more involved than the other steps and I'm presuming many people already know how to do this - so I'm not going to include a tutorial here. There are many such tutorials on the web, e.g. [this one](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2) from DigitalOcean.

Benchmarking
------------

To try things by hand (assuming you've already installed the SSD-Mobilenet-V2 benchmark in `/usr/src/tensorrt/bin` as per the Nvidia [benchmarks page](https://devtalk.nvidia.com/default/topic/1050377/jetson-nano/deep-learning-inference-benchmarking-instructions/)) the steps are as follows.

Reboot the system:

    $ sudo reboot now

After logging in, following the reboot, set the CPU governors:

    $ for gov in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    do
        echo performance | sudo tee -a $gov
    done

Then set the GPU governor:

    $ echo performance | sudo tee -a /sys/devices/57000000.gpu/devfreq/57000000.gpu/governor

Go to the benchmarks directory:

    $ cd /usr/src/tensorrt/bin

Check the GPU temperature etc. before running the benchmark:

    $ timeout 2s tegrastats

Run the benchmark, using `time` to give a total time (as well as the times output by the benchmark):

    $ time sudo ./sample_uff_ssd_rect

Check the GPU temperature etc. after running the benchmark:

    $ timeout 2s tegrastats

Results
-------

If I run through the steps as above the benchmark consistently takes about 12m to run. And no matter how many times I rerun `sample_uff_ssd_rect` it takes a similar amount of time.

However if I reboot the system and _skip_ the setting of the GPU governor, run `sample_uff_ssd_rect` once, and _then_ set the GPU governor, I get much better results when rerunning `sample_uff_ssd_rect`.

Scripts
-------

I've created two scripts called [`bad-performance`](bad-performance) and [`good-performance`](good-performance).

Both reboot the system on completion and I've run both repeatedly overnight from a remote system (to confirm that I get consistent results) like so:

    $ while true
    do
        date
        ssh jetson@jetsonnano.local ./bad-performance
        sleep 120
    done

Obviously you'll need to change `jetson` to the username appropriate for your setup, and change `bad-performance` to `good-performance` if you want to run the other script.

The `bad-performance` script reruns the benchmark three times - showing that it consistently achieves a terrible runtime of 12m.

The `good-performance` script runs the benchmark twice, first without the GPU governor set to performance and then with it set to performance. For comparison it also performs two additional setups:

* The GPU governor set to its default but with the GPU min_freq set to max.
* All clocks etc. set to max with `jetson_clocks`.

As expected using `jetson_clocks` results in the best overall performance but setting all the CPU and GPU governors to performance is almost as good (and probably better when used e.g. in a robot setup, as the governors can still enable thermal throttling when necessary).

You can find the steps for these four comparisons laid out in more detail [here](performance-steps.md).

You can find logs (captured using [`script`](http://man7.org/linux/man-pages/man1/script.1.html)) of running both scripts repeatedly overnight (using the above `while` loop) in [`bad-typescript.txt`](bad-typescript.txt) and [`good-typescript.txt`](good-typescript.txt).
