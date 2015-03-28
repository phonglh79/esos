### Configuring Targets: Fibre Channel ###
Below is an example of configuring a BLOCKIO device using SCST. In the example we are using a Linux block device (RAID volume, etc.) as the back-storage device. There are lots of options with SCST -- instead of using a raw block device, you could partition the device and put a filesystem on and then create multiple virtual devices using files. See the SCST documentation for more information.

The first step is creating the virtual disk using BLOCKIO mode (vdisk\_blockio):
```
scstadmin -open_dev test_disk_1 -handler vdisk_blockio -attributes filename=/dev/sda,blocksize=512
scstadmin -set_dev_attr test_disk_1 -attributes threads_pool_type=per_initiator,threads_num=4
```

In our example, the ESOS storage server has two QLogic FC HBAs connected to two fabrics (two independent Fibre Channel switches). The other server that will be using the storage volume we are configuring (initiators) also has two FC HBAs connected to the fabrics and zoned appropriately.

We will now create two security groups, one for each Fibre Channel target. This allows you to do some fine tuning with LUN masking and a few other attributes (io\_grouping\_type, etc.).
```
scstadmin -add_group myserver -driver qla2x00t -target 21:00:00:1b:32:01:6b:11
scstadmin -add_group myserver -driver qla2x00t -target 21:01:00:1b:32:21:6b:11
```

Next we need to know the WWN's of the initiators on our server that will be using the storage. A couple different ways of getting this... you could look through the system configuration information that the OS provides on the server, you could grab them from your Fibre Channel switches, or you could get them from inside ESOS by looking at data in sysfs. Take a look here:
```
cat /sys/class/fc_host/host8/device/scsi_host/host8/port_database
cat /sys/class/fc_host/host9/device/scsi_host/host9/port_database
```

Once you have the WWN's of the server's initiators, you can add them to the security groups:
```
scstadmin -add_init 21:00:00:e0:8b:9e:ce:24 -driver qla2x00t -target 21:00:00:1b:32:01:6b:11 -group myserver
scstadmin -add_init 21:01:00:e0:8b:be:ce:24 -driver qla2x00t -target 21:01:00:1b:32:21:6b:11 -group myserver
```

Now we can add the LUN's for our device to each target. **Be sure to start LU numbering with 0 (zero) unless you are sure the initiator side has sparse LUN support (eg, VMware ESX).** Its probably also best to keep the LUN's the same for each server/volume combination.
```
scstadmin -add_lun 0 -driver qla2x00t -target 21:00:00:1b:32:01:6b:11 -group myserver -device test_disk_1
scstadmin -add_lun 0 -driver qla2x00t -target 21:01:00:1b:32:21:6b:11 -group myserver -device test_disk_1
```

Finally, we can enable our targets:
```
scstadmin -enable_target 21:00:00:1b:32:01:6b:11 -driver qla2x00t
scstadmin -enable_target 21:01:00:1b:32:21:6b:11 -driver qla2x00t
```

Be sure to save your SCST configuration to a flat file after you have made changes:
```
scstadmin -nonkey -write_config /etc/scst.conf
```

You will need to do an HBA re-scan on the initiator side to find the new targets (SCSI disk); or on some operating systems, issuing a LIP will do the trick:
```
scstadmin -issue_lip
```

You should now have a brand new volume available on the initiator side. Again, please see the SCST documentation -- the above was just one example, SCST can do a whole lot more.

<br>

<h3>Configuring Targets: iSCSI</h3>
In this example, we will be making use of SCST's BLOCKIO mode for our storage device. ESOS supports several different NIC / CNA / TCP Offload Engine (TOE) combinations; see the <a href='Supported_Hardware.md'>Supported_Hardware</a> document for more information.<br>
<br>
The first step is creating the virtual disk using BLOCKIO mode (vdisk_blockio):<br>
<pre><code>scstadmin -open_dev test_disk_1 -handler vdisk_blockio -attributes filename=/dev/sda,blocksize=512<br>
scstadmin -set_dev_attr test_disk_1 -attributes threads_pool_type=per_initiator,threads_num=4<br>
</code></pre>

