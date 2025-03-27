---
layout: page
title: Checklist
permalink: /checklist/
---

This page is an evolving work-in-progress. It aims to document technical
matters you might need address when hosting an IOI.

## Contest networking infrastructure

There are several required and desirable features in networking infrastructure for an IOI:

- the ability to prevent communication between PCs and outside world
- sufficient speed:
    - 1 Gbps for contestant machines
    - 1 Gbps for grading workers
    - preferably 10 Gpbs for servers
    - preferably 10 Gpbs for backbone between switches
- desirable: support for multicast for imaging. Do a quick calculation of the time needed to re-image all computers.
- desirable: intuitive IP addressing scheme to allow for quick diagnosis
- desirable: a local DNS server (synchronizing `/etc/hosts` is painful)

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

If you plan to prepare the machines at another location and move them to the final place soon before the start of the contest, plan for that in advance and preferably design an addressing scheme that works in both locations.

## Desktop environment setup

Typical mid-range laptops are suitable. Identical machines will make life significantly easier. However, past IOIs have been held with slight variations in machine specifications, and made fair by (1) a random allocation of student to machine; (2) ensuring all judging machines (“workers” in CMS-lingo) are identical (3) providing the ability for students to execute test runs on the workers.

- Hardware configuration:
  - at least 16 GB of RAM – as of 2023, about 50% of contestants preferred VS Code, which is quite memory-hungry.
  - Keyboard: ISO physical layout is strongly recommended (it has an extra key between the left shift and Z, which is used in some national layouts)
  - Mouse: a symmetrical one (some people will use it with their left hand)

- BIOS settings:
  - boot order: the machines should be configured to boot from the network and then local disk only. Booting from external media should be prevented.
  - A BIOS password is good for extra security.
  - All unused peripherals should be disabled in the BIOS (e.g. WiFi, sound, Bluetooth, etc), depending on the BIOS.
- Note that sometimes laptop suppliers can set the desired BIOS configuration pre-delivery, so worth asking before doing it yourself on all machines!
- Some HP BIOSes fallback to a “failsafe” menu boot mode after several unsuccessful boot attempts, requiring human intervention. Ideally, this should be disabled. In the scenario of a few failed boot attempts during imaging, the second last thing you want to do is have to manually reboot 350 machines. The last thing you want to do is reboot 350 machines by navigating through menus.
- At some early stage in the contest hall setup, `memtest86`, `badblocks`, and `smartctl` should be executed on every machine. New machines typically fare okay, but second-hand machines may reveal problems. Machines with bad memory and/or bad disk blocks should not be used. Although workarounds exist to ignore bad memory blocks and bad disk blocks, failures often tend to spread further.

Software configuration:
- The list of software packages is generally best based off previous years installations. While a specific Linux distribution is not mandated, recent IOIs have used current Ubuntu releases, largely for pragmatic reasons: the CMS software has seen a large amount of testing on Ubuntu, and keeping the workers, servers, and contestant machines largely uniform is easier to manage.

Extra configuration steps which should be performed on each machine:
- steps to lockdown the software environment:
  - remove user from all groups
- disable swap
  - it defeats the memory limits of isolate (unless CONFIG_MEMCG_SWAP is enabled and possibly `swapaccount=1` on the kernel command-line)
  - it is generally unnecessary on machines with 4GB of RAM or more
  - most importantly, machines are close to unusable when heavily swapping (need not apply if you are swapping to an SSD)
- disable access to external USB media by either:
  - `chmod 700 /media` -- this way, the media will be mounted, but inaccessible, which might be still unsafe
  - blacklist usb_storage and uas modules
  - disable user access do UDisks by commenting out a part of `/etc/dbus-1/system.d/org.freedesktop.UDisks.conf`
