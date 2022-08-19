**Table of contents**

- [Set up the lab](#set-up-the-lab)
  - [Start an ATD instance](#start-an-atd-instance)
  - [Load the EVPN lab on ATD](#load-the-evpn-lab-on-atd)
  - [Check the state of spine1](#check-the-state-of-spine1)
  - [Install the packages on devbox](#install-the-packages-on-devbox)
  - [Check the requirements on the switches](#check-the-requirements-on-the-switches)
  - [Check the inventory files](#check-the-inventory-files)
- [Test devices reachability using EAPI](#test-devices-reachability-using-eapi)
- [Test devices reachability](#test-devices-reachability)
- [Test devices](#test-devices)
  - [Run the tests](#run-the-tests)
  - [Fix the issue](#fix-the-issue)
  - [Re run the tests](#re-run-the-tests)
- [Collect commands output](#collect-commands-output)
- [Collect the scheduled show tech-support files](#collect-the-scheduled-show-tech-support-files)
- [Clear the list of MAC addresses which are blacklisted in EVPN](#clear-the-list-of-mac-addresses-which-are-blacklisted-in-evpn)
  - [Create 5 mac moves within 180 seconds](#create-5-mac-moves-within-180-seconds)
  - [Clear the blacklisted MAC addresses](#clear-the-blacklisted-mac-addresses)
- [Clear counters](#clear-counters)

Here's the instructions to use this repository with an ATD (Arista Test Drive) lab.

# Set up the lab 

## Start an ATD instance

Here's the ATD topology:

![images/atd_topology.png](images/atd_topology.png)

Login to the Arista Test Drive portal and start an instance.

## Load the EVPN lab on ATD

Load the EVPN lab on your ATD instance

![images/atd_configuration.png](images/atd_configuration.png)

This lab uses 2 spines and 2 leaves:

- Spine1 and spine2
- Leaf1 and leaf3

Leaf2 and leaf4 are not used.

Here's the EVPN lab topology:
![images/atd_evpn_lab_topology.png](images/atd_evpn_lab_topology.png)

The script configured the lab with the exception of leaf3:

- Leaves <-> spines interfaces are configured with an IPv4 address
- eBGP is configured between spines and leaves (underlay, IPv4 unicast address family)
- BFD is configured for the eBGP sessions (IPv4 unicast address family)
- 2 loopback interfaces are configured per leaf
- 1 loopback interface is configured per spine
- eBGP is configured between spines and leaves (overlay, EVPN address family, Loopback0)
- VXLAN is configured on the leaves (Loopback1)
- Default VRF only

## Check the state of spine1

ssh to spine1 and run some EOS commands to check the state

![images/atd_spine1.png](images/atd_spine1.png)

```text
spine1#show ip bgp summary
spine1#show bgp evpn summary
spine1#sh lldp neighbors
```

Some BGP sessions are not established.  
This is expected because Leaf3 is not yet configured.

## Install the packages on devbox

Use the devbox shell
![images/atd_devbox_shell.png](images/atd_devbox_shell.png)

Run these commands to clone the repository and to move to the new folder:

```shell
git clone https://github.com/ksator/network-test-automation.git
cd network-test-automation
```

Run this command to build the package [anta](../anta):

```shell
python setup.py build
```

Run this command to install:

- The package [anta](../anta) and its dependencies
- The packages required by these [scripts](../scripts)  
- These [scripts](../scripts) in `/usr/local/bin/`

```shell
sudo python setup.py install
```

Run this command to verify:

```bash
pip list
```

Run this commands on devbox to install some additional packages:

```bash
sudo apt-get install tree unzip -y
```

## Check the requirements on the switches

```text
spine1#show management api http-commands
```

## Check the inventory files

Run this command on devbox to check the inventory files:

```bash
ls examples/demo/inventory
```

# Test devices reachability using EAPI

Start a python interactive session on devbox:

```bash
arista@devbox:~$ python
```

Run these python commands:

```python
from jsonrpclib import Server
import ssl
ssl._create_default_https_context = ssl._create_unverified_context
USERNAME = "arista"
# use the device password of your ATD instance
PASSWORD = "aristatwfn"
IP = "192.168.1.10"
URL=f'https://{USERNAME}:{PASSWORD}@{IP}/command-api'
switch = Server(URL)
result=switch.runCmds(1,['show version'], 'text')
print(result[0]['output'])
exit()
```

# Test devices reachability

Run these commands on devbox:

```bash
check-devices-reachability.py --help
check-devices-reachability.py -i examples/demo/inventory/all.txt -u arista
```

# Test devices

## Run the tests

Run these commands on devbox:

```bash
check-devices.py --help
```

ATD uses cEOS or vEOS so we will skip the hardware tests.  
This lab doesnt use MLAG, OSPF, IPv6, RTC ... so we wont run these tests as well.  
Some tests can be used for all devices, some tests should be used only for the spines, and some tests should be used only for the leaves.  

```bash
ls examples/demo/inventory
```

```bash
ls examples/demo/tests
```

```bash
check-devices.py -i examples/demo/inventory/all.txt -t examples/demo/tests/all.yaml -o examples/demo/tests_result_all.txt -u arista
cat examples/demo/tests_result_all.txt
```

Some tests failed.  
This is expected because leaf3 is not yet configured.  

## Fix the issue

Lets configure leaf3 using EAPI.

```bash
python examples/demo/configure-leaf3.py
```

## Re run the tests

Lets re run all the tests.

```bash
check-devices.py -i examples/demo/inventory/all.txt -t examples/demo/tests/all.yaml -o examples/demo/tests_result_all.txt -u arista
cat examples/demo/tests_result_all.txt
```

# Collect commands output

Run these commands on devbox:

```bash
collect-eos-commands.py --help
more examples/demo/eos-commands.yaml
collect-eos-commands.py -i examples/demo/inventory/all.txt -c examples/demo/eos-commands.yaml -o examples/demo/show_commands -u arista
tree examples/demo/show_commands
more examples/demo/show_commands/192.168.0.10/text/show\ version
```

# Collect the scheduled show tech-support files

```text
spine1# sh running-config all | grep tech
spine1# bash ls /mnt/flash/schedule/tech-support/
```

Run these commands on devbox:

```bash
collect-sheduled-show-tech.py --help
collect-sheduled-show-tech.py -i examples/demo/inventory/all.txt -u arista -o examples/demo/show_tech
tree examples/demo/show_tech
unzip examples/demo/show_tech/spine1/xxxx.zip -d examples/demo/show_tech
ls examples/demo/show_tech/mnt/flash/schedule/tech-support/
ls examples/demo/show_tech/mnt/flash/schedule/tech-support/ | wc -l
```

```bash
spine1# bash ls /mnt/flash/schedule/tech-support/
```

# Clear the list of MAC addresses which are blacklisted in EVPN

## Create 5 mac moves within 180 seconds

Run this command alternately on host1 and host2 in order to create 5 mac moves within 180 seconds:

```bash
bash sudo ethxmit --ip-src=10.10.10.1 --ip-dst=10.10.10.2 -S 948e.d399.4421 -D ffff.ffff.ffff et1 -n 1
```

Leaf1 or leaf3 concludes that a duplicate-MAC situation has occurred (948e.d399.4421)

```text
leaf3#show mac address-table 
leaf3#show bgp evpn host-flap
leaf3#show logging | grep EVPN-3-BLACKLISTED_DUPLICATE_MAC
```

## Clear the blacklisted MAC addresses

Run this command on devbox to clear on devices the list of MAC addresses which are blacklisted in EVPN:

```bash
evpn-blacklist-recovery.py --help
evpn-blacklist-recovery.py -i examples/demo/inventory/all.txt -u arista
```

Verify:

```text
leaf3#show mac address-table 
leaf3#show bgp evpn host-flap
leaf3#show logging | grep EVPN-3-BLACKLISTED_DUPLICATE_MAC
```

# Clear counters

```bash
spine1#sh interfaces counters
```

Run these commands on devbox:

```bash
clear-counters.py --help
clear-counters.py -i examples/demo/inventory/all.txt -u arista
```

```bash
spine1#sh interfaces counters
```
