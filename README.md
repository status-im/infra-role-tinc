# Description

[Tinc](http://tinc-vpn.org/) is a Virtual Private Network (VPN) daemon that uses tunnelling and encryption to create a secure private network between hosts.
Each host has it's own public and private key which is used to authenticate them and encrypt the traffic.

__WARNING:__ This role __assumes__ that Consul is available under `localhost:8500`.

# Setup

Here are the core files defining te setup of `status.im` network:

* `/etc/tinc/status.im/` - Network configuration dir.
* `/etc/tinc/status.im/hosts/` - Contains config files for all connected hosts.
* `/etc/tinc/status.im/tinc.conf` - Main network configuration.
* `/etc/tinc/status.im/tinc-ip` - VPN IP address of the local Tinc peer.
* `/etc/tinc/status.im/tinc-up` - Script for creating the `tun0` interface.
* `/etc/tinc/status.im/tinc-down` - Script for destorying the `tun0` interface.
* `/etc/tinc/status.im/tinc-refresh` - Core script which configures the network.

# Refresh

In order to stay up-to-date with the rest of the network the Tinc server has to know about all of the hosts in the network and their public keys.

To achieve that we run the [`/etc/tinc/status.im/tinc-refresh`](/files/tinc-refresh) script which does the following:

1. Queries the [Consul](https://www.consul.io/) catalog for all Tinc peers across all DCs.
2. __OPTIONAL__: Assigns the current peer an address in the `hosts` dir and `tinc-up`.
3. Generates the `tinc-ip` file to store the peer VPN IP address.
4. Generates the `tinc.conf` file to update the list of peers.
5. Generates the files in `hosts` dir with public and VIP IP addresses and public key.
6. Updates the `/etc/hosts` file with hostnames with the `.tinc` sufix.

This process is configured to be repeated every 30 minutes via cron.

# Usage

In order to allow easy usage of this VPN network all peers have a Consul service configured:
```bash
curl -sk https://localhost:8400/v1/catalog/service/tinc --cert /certs/consul-client.crt --key /certs/consul-client.key  | jq '.[0]'
```
```json
{
  "Address": "35.202.99.224",
  "Datacenter": "gc-us-central1-a",
  "TaggedAddresses": {
    "lan": "35.202.99.224",
    "wan": "35.202.99.224"
  },
  "NodeMeta": {
    "env": "eth",
    "stage": "beta"
  },
  "ServiceID": "tinc",
  "ServiceName": "tinc",
  "ServiceTags": [
    "vpn"
  ],
  "ServiceMeta": {
    "tinc_address": "10.2.0.6",
    "tinc_pub_key": "\n-----BEGIN RSA PUBLIC KEY-----\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\nAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\n-----END RSA PUBLIC KEY-----\n"
  },
  "ServicePort": 655,
}
```

Using the metadata from the catalog contained within `tinc_address` and `tinc_pub_key` variables the `tinc-refresh` script can generate configuration that connects all the known hosts via VPN.
