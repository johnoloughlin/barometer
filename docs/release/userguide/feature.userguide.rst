.. This work is licensed under a Creative Commons Attribution 4.0 International License.
.. http://creativecommons.org/licenses/by/4.0
.. (c) <optionally add copywriters name>

===================================
OPNFV Barometer User Guide
===================================

.. contents::
   :depth: 3
   :local:

Barometer collectd plugins description
---------------------------------------
.. Describe the specific features and how it is realised in the scenario in a brief manner
.. to ensure the user understand the context for the user guide instructions to follow.

collectd is a daemon which collects system performance statistics periodically
and provides a variety of mechanisms to publish the collected metrics. It
supports more than 90 different input and output plugins. Input plugins
retrieve metrics and publish them to the collectd deamon, while output plugins
publish the data they receive to an end point. collectd also has infrastructure
to support thresholding and notification.

Barometer has enabled the following collectd plugins:

* *dpdkstat plugin*: A read plugin that retrieve stats from the DPDK extended
   NIC stats API.

* `ceilometer plugin`_: A write plugin that pushes the retrieved stats to
  Ceilometer. It's capable of pushing any stats read through collectd to
  Ceilometer, not just the DPDK stats.

* *hugepages plugin*:  A read plugin that retrieves the number of available
  and free hugepages on a platform as well as what is available in terms of
  hugepages per socket.

* *RDT plugin*: A read plugin that provides the last level cache utilitzation and
  memory bandwidth utilization

* *Open vSwitch events Plugin*: A read plugin that retrieves events from OVS.

* *mcelog plugin*: A read plugin that uses mcelog client protocol to check for
  memory Machine Check Exceptions and sends the stats for reported exceptions

All the plugins above are available on the collectd master, except for the
plugin as it's a python based plugin and only C plugins are accepted
by the collectd community. The ceilometer plugin lives in the OpenStack
repositories.

Other plugins under development or existing as a pull request into collectd master:

* *dpdkevents*:  A read plugin that retrieves DPDK link status and DPDK
  forwarding cores liveliness status (DPDK Keep Alive).

* *Open vSwitch stats Plugin*: A read plugin that retrieve flow and interface
  stats from OVS.

* *SNMP Agent*: A write plugin that will act as a AgentX subagent that receives
  and handles queries from SNMP master agent and returns the data collected
  by read plugins. The SNMP Agent plugin handles requests only for OIDs
  specified in configuration file. To handle SNMP queries the plugin gets data
  from collectd and translates requested values from collectd's internal format
  to SNMP format. Supports SNMP: get, getnext and walk requests.

* *Legacy/IPMI*: A read plugin that reports platform thermals, voltages,
  fanspeed, current, flow, power etc. Also, the plugin monitors Intelligent
  Platform Management Interface (IPMI) System Event Log (SEL) and sends the

**Plugins included in the Danube release:**

* Hugepages
* Open vSwitch Events
* Ceilometer
* Mcelog

collectd capabilities and usage
------------------------------------
.. Describe the specific capabilities and usage for <XYZ> feature.
.. Provide enough information that a user will be able to operate the feature on a deployed scenario.

**NOTE** Plugins included in the OPNFV D release will be built-in to the fuel
plugin and available in the /opt/opnfv directory on the fuel master. You don't
need to clone the barometer/collectd repos to use these, but you can configure
them as shown in the examples below. Please note, the collectd plugins in OPNFV
are configured with reasonable defaults, but can be overriden.

Building all Barometer upstreamed plugins from scratch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The plugins that have been merged to the collectd master branch can all be
built and configured through the barometer repository.

**NOTE: sudo permissions are required to install collectd.**

**NOTE: These are instructions for Ubuntu 16.04.**

To build and install these dependencies, clone the barometer repo:

.. code:: c

    $ git clone https://gerrit.opnfv.org/gerrit/barometer

Install the build dependencies

.. code:: bash

    $ ./src/install_build_deps.sh

To install collectd as a service and install all it's dependencies:

.. code:: bash

    $ cd barometer/src && sudo make && sudo make install

This will install collectd as a service and the base install directory
is /opt/collectd.

