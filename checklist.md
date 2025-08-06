---
layout: page
title: Checklist
permalink: /checklist/
---

This page is an evolving work-in-progress. It aims to document technical
matters you might need address when hosting an IOI.

## Contest networking infrastructure

There are several required and desirable features in networking infrastructure for an IOI:

- required: the ability to prevent communication between PCs and outside world
- desirable: support for multicast for imaging (preferably at Gigabit to all hosts, or at least 100 Mbps). Do a quick calculation of the time needed to re-image all computers.
- desirable: intuitive IP addressing scheme to allow for quick diagnosis

To prevent contestant PCs communicating, there are several options:

- put each machine onto its own individual VLAN
  - something must be responsible for routing between the individual VLANs and the contest web server (and other services). This could be a network switch performing layer 3 routing (with appropriate firewalling), or each contest server could itself be present on every individual VLAN using 802.1q tagging. A network switch performing layer 3 routing is preferable, because this also allows multicast traffic to span all VLANs. (If the imaging server must itself register onto each VLAN, then the ability to multicast is lost).
  - some Cisco hardware supports the concept of a “Private VLAN”. Machines on a Private VLAN can only send traffic to a designated upstream machine, effectively isolating them from all other machines on the same ethernet segment. This also easily supports multicast

An intuitive IP addressing scheme is very helpful for diagnosing problems on the fly. This can be achieved in several ways, but all require some customisation of the DHCP server:

- Cisco switches can insert an additional option into DHCP requests to identify the switch name and switch port. The DHCP server can then use this to allocate a specific IP address to a specific port.
- If individual VLANs are used, the DHCP server can use the VLAN for identification
- Both methods still need a mapping table between switch port or VLAN to machine location
- Beware that if you pin the IP address offered by DHCP to a specific location or port, then any RFC-respecting DHCP server will not give that IP address to any other machine until its lease has expired. So if you are moving machines between locations or ports, and you have implemented a pinned IP address policy, then you will need to flush any stale leases out from the DHCP server when this happens.

The parts of the network accessible from the outside (public website, online rank list) should be separated from the contest network as much as possible. In particular, a denial of service attack coming from the outside must not affect the competition.

Plan for equipment failures. In particular, make links between switches redundant.

## Desktop environment setup

Typical mid-range laptops are suitable. Identical machines will make life significantly easier. However, past IOIs have been held with slight variations in machine specifications, and made fair by (1) a random allocation of student to machine; (2) ensuring all judging machines (“workers” in CMS-lingo) are identical (3) providing the ability for students to execute test runs on the workers.

- Hardware configuration:
  - Keyboard: ISO physical layout is strongly recommended (it has an extra key between the left shift and Z, which is used in some national layouts)
  - Mouse: a symmetrical one (some people will use it with their left hand)

- BIOS settings:
  - boot order: the machines should be configured to boot from the network and then hard disk only. Booting from external media should be prevented.
  - A BIOS password is good for extra security.
  - All unused peripherals should be disabled in the BIOS (e.g. WiFi, sound, Bluetooth, etc), depending on the BIOS.
- Note that sometimes laptop suppliers can set the desired BIOS configuration pre-delivery, so worth asking before doing it yourself on all machines!
- Some HP BIOSes fallback to a “failsafe” menu boot mode after several unsuccessful boot attempts, requiring human intervention. Ideally, this should be disabled. In the scenario of a few failed boot attempts during imaging, the second last thing you want to do is have to manually reboot 350 machines. The last thing you want to do is reboot 350 machines by navigating through menus.

- At some early stage in the contest hall setup, memtest and badblocks should be executed on every machine. New machines typically fare okay, but second-hand machines may reveal problems. Machines with bad memory and/or bad disk blocks should not be used. Although workarounds exist to ignore bad memory blocks (with the badram kernel patch), and bad disk blocks (ext2/3/4 can maintain a bad block list), failures often tend to spread further.

