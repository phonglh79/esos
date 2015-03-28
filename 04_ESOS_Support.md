### Debugging / Log Files ###
All of the ESOS system log files are written to the "/var/log" directory on a running system. When ESOS reboots or shuts down, the log files are archived to the "esos\_logs" filesystem; the log files are also archived automatically if they grow too large on a running ESOS storage server.

The easiest way to access the archived ESOS log files is to simply mount the "esos\_logs" filesystem:
```
mount /mnt/logs
```
You can then `scp` the tarballs to another machine to examine, or you can take a peek on the ESOS server itself. _Don't forget, any non-standard changes (eg, copying files to /root) made to a running ESOS will be lost on power off or reboot._

There is also a function in the TUI that can be used to easily generate a support archive (Interface -> Support Bundle). This "support bundle" will contain logs from the current set located in /var/log and all of the configuration files from /etc. Obvious files that contain sensitive information are filtered out (eg, SSH key files, /etc/shadow, etc.) from the tarball, but _you should examine the contents of this archive before posting it publicly_.

Another important feature of ESOS is that there are two different kernels. You'll notice two options when booting ESOS (GRUB):
  * ESOS - Enterprise Storage OS $VERSION <**Production**>
  * ESOS - Enterprise Storage OS $VERISON <**Debug**>

The "Production" kernel/mode is what you will typically run in -- its using a kernel with minimal debugging options set and SCST setup for performance.

The "Debug" kernel/mode should be used when you are having problems or issues that you are unable to resolve/diagnose using the normal "Production" kernel. The "Debug" ESOS kernel has a few extra kernel debugging options set and the SCST modules are set for full debugging. When using debug mode, you will have a `trace_level` file in the root of SCST sysfs (/sys/kernel/scst\_tgt/trace\_level). Examine this file for instructions on how to set the trace/log level for SCST. _Some SCST trace settings can produce massive log files; be sure to keep an eye on the root tmpfs file system disk space._

The advantage is you can easily reboot and switch to debugging mode if you are having problems without re-configuring anything. All of the target/device/storage/system/etc. configuration is exactly the same as the production mode.

The init/rc boot messages are also logged to a text file (via bootlogd): /var/log/boot

Check this file if you get a message indicating SCST is not running. Chances are there is some missing SCST device and SCST is automatically unloaded to prevent clobbering the SCST configuration file (/etc/scst.conf).

<br>

<h3>Kernel Panics</h3>
Kdump is implemented in ESOS to capture the memory in case of a system kernel panic. When/if a kernel panic occurs, the dump-capture kernel is loaded and the old memory image (/proc/vmcore) is saved to the "esos_logs" filesystem and then the system is rebooted and the default (production) kernel will be booted like normal.<br>
<br>
If you are unlucky enough to experience a kernel panic, you can retrieve the vmcore file from the esos_logs (label) file system. The vmcore file is a dump of the memory, so it may be quite large; its best to <code>scp</code> it to another machine before decompressing and analyzing.<br>
<br>
<br>

<h3>Performance Issues</h3>
These types of issues can sometimes be quite difficult to diagnose. Assuming the performance problem is realized at the initiator end, you would probably then want to test the performance of your back-end storage to see if that is culprit. ESOS includes <code>fio</code>, a very useful tool that aids in diagnosing these types of problems.<br>
<br>
We'll show a few examples of using fio to perform some tests on block devices from "inside" ESOS. <b>While performing tests using fio that consist only of reads is probably safe, using writes is definitely not if you have data that you care about! You have been warned!</b>

4K random, 100% reads:<br>
<pre><code>fio --bs=4k --direct=1 --rw=randread --ioengine=libaio --iodepth=64 --name=/dev/sdd --runtime=60<br>
</code></pre>

4K random, 100% writes:<br>
<pre><code>fio --bs=4k --direct=1 --rw=randwrite --ioengine=libaio --iodepth=64 --name=/dev/sdd --runtime=60<br>
</code></pre>

4K random, 75% reads + 25% writes:<br>
<pre><code>fio --bs=4k --direct=1 --rw=randrw --ioengine=libaio --iodepth=64 --rwmixread=75 --rwmixwrite=25 --name=/dev/sdd --runtime=60<br>
</code></pre>

These should hopefully at least get you started. The fio tool is very powerful and has extensive documentation (man page) and there are lots of examples available on the 'net.<br>
<br>
If you find your back-end storage is the culprit, then you'll need to address that. If it turns out your back-end storage is quite speedy, you'll need to look at your SCST configuration, and the other layers (eg, LVM2, DRBD, etc.) in between your underlying storage and the initiators. Don't forget about the SAN fabric either!<br>
<br>
<br>

<h3>Getting Help</h3>
These wiki pages should be the first stop when seeking help or getting questions answered. All of the basic information on using ESOS will be listed in that documentation. If you find some key piece of information missing from the wiki pages that is a necessity, please let us know on the <a href='http://groups.google.com/group/esos-users'>esos-users</a> Google Group.<br>
<br>
If you have technical problems or questions, the Google Group (esos-users) is probably the best place to start. Look through the archives for previous posts/answers -- someone may have had your exact problem and may have gotten an answer. It is also worthwhile to take a look on the ESOS Issues (bug tracker) page to see if a bug has been filed, or possibly previously reported and fixed.<br>
<br>
Still don't have a solution to your problem or answer to your question? Post on the <a href='http://groups.google.com/group/esos-users'>esos-users</a> Google Group; this group is the ESOS community. Please be clear and concise with your questions/problems when posting to the group. Be sure to do your homework before posting -- no one wants to see a simple question where the answer could be easily Google'd.