Sample configuration files can be found in '/opt/collectd/etc/collectd.conf.d'

**Note**: Exec plugin requires non-root user to execute scripts. By default,
`collectd_exec` user is used. Barometer scripts do *not* create this user. It
needs to be manually added or exec plugin configuration has to be changed to use
other, existing user before starting collectd service.

Please note if you are using any Open vSwitch plugins you need to run:


.. code:: bash

    $ sudo ovs-vsctl set-manager ptcp:6640

DPDK statistics plugin
^^^^^^^^^^^^^^^^^^^^^^
Repo: https://github.com/collectd/collectd

Branch: master

Dependencies: DPDK (http://dpdk.org/)

To build and install DPDK to /usr please see:
https://github.com/collectd/collectd/blob/master/docs/BUILD.dpdkstat.md

Building and installing collectd:

.. code:: bash

    $ git clone https://github.com/collectd/collectd.git
    $ cd collectd
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-debug
    $ make
    $ sudo make install


This will install collectd to /opt/collectd
The collectd configuration file can be found at /opt/collectd/etc
To configure the hugepages plugin you need to modify the configuration file to
include:

.. code:: bash

    LoadPlugin dpdkstat
    <Plugin dpdkstat>
           Coremask "0xf"
           ProcessType "secondary"
           FilePrefix "rte"
           EnabledPortMask 0xffff
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/collectd/collectd/blob/master/src/collectd.conf.pod

Please note if you are configuring collectd with the **static DPDK library**
you must compile the DPDK library with the -fPIC flag:

.. code:: bash

    $ make EXTRA_CFLAGS=-fPIC

You must also modify the configuration step when building collectd:

.. code:: bash

    $ ./configure CFLAGS=" -lpthread -Wl,--whole-archive -Wl,-ldpdk -Wl,-lm -Wl,-lrt -Wl,-lpcap -Wl,-ldl -Wl,--no-whole-archive"

Please also note that if you are not building and installing DPDK system-wide
you will need to specify the specific paths to the header files and libraries
using LIBDPDK_CPPFLAGS and LIBDPDK_LDFLAGS. You will also need to add the DPDK
library symbols to the shared library path using ldconfig. Note that this
update to the shared library path is not persistant (i.e. it will not survive a
reboot). Pending a merge of https://github.com/collectd/collectd/pull/2073.

.. code:: bash

    $ ./configure LIBDPDK_CPPFLAGS="path to DPDK header files" LIBDPDK_LDFLAGS="path to DPDK libraries"

Hugepages Plugin
^^^^^^^^^^^^^^^^^
Repo: https://github.com/collectd/collectd

Branch: master

Dependencies: None, but assumes hugepages are configured.

To configure some hugepages:

.. code:: bash

   sudo mkdir -p /mnt/huge
   sudo mount -t hugetlbfs nodev /mnt/huge
   sudo echo 14336 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages

Building and installing collectd:

.. code:: bash

    $ git clone https://github.com/collectd/collectd.git
    $ cd collectd
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-hugepages --enable-debug
    $ make
    $ sudo make install

This will install collectd to /opt/collectd
The collectd configuration file can be found at /opt/collectd/etc
To configure the hugepages plugin you need to modify the configuration file to
include:

.. code:: bash

    LoadPlugin hugepages
    <Plugin hugepages>
        ReportPerNodeHP  true
        ReportRootHP     true
        ValuesPages      true
        ValuesBytes      false
        ValuesPercentage false
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/collectd/collectd/blob/master/src/collectd.conf.pod

Intel RDT Plugin
^^^^^^^^^^^^^^^^
Repo: https://github.com/collectd/collectd

Branch: master

Dependencies:

  * PQoS/Intel RDT library https://github.com/01org/intel-cmt-cat.git
  * msr kernel module

Building and installing PQoS/Intel RDT library:

.. code:: bash

    $ git clone https://github.com/01org/intel-cmt-cat.git
    $ cd intel-cmt-cat
    $ make
    $ make install PREFIX=/usr

You will need to insert the msr kernel module:

.. code:: bash

    $ modprobe msr

Building and installing collectd:

.. code:: bash

    $ git clone https://github.com/collectd/collectd.git
    $ cd collectd
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --with-libpqos=/usr/ --enable-debug
    $ make
    $ sudo make install

This will install collectd to /opt/collectd
The collectd configuration file can be found at /opt/collectd/etc
To configure the RDT plugin you need to modify the configuration file to
include:

.. code:: bash

    <LoadPlugin intel_rdt>
      Interval 1
    </LoadPlugin>
    <Plugin "intel_rdt">
      Cores ""
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/collectd/collectd/blob/master/src/collectd.conf.pod

IPMI Plugin
^^^^^^^^^^^^
Repo: https://github.com/maryamtahhan/collectd

Branch: feat_ipmi_events, feat_ipmi_analog

Dependencies: OpenIPMI library

The IPMI plugin is already implemented in the latest collectd and sensors
like temperature, voltage, fanspeed, current are already supported there.
The list of supported IPMI sensors has been extended and sensors like flow,
power are supported now. Also, a System Event Log (SEL) notification feature
has been introduced.

* The feat_ipmi_events branch includes new SEL feature support in collectd
  IPMI plugin. If this feature is enabled, the collectd IPMI plugin will
  dispatch notifications about new events in System Event Log.

* The feat_ipmi_analog branch includes the support of extended IPMI sensors in
  collectd IPMI plugin.

On Ubuntu, install the dependencies:

.. code:: bash

    $ sudo apt-get install libopenipmi-dev

Enable IPMI support in the kernel:

.. code:: bash

    $ sudo modprobe ipmi_devintf
    $ sudo modprobe ipmi_si

**Note**: If HW supports IPMI, the ``/dev/ipmi0`` character device will be
created.

Clone and install the collectd IPMI plugin:

.. code:: bash

    $ git clone  https://github.com/maryamtahhan/collectd
    $ cd collectd
    $ git checkout $BRANCH
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-debug
    $ make
    $ sudo make install

Where $BRANCH is feat_ipmi_events or feat_ipmi_analog.

This will install collectd to default folder ``/opt/collectd``. The collectd
configuration file (``collectd.conf``) can be found at ``/opt/collectd/etc``. To
configure the IPMI plugin you need to modify the file to include:

.. code:: bash

    LoadPlugin ipmi
    <Plugin ipmi>
       SELEnabled true # only feat_ipmi_events branch supports this
    </Plugin>

**Note**: By default, IPMI plugin will read all available analog sensor values,
dispatch the values to collectd and send SEL notifications.

For more information on the IPMI plugin parameters and SEL feature configuration,
please see:
https://github.com/maryamtahhan/collectd/blob/feat_ipmi_events/src/collectd.conf.pod

Extended analog sensors support doesn't require additional configuration. The usual
collectd IPMI documentation can be used:

- https://collectd.org/wiki/index.php/Plugin:IPMI
- https://collectd.org/documentation/manpages/collectd.conf.5.shtml#plugin_ipmi

IPMI documentation:

- https://www.kernel.org/doc/Documentation/IPMI.txt
- http://www.intel.com/content/www/us/en/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html

Mcelog Plugin
^^^^^^^^^^^^^^
Repo: https://github.com/collectd/collectd

Branch: master

Dependencies: mcelog

Start by installing mcelog. Note: The kernel has to have CONFIG_X86_MCE
enabled. For 32bit kernels you need at least a 2.6,30 kernel.

On ubuntu:

.. code:: bash

    $ apt-get update && apt-get install mcelog

Or build from source

.. code:: bash

    $ git clone git://git.kernel.org/pub/scm/utils/cpu/mce/mcelog.git
    $ cd mcelog
    $ make
    ... become root ...
    $ make install
    $ cp mcelog.service /etc/systemd/system/
    $ systemctl enable mcelog.service
    $ systemctl start mcelog.service


Verify you got a /dev/mcelog. You can verify the daemon is running completely
by running:

.. code:: bash

     $ mcelog --client

This should query the information in the running daemon. If it prints nothing
that is fine (no errors logged yet). More info @
http://www.mcelog.org/installation.html

Modify the mcelog configuration file "/etc/mcelog/mcelog.conf" to include or
enable:

.. code:: bash

    socket-path = /var/run/mcelog-client

Clone and install the collectd mcelog plugin:

.. code:: bash

    $ git clone  https://github.com/maryamtahhan/collectd
    $ cd collectd
    $ git checkout feat_ras
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-debug
    $ make
    $ sudo make install

This will install collectd to /opt/collectd
The collectd configuration file can be found at /opt/collectd/etc
To configure the mcelog plugin you need to modify the configuration file to
include:

.. code:: bash

    <LoadPlugin mcelog>
      Interval 1
    </LoadPlugin>
    <Plugin "mcelog">
       McelogClientSocket "/var/run/mcelog-client"
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/maryamtahhan/collectd/blob/feat_ras/src/collectd.conf.pod

Simulating a Machine Check Exception can be done in one of 3 ways:

* Running $make test in the mcelog cloned directory - mcelog test suite
* using mce-inject
* using mce-test

**mcelog test suite:**

It is always a good idea to test an error handling mechanism before it is
really needed. mcelog includes a test suite. The test suite relies on
mce-inject which needs to be installed and in $PATH.

You also need the mce-inject kernel module configured (with
CONFIG_X86_MCE_INJECT=y), compiled, installed and loaded:

.. code:: bash

    $ modprobe mce-inject

Then you can run the mcelog test suite with

.. code:: bash

    $ make test

This will inject different classes of errors and check that the mcelog triggers
runs. There will be some kernel messages about page offlining attempts. The
test will also lose a few pages of memory in your system (not significant)
**Note this test will kill any running mcelog, which needs to be restarted
manually afterwards**.
**mce-inject:**

A utility to inject corrected, uncorrected and fatal machine check exceptions

.. code:: bash

    $ git clone https://git.kernel.org/pub/scm/utils/cpu/mce/mce-inject.git
    $ cd mce-inject
    $ make
    $ modprobe mce-inject

Modify the test/corrected script to include the following:

.. code:: bash

    CPU 0 BANK 0
    STATUS 0xcc00008000010090
    ADDR 0x0010FFFFFFF

Inject the error:
.. code:: bash

    $ ./mce-inject < test/corrected

**Note: the uncorrected and fatal scripts under test will cause a platform reset.
Only the fatal script generates the memory errors**. In order to  quickly
emulate uncorrected memory errors and avoid host reboot following test errors
from mce-test  suite can be injected:

.. code:: bash

       $ mce-inject  mce-test/cases/coverage/soft-inj/recoverable_ucr/data/srao_mem_scrub

**mce-test:**

In addition an more in-depth test of the Linux kernel machine check facilities
can be done with the mce-test test suite. mce-test supports testing uncorrected
error handling, real error injection, handling of different soft offlining
cases, and other tests.

**Corrected memory error injection:**

To inject corrected memory errors:

* Remove sb_edac and edac_core kernel modules: rmmod sb_edac rmmod edac_core
* Insert einj module: modprobe einj param_extension=1
* Inject an error by specifying details (last command should be repeated at least two times):

.. code:: bash

    $ APEI_IF=/sys/kernel/debug/apei/einj
    $ echo 0x8 > $APEI_IF/error_type
    $ echo 0x01f5591000 > $APEI_IF/param1
    $ echo 0xfffffffffffff000 > $APEI_IF/param2
    $ echo 1 > $APEI_IF/notrigger
    $ echo 1 > $APEI_IF/error_inject

* Check the MCE statistic: mcelog --client. Check the mcelog log for injected error details: less /var/log/mcelog.

Open vSwitch Plugins
^^^^^^^^^^^^^^^^^^^^^
OvS Events Repo: https://github.com/collectd/collectd

OvS Stats Repo: https://github.com/maryamtahhan/collectd

OvS Events Branch: master

OvS Stats Branch:feat_ovs_stats

Dependencies: Open vSwitch, libyajl

On Ubuntu, install the dependencies:

.. code:: bash

    $ sudo apt-get install libyajl-dev openvswitch-switch

Start the Open vSwitch service:

.. code:: bash

    $ sudo service openvswitch-switch start

configure the ovsdb-server manager:

.. code:: bash

    $ sudo ovs-vsctl set-manager ptcp:6640

Clone and install the collectd ovs plugin:

.. code:: bash

    $ git clone $REPO
    $ cd collectd
    $ git checkout $BRANCH
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-debug
    $ make
    $ sudo make install

where $REPO is one of the repos listed at the top of this section.

Where $BRANCH is master or feat_ovs_stats.

This will install collectd to /opt/collectd. The collectd configuration file
can be found at /opt/collectd/etc. To configure the OVS events plugin you
need to modify the configuration file to include:

.. code:: bash

    <LoadPlugin ovs_events>
       Interval 1
    </LoadPlugin>
    <Plugin "ovs_events">
       Port 6640
       Socket "/var/run/openvswitch/db.sock"
       Interfaces "br0" "veth0"
       SendNotification false
    </Plugin>

To configure the OVS stats plugin you need to modify the configuration file
to include:

.. code:: bash

    <LoadPlugin ovs_stats>
       Interval 1
    </LoadPlugin>
    <Plugin ovs_stats>
       Port "6640"
       Address "127.0.0.1"
       Socket "/var/run/openvswitch/db.sock"
       Bridges "br0" "br_ext"
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/collectd/collectd/blob/master/src/collectd.conf.pod
and
https://github.com/maryamtahhan/collectd/blob/feat_ovs_stats/src/collectd.conf.pod

SNMP Agent Plugin
^^^^^^^^^^^^^^^^^
Repo: https://github.com/maryamtahhan/collectd/

Branch: feat_snmp

Dependencies: NET-SNMP library

Start by installing net-snmp and dependencies.

On ubuntu:

.. code:: bash

    $ apt-get install snmp snmp-mibs-downloader snmpd libsnmp-dev
    $ systemctl start snmpd.service

Or build from source

Become root to install net-snmp dependencies

.. code:: bash

    $ apt-get install libperl-dev

Clone and build net-snmp

.. code:: bash

    $ git clone https://github.com/haad/net-snmp.git
    $ cd net-snmp
    $ ./configure --with-persistent-directory="/var/net-snmp" --with-systemd --enable-shared --prefix=/usr
    $ make

Become root

.. code:: bash

    $ make install

Copy default configuration to persistent folder

.. code:: bash

    $ cp EXAMPLE.conf /usr/share/snmp/snmpd.conf

Set library path and default MIB configuration

.. code:: bash

    $ cd ~/
    $ echo export LD_LIBRARY_PATH=/usr/lib >> .bashrc
    $ net-snmp-config --default-mibdirs
    $ net-snmp-config --snmpconfpath

Configure snmpd as a service

.. code:: bash

    $ cd net-snmp
    $ cp ./dist/snmpd.service /etc/systemd/system/
    $ systemctl enable snmpd.service
    $ systemctl start snmpd.service

Add the following line to snmpd.conf configuration file
"/usr/share/snmp/snmpd.conf" to make all OID tree visible for SNMP clients:

.. code:: bash

    view   systemonly  included   .1

To verify that SNMP is working you can get IF-MIB table using SNMP client
to view the list of Linux interfaces:

.. code:: bash

    $ snmpwalk -v 2c -c public localhost IF-MIB::interfaces

Clone and install the collectd snmp_agent plugin:

.. code:: bash

    $ git clone  https://github.com/maryamtahhan/collectd
    $ cd collectd
    $ git checkout feat_snmp
    $ ./build.sh
    $ ./configure --enable-syslog --enable-logfile --enable-debug --enable-snmp --with-libnetsnmp
    $ make
    $ sudo make install

This will install collectd to /opt/collectd
The collectd configuration file can be found at /opt/collectd/etc
**SNMP Agent plugin is a generic plugin and cannot work without configuration**.
To configure the snmp_agent plugin you need to modify the configuration file to
include OIDs mapped to collectd types. The following example maps scalar
memAvailReal OID to value represented as free memory type of memory plugin:

.. code:: bash

    LoadPlugin snmp_agent
    <Plugin "snmp_agent">
      <Data "memAvailReal">
        Plugin "memory"
        Type "memory"
        TypeInstance "free"
        OIDs "1.3.6.1.4.1.2021.4.6.0"
      </Data>
    </Plugin>

For more information on the plugin parameters, please see:
https://github.com/maryamtahhan/collectd/blob/feat_snmp/src/collectd.conf.pod

For more details on AgentX subagent, please see:
http://www.net-snmp.org/tutorial/tutorial-5/toolkit/demon/

Installing collectd as a service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
**NOTE**: In an OPNFV installation, collectd is installed and configured as a
service.

Collectd service scripts are available in the collectd/contrib directory.
To install collectd as a service:

.. code:: bash

    $ sudo cp contrib/systemd.collectd.service /etc/systemd/system/
    $ cd /etc/systemd/system/
    $ sudo mv systemd.collectd.service collectd.service
    $ sudo chmod +x collectd.service

Modify collectd.service

.. code:: bash

    [Service]
    ExecStart=/opt/collectd/sbin/collectd
    EnvironmentFile=-/opt/collectd/etc/
    EnvironmentFile=-/opt/collectd/etc/
    CapabilityBoundingSet=CAP_SETUID CAP_SETGID

Reload

.. code:: bash

    $ sudo systemctl daemon-reload
    $ sudo systemctl start collectd.service
    $ sudo systemctl status collectd.service should show success

Additional useful plugins
^^^^^^^^^^^^^^^^^^^^^^^^^^

* **Exec Plugin** : Can be used to show you when notifications are being
 generated by calling a bash script that dumps notifications to file. (handy
 for debug). Modify /opt/collectd/etc/collectd.conf:

.. code:: bash

   LoadPlugin exec
   <Plugin exec>
   #   Exec "user:group" "/path/to/exec"
      NotificationExec "user" "<path to barometer>/barometer/src/collectd/collectd_sample_configs/write_notification.sh"
   </Plugin>

write_notification.sh (just writes the notification passed from exec through
STDIN to a file (/tmp/notifications)):

.. code:: bash

   #!/bin/bash
   rm -f /tmp/notifications
   while read x y
   do
     echo $x$y >> /tmp/notifications
   done

output to /tmp/notifications should look like:

.. code:: bash

    Severity:WARNING
    Time:1479991318.806
    Host:localhost
    Plugin:ovs_events
    PluginInstance:br-ex
    Type:gauge
    TypeInstance:link_status
    uuid:f2aafeec-fa98-4e76-aec5-18ae9fc74589

    linkstate of "br-ex" interface has been changed to "DOWN"

* **logfile plugin**: Can be used to log collectd activity. Modify
  /opt/collectd/etc/collectd.conf to include:

.. code:: bash

    LoadPlugin logfile
    <Plugin logfile>
        LogLevel info
        File "/var/log/collectd.log"
        Timestamp true
        PrintSeverity false
    </Plugin>


Monitoring Interfaces and Openstack Support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. Figure:: monitoring_interfaces.png

   Monitoring Interfaces and Openstack Support

The figure above shows the DPDK L2 forwarding application running on a compute
node, sending and receiving traffic. collectd is also running on this compute
node retrieving the stats periodically from DPDK through the dpdkstat plugin
and publishing the retrieved stats to Ceilometer through the ceilometer plugin.

To see this demo in action please checkout: `Barometer OPNFV Summit demo`_

References
^^^^^^^^^^^
.. [1] https://collectd.org/wiki/index.php/Naming_schema
.. [2] https://github.com/collectd/collectd/blob/master/src/daemon/plugin.h
.. [3] https://collectd.org/wiki/index.php/Value_list_t
.. [4] https://collectd.org/wiki/index.php/Data_set
.. [5] https://collectd.org/documentation/manpages/types.db.5.shtml
.. [6] https://collectd.org/wiki/index.php/Data_source
.. [7] https://collectd.org/wiki/index.php/Meta_Data_Interface

.. _Barometer OPNFV Summit demo: https://prezi.com/kjv6o8ixs6se/software-fastpath-service-quality-metrics-demo/
.. _ceilometer plugin: https://github.com/openstack/collectd-ceilometer-plugin/tree/stable/mitaka