Software configuration:
- The list of software packages is generally best based off previous years installations. While a specific Linux distribution is not mandated, recent IOIs have used current Ubuntu releases, largely for pragmatic reasons: the CMS software has seen a large amount of testing on Ubuntu, and keeping the workers, servers, and contestant machines largely uniform is easier to manage.

Extra configuration steps which should be performed on each machine:
- steps to lockdown the software environment:
  - remove user from all groups
- disable swap
  - it defeats the memory limits of isolate (unless CONFIG_MEMCG_SWAP is enabled and possibly swapaccount=1 on the kernel command-line)
  - it is generally unnecessary on machines with 4GB of RAM or more
  - most importantly, machines are close to unusable when heavily swapping (need not apply if you are swapping to an SSD)
- disable access to external USB media by either:
  - `chmod 700 /media` -- this way, the media will be mounted, but inaccessible, which might be still unsafe
  - blacklist usb_storage and uas modules
  - disable user access do UDisks by commenting out a part of `/etc/dbus-1/system.d/org.freedesktop.UDisks.conf`
- disable and remove network manager, replace with a network configuration based on `/etc/network/interfaces` and ifupdown
- check all suid programs and remove suid bit except where really needed (see also `dpkg-statoverride`)
- some machines beep too loudly. Blacklist pcspkr module if needed.
- setup remote syslog so that log messages are sent in real-time are not lost in the event of a desktop crash. A remote log server should be set up to receive them. e.g. rsyslog which can write each machine's log to a separate file.
- TODO: publish sample debian packages for contestant machines
- Stack limit should match the value used by CMS (infinity), for example in `/etc/security/limits.conf`.
  However, truly infinite stack size breaks GCC's address sanitizer, so use a large finite value instead.
  On the othre hand, VScode has been seen to break with large stack size, please check it.
- Firefox sometimes saves downloaded files into $TMPDIR (e.g. if "Open with..." is used). This may lead some students to save their work in /tmp, which will unknowingly be lost on reboot. A safe solution would be to wrap Firefox with a script to set TMPDIR=$HOME/tmp/ first (and mkdir the directory).

- Distribute as much as possible by some versioned means (e.g. CMS or a SCM tool).
  - Debian packages can be created which maintain a list of dependencies and/or include files to be placed anywhere on the filesystem. A locally-hosted apt repository can make the maintenance of machines much easier.
  - e.g. a ioi-contestant-software package can depend on all provided software versions. an ioi-contestant-settings package can install any miscellaneous configuration and tweaks required.
  - At IOI 2013, CMS was packaged as Debian packages which were deployed to server and worker machines
  - CEOI2015 used Git to distribute CMS to workers.

## Determinism