- disable and remove network manager, replace with a network configuration based on `/etc/network/interfaces` and ifupdown
- check all suid programs and remove suid bit except where really needed (see also `dpkg-statoverride`)
- some machines beep too loudly. Blacklist pcspkr module if needed.
- setup remote syslog so that log messages are sent in real-time are not lost in the event of a desktop crash. A remote log server should be set up to receive them. e.g. rsyslog which can write each machine's log to a separate file. This is particularly important for the offline submit mechanism (ioisubmit).
- Set stack limit to match the value used by CMS (infinity), for example in `/etc/security/limits.conf`
- Disable Linux's address space randomisation (set `/proc/sys/kernel/randomize_va_space` to 0, using an entry in `/etc/sysctl.d/`) to make debugging easier.
- Firefox sometimes saves downloaded files into `$TMPDIR` (e.g. if "Open with..." is used). This may lead some students to save their work in /tmp, which will unknowingly be lost on reboot. Preferably set `TMPDIR=$HOME/tmp/` for the whole desktop session (and mkdir the directory).
- Distribute as much as possible by some versioned means (e.g. CMS or a SCM tool).
  - Debian packages can be created which maintain a list of dependencies and/or include files to be placed anywhere on the filesystem. A locally-hosted apt repository can make the maintenance of machines much easier.
  - e.g. a ioi-contestant-software package can depend on all provided software versions. an ioi-contestant-settings package can install any miscellaneous configuration and tweaks required.
  - At IOI 2013, CMS was packaged as Debian packages which were deployed to server and worker machines
  - CEOI 2024 used Git to distribute CMS to workers.
  - Some IOIs also used Ansible.

## Determinism

