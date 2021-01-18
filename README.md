# Automatic PIA VPN Port Forwarding for OpenWRT

This repository is a fork of [pia-foss/manual-connections](https://github.com/pia-foss/manual-connections) and contains documentation on how initiate port forwarding automatically upon establishing an OpenVPN tunnel from an [OpenWRT](https://openwrt.org)-based device for subscribers of Private Internet Access VPN.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Procedure Overview](#procedure-overview)
- [Script Overview](#script-overview)
- [Dependencies](#dependencies)                                                                                                              
- [Setup](#setup)
- [Usage](#usage)
- [License](#license)

## Prerequisites
The procedure described below was tested on a Netgear R7800 router running OpenWrt 19.07.5, where PIA VPN support was set up following
[this](https://www.privateinternetaccess.com/helpdesk/guides/routers/lede-19-07-2-openvpn-setup-from-config-file) official guide.

This means
 * there is a specific PIA OpenVPN profile which is marked enabled on the OpenVPN LuCI page
 * the OpenVPN LuCI page is used to start/stop PIA VPN connections using the `start`/`stop` buttons

## Procedure Overview
The [parent repository](https://github.com/pia-foss/manual-connections) contains scripts that provide the following capabilities:
 * find the best PIA server based on the signal time
 * get a token for VPN authentification
 * establish PIA VPN connection using OpenVPN or WireGuard
 * enable port forwarding for an established connection

The scripts in this repository let you set up OpenWRT in a way that once you establish your PIA VPN connection from the OpenVPN page in LuCI,
the port forwarding request is performed automatically and you see the forwarded port in the System Log.

These scripts **do not** automatically set up any firewall port forwardings.
In the current state of affairs once I know the forwarded port, I set up the firewall rule manually for a specific device on the network.
Feel free to automate this step for your needs.

## Script Overview
Below I will only list the scripts that were modified or added to the [parent repository](https://github.com/pia-foss/manual-connections).
 * [get_region_and_token.sh](get_region_and_token.sh)
   * Without any parameters this script finds the closest PIA-enabled region based on signal time and lists its servers for each protocol.
   * Adding your PIA credentials to environment  `PIA_USER` and `PIA_PASS` will allow the script to also get a VPN authentication token for the region.
   * The script modification in this repository also accepts the `PIA_COUNTRY` variable to limit the region selection, since, as mentioned in [#prerequisites](prerequisites), we already have a region-based connection profile of choice.
 * [get_token.sh](get_token.sh): This script only outputs the VPN authentication token based on the required variables `PIA_USER` and `PIA_PASS`. When provided the `PIA_COUNTRY` variable as well, the region will be selected based on its value; otherwise by signal latency.
 * [port_forwarding.sh](port_forwarding.sh): Enables you to add Port Forwarding to an existing VPN connection. Adding the environment variable `PIA_PF=true` to any of the previous scripts will also trigger this script.
 * [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf): A Procd script that kicks off the port forwarding request upon detecting a newly established PIA VPN connection.

## Dependencies
In order for the scripts to work, you will need the following packages:
 * `curl`
 * `jq`
 * `bash`
 * `xargs` (the full version, not the one that comes pre-installed as part of BusyBox)
 * (`git`)

I am writing this list after having set up the process, so it is very likely that I miss a dependency or two.

## Setup
1. Clone this repository into some location such as `/usr/local/pia`:
```
git clone git@github.com:deletio/openwrt-pia-pf.git
```
2. Set up the [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf) script.
   1. PIA credentials. In my case the PIA OpenVPN profile refers to the `/etc/openvpn/pia.auth` file for credentials, so the same file is used in the [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf) script. It only has the user name on the first line and the password on the second. Either make sure you have the same file in the same location or edit the `PIA_USER` and `PIA_PASS` variables at the top of the [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf) script according to your setup.
   2. Region. After the authentication variables the [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf) script has the line `PIA_COUNTRY="Japan"`, where you should change the variable value to the region you are connecting to. The correct region name can be obtained by running the [get_region_and_token.sh](get_region_and_token.sh) script without parameters.
   3. Edit the line starting with `path_to_scripts=` to point the variable to the actual location of the repository.
   4. VPN interface name. **Importantly**, if your PIA VPN interface name is not `tun0`, change it on the line that starts with `vpn_interface=`.
3. Copy or the [hotplug.d-iface/99-pia-pf](hotplug.d-iface/99-pia-pf) script or create a link to it in `/etc/hotplug.d/iface`. Note that the script name starts with `99` in order for it to be lexicographically last among the scripts in that directory and therefore exeute after all others. It may not be important, but worth keeping in mind.

## Usage

Bring up the PIA VPN tunnel from the OpenVPN LuCI page and watch the System Log page (Status -> System Log) for messages prefixed with `user.notice hotplug`.
If everything goes smooth, you shoud see a sequence of messages of the following kind:
```
user.notice hotplug: PIA VPN interface tun0 is up. Trying to get the PIA host and token.
user.notice hotplug: PIA_USER=p0123456 PIA_PASS=xxx PIA_COUNTRY=Japan /usr/local/pia/manual-connections/get_token.sh
...
user.notice hotplug: Gateway is 87.654.32.1. Obtained host tokyo123 and token
user.notice hotplug: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
user.notice hotplug: Starting the port forwarding script with a 10s time limit.
user.notice hotplug: PF_GATEWAY=87.654.32.1 PF_HOSTNAME=tokyo123 PIA_TOKEN="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" /usr/local/pia/manual-connections/port_forwarding.sh &
user.notice hotplug: The forwarded port is 77777.
```
Use the port number to set up port forwarding between zones in the firewall settings (Network -> Firewall).

### Good to Know
 * In order to keep the forwarded port binding on the server side, the [port_forwarding.sh](port_forwarding.sh) script will be running in the background and sending corresponding requests to the PIA server every 15 minutes.
 * The forwarded port will be written to the `pia_forwarded_port` file in the location of the repository.
 * Upon termination of the VPN connection the port forwarding script will be terminated and the `pia_forwarded_port` file will be removed.

## License
This project is licensed under the [MIT (Expat) license](https://choosealicense.com/licenses/mit/), which can be found [here](/LICENSE).