To attain more deterministic execution times on contestant machines and workers, some kernel and system settings should be tuned. The isolate sandbox ships with a script called [isolate-check-environment](https://github.com/ioi/isolate/blob/master/isolate-check-environment) that validates and can alter the configuration for many (but not all) of the settings below.

- Disable Linux's address space randomisation (set `/proc/sys/kernel/randomize_va_space` to 0, using an entry in `/etc/sysctl.d/`)
- Run evaluations on a single CPU. The Linux scheduler has a tendency to randomly migrate tasks between CPUs, incurring cache migration costs. Use isolate's configuration file to pin a task to a single CPU and prevent migration (also, this prevents contestants from getting extra cache by running multiple threads)
- Set the cpufreq governor to "performance" to prevent dynamic CPU frequency control (mostly, see Turboboost below)
- Consider disabling Turboboost on CPUs that might support it (most i3/i5/i7 Intel CPUs). Approach this one with caution. Disabling a CPU that Turboboosts from 2.3 GHz to 2.6 GHz would have minimal impact on run-times in exchange for determinism, but the same on a CPU that Turboboosts from 1.6 GHz to 2.8 GHz will incur a much more dramatic slowdown. Perhaps if the ambient temperature is controlled and only one single-threaded task is keeping the CPU busy at 100%, then TB's behaviour may be reasonably deterministic - requires further experimentation to confirm.
- Java: In 2017, it was found that the run-time of the JVM was made much more deterministic by adding the following flags to the invocation of java: `-Xbatch -XX:+UseSerialGC -XX:-TieredCompilation -XX:CICompilerCount=1`. These remove some (but not all) sources of non-deterministic scheduling and execution.
- Make sure that kernel support for transparent huge pages is disabled: `/sys/kernel/mm/transparent_hugepage/enabled` and `/sys/kernel/mm/transparent_hugepage/defrag` should be set to `madvise` or `never`, `/sys/kernel/mm/transparent_hugepage/khugepaged/defrag` to 0.

## Web servers

- nginx is often used as a load-balancer for CWS and other services. The `ip_hash` parameter in nginx will only hash the first 3 octets of an IPv4 address (i.e. 192.168.123.xx). In some setups, these octets will always be nearly the same. In such cases, don't use ip_hash. Instead use hash: `hash $remote_addr consistent;`

## Mass imaging of contestant machines and workers

The udpcast tool has been used at past IOIs to send images to machines, either standalone as part of a custom boot script (2013, 2015) or with CloneZilla tool (2014). This allows imaging to many machines simultaneously using multicast.

Depending on your network and hardware, no further optimisation may be required. But if it is too slow, some ideas:

- Use image compression: compress the image with an algorithm supporting fast decompression such as LZ4 or Snappy
- HDD partitioning: partition the HDD into at least two parts:
  - The first partition is the SYSTEM space that contains the OS and necessary software. This partition is read-only for contestants, and it is expected to be unchanged during the contest. The size of this partition should be kept minimal in order to reduce the time required to re-image all contest machines (when there is a need).
  - The second partition is the USER space, which can be easily reset before/after each contest day.
- In 2015, attempting to re-image all machines simultaneously ended up being very slow, due to excessive retransmissions of lost packets. Re-imaging just one quarter of machines at a time allowed the imaging to complete quickly, and still allowed all machines to be re-imaged in under 15 minutes.

## Worker provisioning

- HTC must tell HSC requirements for task time and memory limits, and also time required for checkers
- For typical task evaluation times (e.g. < 1 minute), 1 worker to 20 students has been a successful rule of thumb, with provision for 3x more than that for safety. e.g. a contest with 300 students should have at least 300 / 20 x 3 = 45 workers
- test evaluation of a while(1) solution, and a sleep(inf) solution

## Database setup

- PostgreSQL replication to a hot-standby slave is reasonably easy to set up, and allows for the entire contest database to be in a consistent state in the event of a DB server failure, without significant loss of data.
- At the end of the competition, you can also simply decouple the replication and use the slave to dump the database, rather than waiting for a dump to complete before you can allow appeals submissions to begin.
- pg_dump may refuse to complete on a slave if there are rows being deleted from the database. So you should stop replication first, or dump on the primary.
- pg_reload from a dumped SQL file can be unreliable with LO. Always check for errors when restoring the database, and check that large objects get restored correctly. The bug may be related to auto vacuum co-inciding with restore.

## Printing

- In IOI 2015, a custom cups filter was used to watermark each page of a printout, and add front and back pages. See https://github.com/ioi/ioi-utilities/tree/master/print-filter for details.
- In IOI 2014, a header page was created for each printout. The header page contained the contestant's name and the seat number, so that the runner can deliver it efficiently. It also contained an image, which was impossible to be generated on contestant's machines, in order to prevent attacks using fake header pages. Two different images were used on different contest days.
- Delivering printouts is non-trivial. Train the runners:
  - how to identify a single print job
  - how to identify the correct table to deliver to

## Things to simulate before the contest

- simulate entire contest
- while(1) solutions
- deleting, adding and replacing test cases, while the competition is in flight. Can you re-evaluate everything in time?

The steps for deleting, adding, and replacing test cases, changing bounds, etc, should be documented so that in the heat of the contest, no steps are forgotten.

- Dealing with faulty contestant machines: moving a student or replacing their machine
- Dealing with faulty workers: unplug a worker and ensure judging continues (not the same as stopping the CMS process, as this results in "connection refused", whereas unplugging it results in no response which is more likely if the hardware or network has failed)

- Power outages: if backup power systems are in place, they should be tested.

## Practice Competition

- How will keyboards/mice/mascots be collected, registered and re-distributed to the correct machines?

## Contest procedures

- How will the students request clarifications? Some students may not be able to input questions in their native language on a keyboard, so will require a paper form to allow translation.
- Will the students have paper and pens?
- Will the students be provided with food and water? (Dry snacks are preferable, without noisy wrappers)
- Announcements should be published in a known place accessible from the students computer. No verbal announcements should be made, other than “There is an announcement. See the announcements page”. Students should also know to check the announcements page after bathroom breaks.
- Students should be notified that submissions near the end of contest may not be judged before the contest finishes (although are still evaluated after the contest and contribute to their score)
- You will need people on the contest floor during the competition able to debug issues with machines, or direct to those who can. All issues with contestant machines should be logged, in case of appeals.

## Appeals

There is little time between the end of the competition and the start of appeals. You should rehearse what will happen in this time:
- Judging needs to complete. Perhaps a re-judge to ensure stability of results (border-line timeout cases are inevitably going to change the results, so don't worry if they do but stick with the results shown to the students the first time!)
- The state of the contest database should be dumped for posterity.
- Test data must be made available to students, along with a mechanism to allow students to run their solutions on the provided test data. CMS now provides an "appeals mode" to support this.
- Students should be able to download their submissions (on day 2, you may want to provide access to both day 1 and day 2 submissions)
- Appeals forms should be ready.

## Organisational aspects

- secrecy of tasks: there may be reasons to keep task details largely secret from the technical committee until quite close to the IOI. But some overlap between HTC and HSC must exist, and is responsible for validating final tasks on contest environment

- Take timestamped notes of when stuff happens, as this is invaluable for determining which students might have affected by any issues and for how long.

- Make sure that the people communicating with contestants before the contest (forum at the website, e-mails, official Facebook, etc.) are familiar with contest rules, e.g., what are the contestants allowed to bring to the contest hall.

- Plan how to update the public website during the contest. At the start of each competition day, publish a link to the online rank list and also the task statements. (Some team leaders are going to put them online anyway, so better do it officially.)

## Translation (and other) network infrastructure

- With the number of attendees at an IOI, a DHCP pool with anything less than 1000 IP addresses will NOT be sufficient, and will almost certainly end up with people unable to connect due to exhaustion of DHCP leases. Ensure all networks to be used by the IOI leaders/guests, have at least 1000 IPs (/22), and any networks for all attendees have at least 2000 or 4000 IPs (/20).

- Internet access is needed during translation of tasks.

- People still use WiFi clients which support only the 2.4 GHz band. Preferably set up a different ESSID in this band, so that it will be occupied only by people who require it.

- DHCP servers embedded in cheap routers are generally crappy. Use a Linux machine as a DHCP server instead. This applies even to the translation network.

- Avoid NAT between parts of the network, it makes debugging of faults harder. Generally, try to keep things as simple as possible.

## Notes from 2015

pg_dump / pg_reload unreliable with LO. Related to auto vacuum maybe? Check errors!

RWS, using at least 10 Mbit/sec BW at the end of the competition.

Clearing the database between day1 and 2 caused submission ids to be reused, and id clashes in RWS. This can be avoided by using the same database, or simply change all ids in day1 to avoid clashes (e.g., prepending "A"). This will be fixed in the CMS repository.

Using LogService may make it easier to monitor issues across all services.

Java requires a large memory margin, otherwise it runs the garbage collector too often, which leads to slowdown. We used 2GB for all tasks.