To attain more deterministic execution times on contestant machines and workers, some kernel and system settings should be tuned.
See [manual page](https://www.ucw.cz/moe/isolate.1.html) of the isolate sandbox
and the [isolate-check-environment](https://github.com/ioi/isolate/blob/master/isolate-check-environment) script
that validates and can alter the configuration for many (but not all) of the settings below.

- Disable Linux's address space randomisation (set `/proc/sys/kernel/randomize_va_space` to 0, using an entry in `/etc/sysctl.d/`)
- Run evaluations on a single CPU. The Linux scheduler has a tendency to randomly migrate tasks between CPUs, incurring cache migration costs. Use isolate's configuration file to pin a task to a single CPU and prevent migration (also, this prevents contestants from getting extra cache by running multiple threads)
- For interactive tasks, it is often better to keep both the solution and the grader on the same logical CPU.
  (Also, allocation of sandbox IDs in CMS is chaotic, so it is hard to pin the solution to one core and the grader to a different one.)
- Set the cpufreq governor to "performance" to prevent dynamic CPU frequency control.
- Consider disabling frequency boosting on CPUs that might support it (this includes most i3/i5/i7 Intel CPUs and the AMD Zen architecture).
  This is done either by writing 1 to `/sys/devices/system/cpu/intel_pstate/no_turbo` (on Intel CPUs)
  or by writing 0 to `/sys/devices/system/cpu/cpufreq/boost` (other machines).
- Java: In 2017, it was found that the run-time of the JVM was made much more deterministic by adding the following flags to the invocation of java: `-Xbatch -XX:+UseSerialGC -XX:-TieredCompilation -XX:CICompilerCount=1`. These remove some (but not all) sources of non-deterministic scheduling and execution.
- Make sure that kernel support for transparent huge pages is disabled: `/sys/kernel/mm/transparent_hugepage/enabled` and `/sys/kernel/mm/transparent_hugepage/defrag` should be set to `madvise` or `never`, `/sys/kernel/mm/transparent_hugepage/khugepaged/defrag` to 0.
- Consider using the `isolcpu` kernel parameter to prevent the OS from scheduling regular processes on the same CPU as the ones used for evaluation.
- Consider disabling hyperthreading on CPUs that support it (most Intel CPUs).
- Recent Intel CPUs have two types of cores: performance and efficiency.
  On grading workers, disable the efficiency cores or pin sandboxes on the performance
  cores.
- In IOI 2020/2021, workers were AWS c5.metal instances. It was concluded that using bare metal instances resulted in more deterministic execution times as supposed to non-bare metal instances.
- In some instances, specifying CPU idle states in BIOS did help increase determinism.
  (I.e., adding `intel_idle.max_cstate=0,intel_pstate=disable` to `GRUB_CMDLINE_LINUX`.)

## Web servers

- nginx is often used as a load-balancer for CWS and other services. The `ip_hash` parameter in nginx will only hash the first 3 octets of an IPv4 address (i.e. 192.168.123.xx). In some setups, these octets will always be nearly the same. In such cases, don't use ip_hash. Instead use hash: `hash $remote_addr consistent;`
  - It is convenient to log request processing time, so that you can tell if the server was overloaded at a given time.
- In IOI 2020, it was found that the browsers on some contestant VMs cached the PDF statements and attachments. When there is an update to these files, it was troublesome to invalidate the cache in order to receive the new versions. A quick fix was to instead configure nginx to always instruct the browser to not cache any files from CMS. i.e. `proxy_hide_header "Cache-Control"; add_header 'Cache-Control' "no-store, no-cache, must-revalidate, max-age=0";`
- DNS entries for the CMS and public rank list should have small TTL (on the order of minutes), so that they can be changed should things go wrong. (In 2023, ranking web server was blackholed by DDOS protections of the cloud vendor and we wanted to relocate it quickly.)


## Mass imaging of contestant machines and workers

The udpcast tool has been used at past IOIs to send images to machines, either standalone as part of a custom boot script (2013, 2015) or with CloneZilla tool (2014). This allows imaging to many machines simultaneously using multicast.

Depending on your network and hardware, no further optimisation may be required. But if it is too slow, some ideas:

- Use image compression: compress the image with an algorithm supporting fast decompression such as LZ4 or Snappy
- HDD partitioning: partition the HDD into at least two parts:
  - The first partition is the SYSTEM space that contains the OS and necessary software. This partition is read-only for contestants, and it is expected to be unchanged during the contest. The size of this partition should be kept minimal in order to reduce the time required to re-image all contest machines (when there is a need).
  - The second partition is the USER space, which can be easily reset before/after each contest day.
- In 2015, attempting to re-image all machines simultaneously ended up being very slow, due to excessive retransmissions of lost packets. Re-imaging just one quarter of machines at a time allowed the imaging to complete quickly, and still allowed all machines to be re-imaged in under 15 minutes.

Between contest days (and also between practice competition and the first contest day),
all files which might have been created or modified by the contestants must be erased.
Such files can appear at unexpected locations (have you ever hidden your editor
configuration in the user's crontab?), some even not owned by the user. It is therefore
recommended to re-image the machines, or at least to use a file system with snapshots
and revert to a clean snapshot.

## Worker provisioning

- HTC must tell HSC requirements for task time and memory limits, and also time required for checkers
- For typical task evaluation times (e.g. < 1 minute), 1 worker to 20 students has been a successful rule of thumb, with provision for 3x more than that for safety. e.g. a contest with 300 students should have at least 300 / 20 x 3 = 45 workers
- Beware that workers can use substantial disk space by caching of test data and compiled executables.

## Database setup

- PostgreSQL replication to a hot-standby slave is reasonably easy to set up, and allows for the entire contest database to be in a consistent state in the event of a DB server failure, without significant loss of data.
- At the end of the competition, you can also simply decouple the replication and use the slave to dump the database, rather than waiting for a dump to complete before you can allow appeals submissions to begin.
- pg_dump may refuse to complete on a slave if there are rows being deleted from the database. So you should stop replication first, or dump on the primary.
- pg_reload from a dumped SQL file can be unreliable with large objects. Always check for errors when restoring the database, and check that large objects get restored correctly. The bug may be related to auto vacuum co-inciding with restore.

## Backup

You should regularly back up home directories of contestants' machines, reasonable
frequency is about once per 10 minutes. It is useful to keep all versions.
We suggest using `rsync` with:

- `--link-dest` to hard-link files which did not change from the previous backup
- `--max-size` to avoid backing up large files (e.g., larger than 10 MB)
- `--exclude` directories with non-interesting, but frequently changing files like `.cache`
and `.mozilla`.

## Things to test before the contest

- simulate an entire contest (we have dumps from previous years in the ITC archive)
- evaluation of a `while(1)` solution and a `sleep(inf)` solution
- deleting, adding and replacing test cases, while the competition is in flight. Can you re-evaluate everything in time?

The steps for deleting, adding, and replacing test cases, changing bounds, etc, should be documented so that in the heat of the contest, no steps are forgotten.

- Dealing with faulty contestant machines: moving a student or replacing their machine
- Dealing with faulty workers: unplug a worker and ensure judging continues (not the same as stopping the CMS process, as this results in "connection refused", whereas unplugging it results in no response which is more likely if the hardware or network has failed)
- Power outages: if backup power systems are in place, they should be tested.
- Load test ContestWebService by simulating all the contestants downloading statements and attachments at once. In IOI 2020/2021, this was performed using apache bench.

## Contest procedures

- How will keyboards/mice/mascots be collected, registered and re-distributed to the correct machines?
    - If an item is rejected, the team leader should be notified early enough
      to communicate it to the contestant before the quarantine starts.
- How will the students request clarifications? Some students may not be able to input questions in their native language on a keyboard, so will require a paper form to allow translation. (IOI 2022: Matrix used to send photos of the forms to the GA mailing list for translation.)
- Will the students have paper and pens?
- Will the students be provided with food and water? (Dry snacks are preferable, without noisy wrappers)
- Announcements should be published in a known place accessible from the students computer. No verbal announcements should be made, other than “There is an announcement. See the announcements page”. Students should also know to check the announcements page after bathroom breaks.
- Students should be notified that submissions near the end of contest may not be judged before the contest finishes (although are still evaluated after the contest and contribute to their score).
- You will need people on the contest floor during the competition able to debug issues with machines, or direct to those who can. All issues with contestant machines should be logged, in case of appeals.

## Appeals

There is little time between the end of the competition and the start of appeals. You should rehearse what will happen in this time:
- Local submissions must be retrieved and added to CMS.
- Judging needs to complete. Perhaps a re-judge to ensure stability of results (border-line timeout cases are inevitably going to change the results, so don't worry if they do but stick with the results shown to the students the first time!)
- During the contest, for several reasons, the connection between ProxyService and Ranking instances may experience data losses. In this case, some or all instances might miss a submission. To ensure rankings are all up to date, restart ProxyService once the judging is finished. ProxyService sends a fresh copy of the latest data at start time. Make sure that this transfer succeeds on all instances. Double check the data on ranking instances with the ranking from admin to make sure they match.
- The state of the contest database should be dumped for posterity.
- Test data must be made available to students, along with a mechanism to allow students to run their solutions on the provided test data. CMS provides an "analysis mode" to support this.
- Students should be able to download their submissions (on day 2, you may want to provide access to both day 1 and day 2 submissions). In 2024, we used the [mailer script](https://github.com/ioi/solution-mailer) to mail out the submissions to team leaders.
- Appeals forms should be ready.

## Organisational aspects

- Secrecy of tasks: there may be reasons to keep task details largely secret from the technical committee until quite close to the IOI. But some overlap between HTC and HSC must exist, and is responsible for validating final tasks on contest environment
- Take timestamped notes of when stuff happens, as this is invaluable for determining which students might have affected by any issues and for how long.
- Make sure that the people communicating with contestants before the contest (forum at the website, e-mails, official Facebook, etc.) are familiar with contest rules, e.g., what are the contestants allowed to bring to the contest hall.
- Plan how to update the public website during the contest. At the start of each competition day, publish a link to the online rank list and also the task statements. (Some team leaders are going to put them online anyway, so better do it officially.)

## Translation (and other) network infrastructure

- With the number of attendees at an IOI, a DHCP pool with anything less than 1000 IP addresses will NOT be sufficient, and will almost certainly end up with people unable to connect due to exhaustion of DHCP leases. Ensure all networks to be used by the IOI leaders/guests, have at least 1000 IPs (/22), and any networks for all attendees have at least 2000 or 4000 IPs (/20).
- Internet access is needed during translation of tasks.
- 2019: People still use WiFi clients which support only the 2.4 GHz band. Preferably set up a different ESSID in this band, so that it will be occupied only by people who require it.
- DHCP servers embedded in cheap routers are generally crappy. Use a Linux machine as a DHCP server instead. This applies even to the translation network.
- Avoid NAT between parts of the network, it makes debugging of faults harder. Generally, try to keep things as simple as possible.

## Printing of translations

- The volume of task statements including translations is quite large. For example, English statements of Day 1 in 2023 had 16 pages. Multiplied by 360 contestants, it is almost 6000 pages for English only. Translations will have similar volume.
- We suggest having at least 3 printers capable of printing at least 40 pages per minute.
- The printers should be accessible from the translation network and also from the translation server.
- The translation server prints all papers for a single contestant as a single print job with a banner sheet containing the ID of the contestant and expected contents of their envelope.

## Assorted notes

- 2015: Clearing the database between day1 and 2 caused submission ids to be reused, and id clashes in RWS. This can be avoided by using the same database, or simply change all ids in day1 to avoid clashes (e.g., prepending "A"). This will be fixed in the CMS repository.
- 2015: Java requires a large memory margin, otherwise it runs the garbage collector too often, which leads to slowdown. We used 2GB for all tasks.

## Notes from 2020 and 2021 (online IOI format)

- There is an implicit limit of 100 workers assumed by CMS in `cms/grading/Sandbox.py`, do update the `box_id` allocation method when using more than 100 workers.
- When there is a huge number of workers, CMS's `EvaluationService` becomes a bottleneck when distributing tasks to workers. It was found that running `EvaluationService` on a host with faster CPU and low latency to the postgres database improved the overall regrading time. In IOI 2020 and 2021, AWS z1d instances were specifically selected for the `EvaluationService`.
- CMS workers downloads testcases from the postgres database instance. When workers are located remotely (e.g. in a separate AWS region) from the database instance, it takes the workers a significant amount of time to retrieve these testcases. A potential solution is to pre-cache all the testcases on every worker instance before the start of the contest. This can be sped up by first having a worker with all the testcases pre-cached, then copying the fs-cache directory to remote workers. The fs-cache directory could be found at `/var/local/cache/cms/fs-cache-shared/`. 
- Browser histories are useful information that could be logged/stored for use in handling appeals.
- In IOI 2021, using AWS Global Accelerator improved latency and bandwidth to various contest sites significantly, albeit at an increased cost to the host. In IOI 2020, when it was not used, ping latencies to some contestant VMs averaged at 700-900ms consistently. In IOI 2021, when AWS Global Accelerator was used, the highest ping latencies observed were less than 500ms. Do note that this could also be due to improved connectivity on the part of each country's local setup, but its worth pointing out that the host of IOI 2021 have had positive experiences with AWS Global Accelerator.
- The HTC requested HSC to design the Practice Contest such that it has a greater grading workload as compared to the shortlisted problem sets for Day 1 and Day 2. This was very useful as HTC could use the Practice Contest as a dry-run and estimate what to expect for Day 1 and Day 2.
- Tinc was chosen as the VPN solution for connecting Contestant VMs to the contest infrastructure on AWS.
- Ansible was extensively used to configure CMS deployments as well as contestant VMs.