Before creating the iSCSI target(s) in SCST, you will need to come up with a iSCSI Qualified Name (IQN) for each target on your ESOS storage server. <a href='http://tools.ietf.org/html/rfc3721#section-1.1'>RFC 3721</a> contains more information and examples on IQN, but briefly it goes like this (example taken from Wikipedia):<br>
<br>
<pre><code>                 Naming     Optional String Defined By<br>
    Type  Date   Authority  "example.com" Naming Authority<br>
    +--++------++---------++-----------------------------+<br>
    |  ||      ||         ||                             |<br>
<br>
    iqn.2001-04.com.example:storage:diskarrays-sn-a8675309<br>
    iqn.2001-04.com.example<br>
    iqn.2001-04.com.example:storage.tape1.sys1.xyz<br>
    iqn.2001-04.com.example:storage.disk2.sys1.xyz<br>
</code></pre>

<ul><li>'Type' is always the string "iqn"<br>
</li><li>'Date' is the date that the naming authority took ownership of the domain in "yyyy-mm" format<br>
</li><li>'Naming Authority' is the reversed domain name of the authority (eg, example.com is com.example)<br>
</li><li>The last section is optional; prefix with a ':' and give a storage target name specified by the naming authority</li></ul>

Once you have a decided on a name, you can now create the iSCSI target using <code>scstadmin</code>:<br>
<pre><code>scstadmin -add_target iqn.2012-02.local.esoshost:storage1 -driver iscsi<br>
</code></pre>

We can now create a security group for our new iSCSI target so only specified initiators can access the storage:<br>
<pre><code>scstadmin -add_group myserver -driver iscsi -target iqn.2012-02.local.esoshost:storage1<br>
</code></pre>

Now add an initiator to the group:<br>
<pre><code>scstadmin -add_init iqn.1991-05.com.microsoft:computer.domain -driver iscsi -target iqn.2012-02.local.esoshost:storage1 -group myserver<br>
</code></pre>

Next we will setup our BLOCKIO device as LUN 0 in our security group:<br>
<pre><code>scstadmin -add_lun 0 -driver iscsi -target iqn.2012-02.local.esoshost:storage1 -group myserver -device test_disk_1<br>
</code></pre>

Finally we can enable the iSCSI target:<br>
<pre><code>scstadmin -enable_target iqn.2012-02.local.esoshost:storage1 -driver iscsi<br>
scstadmin -set_drv_attr iscsi -attributes enabled=1<br>
</code></pre>

Don't forget to save your SCST configuration:<br>
<pre><code>scstadmin -nonkey -write_config /etc/scst.conf<br>
</code></pre>

You can now discover the new target via your iSCSI initiator.<br>
<br>
<br>

<h3>Configuring Targets: IB / SRP</h3>
Again, in this example we will use BLOCKIO mode for our device. We start with creating the SCST device, and then move to configuring the security group, and adding the LUN. This example uses InfiniBand/SRP (ib_srpt SCST driver) as the target.<br>
<br>
The first step is creating the virtual disk using BLOCKIO mode (vdisk_blockio):<br>
<pre><code>scstadmin -open_dev test_disk_1 -handler vdisk_blockio -attributes filename=/dev/sda,blocksize=512<br>
scstadmin -set_dev_attr test_disk_1 -attributes threads_pool_type=per_initiator,threads_num=4<br>
</code></pre>

Next, we need the ib_srpt target name. The <code>scstadmin</code> utility has lots of useful functions for getting information from SCST too; see the SCST documentation for more details.<br>
<pre><code>scstadmin -list_target -driver ib_srpt<br>
</code></pre>

Now we create a security group for the InfiniBand/SRP target:<br>
<pre><code>scstadmin -add_group myserver -driver ib_srpt -target ib_srpt_target_0<br>
</code></pre>

Now add the IB/SRP initiator to the security group:<br>
<pre><code>scstadmin -add_init 0x0002c9020022f9fd0002c802003210cb -driver ib_srpt -target ib_srpt_target_0 -group myserver<br>
</code></pre>

Next add the LUN mapping (0) for our BLOCKIO device:<br>
<pre><code>scstadmin -add_lun 0 -driver ib_srpt -target ib_srpt_target_0 -group myserver -device test_disk_1<br>
</code></pre>

Finally, enable the IB/SRP target:<br>
<pre><code>scstadmin -enable_target ib_srpt_target_0 -driver ib_srpt<br>
</code></pre>

Save your SCST configuration:<br>
<pre><code>scstadmin -nonkey -write_config /etc/scst.conf<br>
</code></pre>

You can now re-scan from the initiator side to access the new target.