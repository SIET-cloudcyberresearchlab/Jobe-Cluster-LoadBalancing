# Jobe Cluster-Based Online Code Evaluation System

> Documentation for a Jobe-based code execution cluster with **1 master
> node and 4 worker nodes**. This guide covers planning, installation,
> networking, deployment checks, performance tuning, monitoring,
> testing, and troubleshooting.

**Important:** Jobe is a code execution service commonly used by
CodeRunner. A master/worker cluster is not automatically created merely
by installing Jobe on several machines. The master-to-worker dispatch
layer may be provided by your application, a reverse proxy, a
queue/scheduler, or custom scripts. Follow the sections below that match
your actual deployment, and do not assume a command or configuration
file exists until you verify it on your systems.

------------------------------------------------------------------------

## Table of Contents

-   [1. System Summary](#1-system-summary)
-   [2. Architecture](#2-architecture)
-   [3. Hardware and Software](#3-hardware-and-software)
-   [4. Plan the Network](#4-plan-the-network)
-   [5. Prepare Every Machine](#5-prepare-every-machine)
-   [6. Configure Hostnames and
    Connectivity](#6-configure-hostnames-and-connectivity)
-   [7. Install Jobe](#7-install-jobe)
-   [8. Configure the Master and Worker
    Design](#8-configure-the-master-and-worker-design)
-   [9. Secure the Execution Service](#9-secure-the-execution-service)
-   [10. Compilation and Runtime
    Configuration](#10-compilation-and-runtime-configuration)
-   [11. Performance Optimization](#11-performance-optimization)
-   [12. Monitoring and Logs](#12-monitoring-and-logs)
-   [13. Testing and Validation](#13-testing-and-validation)
-   [14. Benchmarking](#14-benchmarking)
-   [15. Backup and Recovery](#15-backup-and-recovery)
-   [16. Troubleshooting](#16-troubleshooting)
-   [17. Operational Checklist](#17-operational-checklist)
-   [18. Repository Documentation
    Layout](#18-repository-documentation-layout)

------------------------------------------------------------------------

## 1. System Summary

This project distributes submitted-program compilation and execution
across multiple machines. A coordinating component receives or routes
requests, while worker machines perform code execution. Distributing
jobs can increase throughput when several users submit programs at the
same time.

### What Jobe does

Jobe provides a service for compiling and running programs in a
controlled environment. It is often used as the execution backend for an
online programming assessment system. The user interface, assessment
logic, job queue, and cluster dispatch behavior may be supplied by other
components.

### What this cluster is intended to do

-   Receive program-execution requests from the connected application.
-   Direct work to one of four worker nodes through the cluster's
    dispatch mechanism.
-   Compile and execute supported languages on a worker.
-   Return output, errors, and execution status to the calling
    application.
-   Keep temporary compilation files from filling persistent storage.
-   Provide visibility into CPU, memory, disk, and service health.

### Current hardware details

The project has been described as **one master and four workers**, with
**16 GB RAM and 32 GB SD-card storage reported for the machines**.
Confirm whether those figures apply to every node before publishing them
as a definitive inventory.

  ----------------------------------------------------------------------------
  Node                  Count             RAM         Storage Main
                                                              responsibility
  ----------- --------------- --------------- --------------- ----------------
  Master                    1  16 GB reported   32 GB SD card Coordination or
                                                     reported request routing,
                                                              according to the
                                                              deployed design

  Worker                    4      16 GB each   32 GB SD card Compilation and
                                     reported   each reported program
                                                              execution

  Total                     5   Verify actual   Verify actual Distributed code
                                    inventory       inventory execution
  ----------------------------------------------------------------------------

> Do not claim a specific throughput, percentage improvement, or
> high-availability guarantee without benchmark results. A cluster with
> multiple workers can improve concurrent capacity, but actual
> performance depends on CPU, networking, scheduling, sandboxing, and
> workload.

------------------------------------------------------------------------

## 2. Architecture

### Logical architecture

``` mermaid
flowchart TD
    U[Student or Client Application] --> A[Application / Jobe API Client]
    A --> M[Master / Dispatch Layer]
    M --> W1[Worker 1: Jobe execution service]
    M --> W2[Worker 2: Jobe execution service]
    M --> W3[Worker 3: Jobe execution service]
    M --> W4[Worker 4: Jobe execution service]
    W1 --> M
    W2 --> M
    W3 --> M
    W4 --> M
    M --> A
    A --> U
```

The diagram describes the intended request path. The exact
implementation may differ. In particular, **standard Jobe installations
on their own do not necessarily provide a central scheduler that
automatically distributes requests among four independent Jobe
servers**. Identify the component that selects a worker in your
deployment and document its actual configuration.

### Responsibilities

**Master / coordination layer** - Receives requests from the
application, if the design uses the master as the entry point. - Selects
a worker or forwards requests to the configured execution service. -
Collects and returns results. - Exposes logs and health information for
the components it actually manages.

**Worker nodes** - Run the Jobe execution service. - Have the compilers
and interpreters required by the supported languages. - Execute
submissions under configured restrictions. - Return the result to the
caller or dispatch layer.

### Request sequence

``` mermaid
sequenceDiagram
    participant C as Client/Application
    participant M as Master/Dispatcher
    participant W as Selected Worker
    C->>M: Submit execution request
    M->>M: Select worker (if dispatcher exists)
    M->>W: Forward request
    W->>W: Compile and execute in sandbox
    W-->>M: Output, errors, status
    M-->>C: Return result
```

### Important design distinction

A machine called "master" is not automatically a load balancer. A
working cluster needs a real routing/dispatch mechanism, such as: - An
application that selects one worker endpoint per request. - A queue and
worker-consumer design. - A reverse proxy or load balancer, if the
request model is compatible with it. - A custom integration that calls a
selected Jobe server.

Record which mechanism is used in this project. Do not add a second
dispatcher if your application already handles worker selection.

------------------------------------------------------------------------

## 3. Hardware and Software

### Hardware

Recommended baseline for this deployment:

-   1 master/coordinator machine.
-   4 worker machines.
-   Network connectivity between the nodes.
-   16 GB RAM and 32 GB SD-card storage as reported; verify per node.
-   Reliable power and cooling.
-   A backup path for configuration files and monitoring data.

An SD card can be slower and less durable than an SSD under sustained
write workloads. Monitor disk latency, free space, and card health where
supported.

### Operating system

The commands in this guide use Debian/Ubuntu-style Linux. Package names
and service paths can differ between distributions and Jobe versions.

Check the operating system on each node:

``` bash
cat /etc/os-release
```

This prints the distribution and version so the correct installation
instructions can be selected.

### Inspect CPU, memory, disk, and kernel

Run these commands on the master and on every worker:

``` bash
hostname
lscpu
nproc
free -h
df -h
uname -a
```

-   `hostname` shows the machine's current name.
-   `lscpu` describes the processor and core/thread layout.
-   `nproc` reports the number of processing units available to the
    current process.
-   `free -h` summarizes memory and swap in readable units.
-   `df -h` reports filesystem capacity and free space.
-   `uname -a` prints kernel and system information.

Keep the outputs in your private deployment notes. Remove public IP
addresses, usernames, tokens, and other sensitive information before
publishing diagnostic output.

### Software inventory

Depending on the deployment, you may need:

-   Jobe and its documented dependencies.
-   GCC/G++ for C and C++.
-   Java runtime/compiler if Java submissions are supported.
-   Python if Python submissions are supported.
-   Web server or application runtime if the master hosts the calling
    application.
-   SSH for administration.
-   Monitoring utilities such as `htop`, `sysstat`, and `curl`.

Install only the components required by your chosen Jobe version and
supported language set. Follow the official Jobe installation
instructions for the exact release.

------------------------------------------------------------------------

## 4. Plan the Network

Use stable private addresses or reliable DNS names for all nodes. The
addresses below are examples only; replace them with the real addresses
on your network.

  Role       Example hostname   Example private IP
  ---------- ------------------ --------------------
  Master     `jobe-master`      `192.168.1.10`
  Worker 1   `jobe-worker1`     `192.168.1.11`
  Worker 2   `jobe-worker2`     `192.168.1.12`
  Worker 3   `jobe-worker3`     `192.168.1.13`
  Worker 4   `jobe-worker4`     `192.168.1.14`

### Network diagram

``` text
                 Student / Application
                          |
                    Master / Router
                          |
             Private cluster network
        __________|_________|_________|_________
       |          |         |         |         |
    Worker 1   Worker 2  Worker 3  Worker 4   (services)
```

### Choose the correct addresses

On each node, run:

``` bash
ip -br address
ip route
```

-   `ip -br address` gives a compact list of interfaces and addresses.
-   `ip route` shows the routing table and default gateway.

Do not copy the example IP addresses blindly. Use addresses reserved for
the actual network. If you configure static addresses, make sure they do
not conflict with DHCP assignments or other devices. Network
configuration procedures vary by operating system and network manager.

### Name resolution

If the environment does not have working internal DNS, a consistent
`/etc/hosts` file can be used on the relevant nodes. Example only:

``` text
192.168.1.10  jobe-master
192.168.1.11  jobe-worker1
192.168.1.12  jobe-worker2
192.168.1.13  jobe-worker3
192.168.1.14  jobe-worker4
```

Edit it with administrator privileges:

``` bash
sudo nano /etc/hosts
```

Save the file, then test name resolution:

``` bash
getent hosts jobe-master
getent hosts jobe-worker1
```

Use your actual addresses and hostnames. This file must be consistent on
each machine that relies on these names.

------------------------------------------------------------------------

## 5. Prepare Every Machine

Perform the operating-system preparation on all five machines.

### 5.1 Update package metadata

``` bash
sudo apt update
```

This refreshes the package index. It does not, by itself, upgrade
installed packages.

### 5.2 Upgrade installed packages

``` bash
sudo apt upgrade
```

Review the changes before confirming. On production systems, schedule
upgrades and reboots carefully.

### 5.3 Install basic administration tools

``` bash
sudo apt install -y openssh-client openssh-server curl wget git htop
```

-   `openssh-client` provides SSH client tools.
-   `openssh-server` allows remote SSH access.
-   `curl` and `wget` can test HTTP endpoints and retrieve files.
-   `git` manages source repositories.
-   `htop` displays processes and resource usage.

Some packages may already be installed. If your distribution does not
provide a package under the same name, consult its package
documentation.

### 5.4 Confirm SSH service status

``` bash
sudo systemctl status ssh
```

On distributions using a different service name, check the
distribution's SSH service documentation. If SSH is installed but
stopped, start it:

``` bash
sudo systemctl enable --now ssh
```

This enables the service at boot and starts it now. Only expose SSH to
trusted administration networks, and use key-based authentication where
practical.

### 5.5 Set clear hostnames

Set a unique hostname on each machine. For example, on the first worker:

``` bash
sudo hostnamectl set-hostname jobe-worker1
```

Use `jobe-master`, `jobe-worker2`, and so on on the other machines.
Changing a hostname can affect monitoring, certificates, scripts, and
name resolution, so update related configuration as needed.

Verify:

``` bash
hostnamectl
```

------------------------------------------------------------------------

## 6. Configure Hostnames and Connectivity

### 6.1 Test IP connectivity

From the master, test each worker's actual IP:

``` bash
ping -c 4 192.168.1.11
ping -c 4 192.168.1.12
ping -c 4 192.168.1.13
ping -c 4 192.168.1.14
```

Replace these sample addresses. Some networks block ICMP, so a failed
ping does not always mean the application port is unreachable.

### 6.2 Test SSH access

``` bash
ssh your_user@jobe-worker1
```

Replace `your_user` with a real account on the worker. If login fails,
inspect the SSH error and the server logs before changing permissions or
firewall rules.

### 6.3 Optional SSH keys for administration

On the administrator's trusted machine, create a key if one does not
already exist:

``` bash
ssh-keygen -t ed25519
```

Choose an appropriate key location and passphrase. Then copy the public
key to a worker:

``` bash
ssh-copy-id your_user@jobe-worker1
```

Repeat for the other nodes if needed. This is for administration; **do
not assume Jobe requires SSH between the master and workers**. If your
dispatch layer communicates over HTTP(S), configure and secure that
mechanism instead.

Test:

``` bash
ssh your_user@jobe-worker1 hostname
```

The command should print the remote hostname after successful
authentication.

### 6.4 Firewall planning

Permit only the ports required by the design: - SSH from trusted
administration hosts. - The Jobe API port on worker nodes from the
master/application that needs it. - Web/application ports only where
required. - Monitoring ports only from the monitoring system.

Do not expose a code-execution API directly to the public internet
unless it is intentionally designed and protected for that use. Jobe API
ports and paths depend on the deployed version and web-server
configuration; inspect the actual configuration rather than guessing a
port.

------------------------------------------------------------------------

## 7. Install Jobe

Jobe installation details vary by release and operating system. Use the
official Jobe documentation and installation package for the exact
version you intend to run. Do not run an unverified installation script
as root.

### 7.1 Decide where Jobe runs

For a worker-execution architecture, each worker that receives execution
requests needs a correctly configured Jobe service and the runtimes for
the languages it supports. The master may run Jobe too, but that depends
on your design; do not install it on the master just because the machine
is called "master."

### 7.2 Record the version

After installation, identify the installed Jobe version using the method
documented for that release. Also record the OS version and installed
language runtimes:

``` bash
cat /etc/os-release
gcc --version
g++ --version
python3 --version
java -version
```

Some commands may not be installed if those languages are not supported.
Install the relevant compiler/runtime only if required.

### 7.3 Follow the release-specific installation guide

For each node designated as a Jobe execution server:

1.  Review the official installation instructions for your OS and Jobe
    release.
2.  Install the listed prerequisites.
3.  Install Jobe using the documented method.
4.  Configure the web server and Jobe endpoint as directed.
5.  Install the language runtimes needed by your assessment platform.
6.  Configure the sandbox and resource limits.
7.  Restrict network access to the service.
8.  Start or reload the required services using the service names
    actually installed.
9.  Verify the endpoint locally before integrating the node with the
    master/dispatcher.

Avoid copying configuration paths from an unrelated Jobe release. First
identify the installed paths:

``` bash
systemctl list-units --type=service --all | grep -i jobe
systemctl list-units --type=service --all | grep -E 'apache|nginx'
```

These commands search the system's service list. A service may have a
different name or may not be managed by systemd in a particular
deployment.

### 7.4 Verify a service

After identifying the actual service name, inspect it:

``` bash
sudo systemctl status SERVICE_NAME
```

Replace `SERVICE_NAME` with the service found on your machine. Do not
literally use `SERVICE_NAME` as a service name.

For recent logs:

``` bash
sudo journalctl -u SERVICE_NAME -n 100 --no-pager
```

Some web-server logs are stored in files rather than the system journal.
Check the configured logging destination if no relevant entries appear.

### 7.5 Test the endpoint

Use the exact URL and API method documented for your Jobe version and
integration. For example, `curl` can check whether a known health URL
responds:

``` bash
curl -i http://127.0.0.1/
```

This only tests the root URL; it does **not** prove that a Jobe API
request succeeds. Use the documented API test request for your
installation and verify a harmless sample program before declaring the
service ready.

------------------------------------------------------------------------

## 8. Configure the Master and Worker Design

This section connects the Jobe execution servers to the application that
dispatches work. The exact configuration depends on how this project was
implemented.

### 8.1 Inventory the current dispatch mechanism

Answer these questions and record the answers in your deployment notes:

1.  What receives the original student request?
2.  Does the application call Jobe directly, or does it call a master
    endpoint?
3.  What component chooses one of the four worker addresses?
4.  Is worker selection round-robin, queue-based, least-loaded, or
    something else?
5.  How are unavailable workers detected?
6.  Where are endpoint URLs and credentials configured?
7.  How are timeouts and failed requests handled?

Do not claim that Jobe automatically registers worker nodes unless your
installed version or custom integration explicitly implements that
behavior.

### 8.2 Example endpoint inventory

Use a private configuration table to document the real endpoints. The
addresses and paths below are placeholders.

  Node       Example endpoint         Status
  ---------- ------------------------ -----------------------
  Worker 1   `http://192.168.1.11/`   Replace with real URL
  Worker 2   `http://192.168.1.12/`   Replace with real URL
  Worker 3   `http://192.168.1.13/`   Replace with real URL
  Worker 4   `http://192.168.1.14/`   Replace with real URL

Use HTTPS where appropriate and restrict access to trusted systems. Do
not commit passwords, API tokens, private keys, or other secrets to Git.

### 8.3 Dispatch behavior

If the application already selects a worker, document its existing
policy. If no dispatch layer exists, implement or configure one before
describing the setup as a working cluster. The dispatcher should
ideally: - Send each request to one configured worker. - Apply request
timeouts. - Handle failed workers without endlessly retrying unsafe
requests. - Record which worker processed a request. - Avoid sending
more work to a worker known to be unavailable. - Return a clear error if
no worker is available.

### 8.4 Verify all workers independently

From a trusted machine, test each worker's documented API endpoint using
the same harmless test request. Record success, response time, and any
errors. Test each worker directly before testing through the master;
otherwise, a failure in routing can be confused with a failure in Jobe
itself.

------------------------------------------------------------------------

## 9. Secure the Execution Service

Executing untrusted code is a security-sensitive operation. A program
may consume CPU and memory, create files, attempt to access the network,
or exploit a misconfiguration.

### 9.1 Use the supported sandbox

Use the sandbox mechanism supported by the installed Jobe version and
follow its documented installation and configuration instructions. Do
not assume that running a program as an ordinary web-server user is
sufficient isolation.

### 9.2 Apply resource limits

Configure and verify appropriate limits for: - CPU time and wall-clock
execution time. - Memory. - Output size. - Number of
processes/threads. - File size and file access. - Temporary storage. -
Network access, preferably disabled for submissions unless explicitly
needed.

The actual supported controls and configuration syntax depend on Jobe,
the sandbox, and the operating system.

### 9.3 Network access

Worker execution nodes should not be able to reach sensitive internal
services unnecessarily. Use firewall rules and network segmentation to
limit access. Do not expose administrative ports to student networks.

### 9.4 Keep software patched

Update the operating system, Jobe, compilers, language runtimes, and
sandbox components through a controlled maintenance process. Test
updates before using them for a live examination.

### 9.5 Secrets and repository hygiene

Never commit: - Passwords or API tokens. - SSH private keys. -
Production credentials. - Internal secrets in logs. - Personal data from
student submissions.

Use environment-specific configuration outside the public repository and
limit permissions on files containing secrets.

------------------------------------------------------------------------

## 10. Compilation and Runtime Configuration

### 10.1 Compilation vs. execution optimization

Compiler flags such as `-O2` mostly affect the generated program's
runtime performance; they do not guarantee faster compilation. In some
cases, stronger optimization can make compilation slower. Choose flags
based on the assessment requirements and benchmark both compile time and
runtime.

### 10.2 Example C compilation

For a general-purpose C submission, a compiler command may resemble:

``` bash
gcc -O2 -pipe -std=c17 source.c -o program
```

For C++:

``` bash
g++ -O2 -pipe -std=c++20 source.cpp -o program
```

These are illustrative commands, not a direction to edit a Jobe
configuration file blindly. Configure the compiler options using the
mechanism supported by your Jobe integration.

-   `-O2` enables a broad set of optimizations.
-   `-pipe` uses pipes between compiler stages where supported; its
    benefit is often small on modern systems.
-   `-std=c17` or `-std=c++20` selects a language standard. Choose the
    standard required by your course or contest.
-   `-o program` names the output executable.

### 10.3 Avoid unsafe or incompatible flags

-   `-Ofast` may enable transformations that change strict
    language-standard behavior.
-   `-march=native` may generate code specific to the worker's CPU.
    Avoid it if workers may have different processor capabilities.
-   Do not increase limits globally just to make one submission pass;
    set limits based on the assessment policy.

### 10.4 Language runtimes

Install only the language compilers/interpreters your platform supports.
For example, Python submissions generally use an interpreter, while
C/C++ submissions require compilation. Java commonly requires a JDK to
compile source code. Ensure all workers have compatible versions so
results do not vary unexpectedly.

------------------------------------------------------------------------

## 11. Performance Optimization

Optimize using measurements rather than assumptions. The likely
bottleneck may be CPU, memory, storage, the scheduler, or network
latency.

### 11.1 Confirm `/tmp` uses tmpfs

On the worker, run:

``` bash
mount | grep ' /tmp '
```

If the output contains a line similar to:

``` text
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,...)
```

then `/tmp` is mounted as a memory-backed filesystem. The precise mount
options vary. If there is no output, `/tmp` may use another filesystem
or may be mounted differently.

**Observed configuration:** the output shared for this project showed
`/tmp` mounted as `tmpfs`. This is useful for temporary-file I/O, but
does not by itself prove that compilation is faster.

### 11.2 Understand tmpfs memory usage

A tmpfs filesystem uses RAM and may be eligible to use swap, depending
on the system. Its capacity is a limit, not memory reserved exclusively
for `/tmp`. Files in it compete with Jobe, the operating system, and
running submissions for memory.

Check memory and temporary filesystem usage:

``` bash
free -h
df -h /tmp
```

Check which directories are consuming space:

``` bash
sudo du -xhd1 /tmp 2>/dev/null | sort -h
```

Use care before deleting temporary files: a running compilation or
execution may still depend on them. Do not add broad cleanup commands to
cron without understanding Jobe's temporary-directory lifecycle.

### 11.3 Do not blindly increase tmpfs size

A mount option such as `size=4G` is only an example, not a universal
recommendation. On a 16 GB machine, several concurrent jobs can consume
significant RAM. Increasing the tmpfs limit does not create more
physical memory.

Before changing `/etc/fstab` or remounting `/tmp`: 1. Measure peak
memory use under realistic concurrency. 2. Check how much temporary
space jobs need. 3. Preserve any required `nosuid`, `nodev`, and other
security options. 4. Test changes on one worker. 5. Ensure the machine
can reboot cleanly with the new configuration.

### 11.4 CPU and concurrency

Check the number of available CPUs:

``` bash
nproc
lscpu
```

Monitor load:

``` bash
uptime
```

High load does not automatically mean the system is unhealthy; compare
it with CPU count, CPU utilization, run queue, and response time. Do not
assume that one job per CPU core is always optimal. Compilers,
memory-heavy programs, and sandbox processes have different resource
profiles.

### 11.5 Memory pressure

``` bash
free -h
vmstat 1
```

In `vmstat`, inspect free memory, swap activity, runnable processes, and
I/O wait. The first output row often summarizes activity since boot; use
subsequent rows to evaluate current behavior. Sustained swap-in/swap-out
can indicate memory pressure.

### 11.6 SD-card storage

``` bash
df -h
df -i
```

-   `df -h` checks filesystem capacity.
-   `df -i` checks inode availability.

Check storage performance during a planned test window with suitable
tools such as `iostat` if installed. Avoid running destructive disk
benchmarks on a production SD card. Logs and persistent files still use
storage even when `/tmp` is tmpfs.

### 11.7 Keep logs and temporary files controlled

Review Jobe and web-server log rotation. Do not delete active logs or
temporary job directories indiscriminately. Use the log rotation and
cleanup mechanism appropriate to the installed services.

### 11.8 Balance the workers

A dispatcher can improve throughput only if it sends work to functioning
workers and does not overload one node. Track per-worker job count,
failures, response time, CPU, and memory. If worker utilization differs
greatly, investigate the dispatch policy and hardware differences.

------------------------------------------------------------------------

## 12. Monitoring and Logs

Monitoring helps distinguish service failures from resource saturation.

### 12.1 Quick health checks

Run these commands on each node:

``` bash
uptime
free -h
df -h
vmstat 1 5
```

They provide load averages, memory usage, disk space, and a short sample
of system activity.

### 12.2 Process monitoring

``` bash
htop
```

Look for sustained CPU saturation, memory pressure, unexpected
processes, and excessive concurrent compiler or execution processes.
Exit `htop` with `q`.

### 12.3 Service status

After identifying the actual service names:

``` bash
systemctl --type=service --state=running
sudo systemctl status SERVICE_NAME
```

Use the correct service name for Jobe, Apache, Nginx, or the application
component installed on that node.

### 12.4 Logs

``` bash
sudo journalctl -u SERVICE_NAME -n 100 --no-pager
```

To inspect recent system errors:

``` bash
sudo journalctl -p err -n 100 --no-pager
```

Web servers and applications may also write logs to configured files.
Consult their configuration to find the correct paths.

### 12.5 Network checks

``` bash
ip -br address
ip route
ss -lnt
```

`ss -lnt` lists listening TCP sockets. Verify that the expected service
is listening on the intended interface and port, without exposing it to
untrusted networks.

### 12.6 Monitoring dashboard

If Prometheus and Grafana are part of your actual deployment,
document: - Which exporter runs on each machine. - Which metrics are
collected. - Where Prometheus scrapes metrics. - Which dashboards are
used. - Which alerts indicate worker or service failure.

Do not claim Prometheus or Grafana is installed unless it is. A
monitoring stack is optional and separate from the basic Jobe execution
path.

------------------------------------------------------------------------

## 13. Testing and Validation

Run tests in a controlled environment with harmless sample programs.
Never use untrusted stress tests against a live assessment service.

### 13.1 Verify all five machines

On every machine:

``` bash
hostname
free -h
df -h
```

Confirm that each hostname matches the intended role and that sufficient
storage is available.

### 13.2 Verify worker service health

For each worker: 1. Check the Jobe service status. 2. Check its logs for
startup errors. 3. Test its API using the documented method for your
installed version. 4. Submit a harmless sample program directly to that
worker. 5. Confirm the returned result is correct.

### 13.3 Sample C program

Create a test file on a development machine:

``` c
#include <stdio.h>

int main(void) {
    printf("Hello from the Jobe worker!\n");
    return 0;
}
```

Expected output:

``` text
Hello from the Jobe worker!
```

Submit it through the supported Jobe API or the connected application.
Do not assume that uploading a source file to the server's filesystem is
equivalent to submitting a Jobe API request.

### 13.4 Sample Python program

``` python
print("Hello from the Jobe worker!")
```

Expected output:

``` text
Hello from the Jobe worker!
```

Use this test only if Python is enabled in the deployment.

### 13.5 Test error handling

Use harmless test submissions to check: - Invalid syntax produces a
compilation or interpreter error. - A program that prints output returns
that output. - A program that exits with an error is reported
appropriately. - An intentionally time-consuming test is stopped by the
configured limit. - An oversized-output test is stopped or limited
according to policy.

Keep resource-limit tests small and run them only in a controlled
environment.

### 13.6 Test the cluster path

After direct worker tests succeed: 1. Submit a test through the
master/application endpoint. 2. Confirm that a configured worker
actually receives the request. 3. Check dispatcher logs or request logs
to identify the selected worker. 4. Repeat requests and verify that the
observed distribution matches the configured policy. 5. Stop or isolate
one worker during a maintenance test and verify the documented failure
behavior. 6. Restore the worker and verify that it becomes available
again.

If all requests still go to one worker, investigate the dispatch
configuration rather than assuming load balancing is active.

### 13.7 Concurrent testing

Start with a small number of harmless concurrent requests, then increase
gradually while monitoring CPU, memory, `/tmp`, disk space, errors, and
latency. Stop if the service becomes unstable. Use a dedicated test
environment for larger stress tests.

------------------------------------------------------------------------

## 14. Benchmarking

A performance claim should be supported by repeatable measurements.

### 14.1 What to measure

Record at least: - End-to-end request latency. - Compilation time, where
available. - Program execution time. - Successful jobs and failed
jobs. - Jobs completed per minute. - CPU and memory use on each
worker. - Disk and temporary-filesystem usage. - Number of simultaneous
requests.

### 14.2 Fair comparison

To compare one worker with four workers: 1. Use the same sample programs
and limits. 2. Keep software versions and compiler flags consistent. 3.
Use the same client workload. 4. Warm up the system before recording
measurements. 5. Repeat each run several times. 6. Record errors and
resource usage, not only speed. 7. Clearly state whether measurements
are compile-only or end-to-end.

### 14.3 Results template

Replace the blank fields with real measured results. Do not publish
invented numbers.

  ----------------------------------------------------------------------------
  Test            Workers   Concurrent       Median   Jobs/minute   Error rate
                available     requests      latency               
  ---------- ------------ ------------ ------------ ------------- ------------
  Baseline              1      Fill in      Fill in       Fill in      Fill in

  Cluster               4      Fill in      Fill in       Fill in      Fill in
  ----------------------------------------------------------------------------

A four-worker cluster may increase total throughput, but a single
request may not become four times faster. The result depends on the
workload and the dispatcher.

------------------------------------------------------------------------

## 15. Backup and Recovery

### 15.1 What to back up

Back up the configuration needed to rebuild the system: - Jobe and
web-server configuration. - Dispatcher/application configuration. -
Worker endpoint inventory. - Monitoring configuration and dashboards, if
used. - Deployment scripts and documentation. - Any required application
data.

Do not back up secrets into the public Git repository. Keep secure
backups of secrets separately with access controls.

### 15.2 Store configuration in version control

Keep sanitized configuration examples and scripts in Git. Remove
credentials, tokens, private keys, and environment-specific secrets
before committing.

Check repository status:

``` bash
git status
```

Review the diff before committing:

``` bash
git diff
```

### 15.3 Recovery approach

A practical recovery sequence is: 1. Reinstall the supported operating
system. 2. Restore network and hostname configuration. 3. Install the
documented Jobe version and dependencies. 4. Restore sanitized service
configuration and securely restore required secrets. 5. Verify
sandboxing and resource limits. 6. Test the node directly. 7. Re-enable
it in the dispatch layer. 8. Monitor errors and latency after recovery.

Maintain a tested recovery procedure; a backup is only useful if it can
be restored.

------------------------------------------------------------------------

## 16. Troubleshooting

### Jobe service does not start

**Possible causes:** invalid configuration, missing dependency,
permission issue, incompatible package, or port conflict.

Check:

``` bash
sudo systemctl status SERVICE_NAME
sudo journalctl -u SERVICE_NAME -n 100 --no-pager
```

Replace the placeholder with the real service name. Read the first
relevant error in the logs and fix the underlying cause rather than
repeatedly restarting the service.

### Master cannot reach a worker

**Possible causes:** incorrect IP/hostname, route problem, firewall
restriction, service not listening, or wrong API endpoint.

Check:

``` bash
getent hosts jobe-worker1
ip route
ss -lnt
```

Test the actual worker endpoint from the master with the correct URL and
API request. A successful ping alone does not prove that the Jobe API is
accessible.

### SSH reports "Permission denied"

**Possible causes:** incorrect username, wrong key, missing public key
on the worker, or server-side SSH policy.

Check the username and key being used. Review SSH logs on the worker. Do
not make private keys world-readable or disable authentication
safeguards to bypass the error.

### Compilation fails

**Possible causes:** unsupported language, missing compiler, invalid
source code, wrong compiler version, or incorrect flags.

Check the installed compiler:

``` bash
gcc --version
g++ --version
```

Read the compilation output returned by Jobe. Distinguish a source-code
error from a missing dependency or a Jobe configuration problem.

### Python or Java runtime is missing

Check the runtime on every worker that is expected to support it:

``` bash
python3 --version
java -version
```

Install the required runtime using the supported package source for your
OS, then retest the worker. Java compilation generally requires a JDK,
not only a runtime.

### `/tmp` fills up

Check:

``` bash
df -h /tmp
sudo du -xhd1 /tmp 2>/dev/null | sort -h
free -h
```

Find out which process or workload is consuming space. Use the service's
documented cleanup behavior. Do not delete active job directories
blindly.

### Worker runs out of memory

Check:

``` bash
free -h
vmstat 1 5
```

Reduce excessive concurrency, review per-job memory limits, and inspect
whether tmpfs usage is contributing to memory pressure. Do not solve the
issue only by increasing `/tmp` capacity.

### Disk space is low

Check:

``` bash
df -h
df -i
```

Inspect log growth, package caches, and persistent application data.
Clean only known-safe files using the appropriate package and
log-rotation tools. Do not remove unknown system files to free space.

### All requests go to one worker

**Possible causes:** no dispatcher, static endpoint configuration, a
faulty selection policy, or routing that sends every request to the same
endpoint.

Inspect the component that selects workers. Record the selected worker
for each test request and verify that the configured policy is actually
being applied.

### Worker is offline

Check the worker's power/network state, IP address, service status, and
logs. Confirm that the master uses the correct endpoint. If automatic
health checks and failover are implemented, test their behavior;
otherwise, document the manual recovery process.

### Results are slow

Measure end-to-end latency and separately inspect CPU, memory, disk, and
network. Compare direct worker requests with requests through the
master. This helps determine whether the bottleneck is compilation,
execution, scheduling, storage, or routing.

------------------------------------------------------------------------

## 17. Operational Checklist

### Before deployment

-   [ ] Confirm OS and Jobe versions.
-   [ ] Record CPU, RAM, storage, and IP address for all five nodes.
-   [ ] Confirm hostnames resolve correctly.
-   [ ] Confirm only required ports are reachable.
-   [ ] Verify sandbox and resource limits.
-   [ ] Verify each supported language on each worker.
-   [ ] Identify the actual dispatch/load-balancing component.
-   [ ] Remove secrets from repository files.

### Before a contest or lab

-   [ ] Check master and worker service status.
-   [ ] Check available RAM and disk space.
-   [ ] Check `/tmp` usage.
-   [ ] Submit one harmless test to each worker.
-   [ ] Submit one test through the normal application path.
-   [ ] Confirm worker selection and result reporting.
-   [ ] Check logs and monitoring alerts.
-   [ ] Ensure a recovery contact and procedure are available.

### After a contest or lab

-   [x] Review errors and failed jobs.
-   [x] Check disk and temporary-file usage.
-   [x] Review worker utilization and request latency.
-   [x] Preserve only the logs required by policy.
-   [x] Record observed bottlenecks and measured performance.
-   [x] Plan maintenance based on evidence.

------------------------------------------------------------------------

## 18. Repository Documentation Layout

A simple repository layout could be:

``` text
jobe-cluster/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── installation.md
│   ├── performance.md
│   ├── monitoring.md
│   └── troubleshooting.md
├── configs/
│   └── README.md
├── scripts/
│   └── README.md
└── screenshots/
    └── README.md
```

These directories are suggestions. Create only the files you actually
maintain. Do not commit live configuration files containing credentials
or internal secrets.

