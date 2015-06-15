# Dynamic DNS with BIND and dhclient {#dyn_dns}

In this blogpost we're going to configure the BIND server to accept dynamic updates. Client machines themselves will send the updates to the DNS server instead of letting DHCP server update the DNS. A great setup for situations where the DHCP server is not in your control.

<!-- more -->

Examples in this article work on RHEL6 that comes with BIND 9. You'll need to have `bind` and `bind-utils` RPM packages installed. In the following, the BIND server with host name `ns.somedomain.com` is an authoritative DNS server for the fictive zone `somedomain.com`.

# Dynamic DNS with BIND {#dyn_dns_bind}

In our example we're going to configure the BIND server to accept DNS updates for `somedomain.com` zone from any client. In production environment you'd use encryption keys to secure the access to the DNS server. You can read more on the secure configuration in [this](http://linux.yyz.us/nsupdate/ "nsupdate: Painless Dynamic DNS") excellent article. To allow any client to update the `somedomain.com` zone add the `allow-update { 0/0; };` option into your `/etc/named.conf` file:

~~~
zone "somedomain.com" in {
        type master;
        file "db.somedomain.com";
        allow-update { 0/0; };
};
~~~

After restarting the DNS server with `sudo /etc/init.d/named restart` we can test that the DNS updates are working. Let's ask the DNS server `ns.somedomain.com` to register a host `somehost.somedomain.com` with IP address `192.168.100.200`:
~~~
nsupdate -d << EOF
server ns.somedomain.com
update add somehost.somedomain.com 300 A 192.168.100.200
send
EOF
~~~
If everything worked fine you should see `status: NOERROR` in the reply from update query. The DNS server created a new record in its database pairing the `somehost.somedomain.com` host name with the IP address `192.168.100.200`. Let's check that the host name resolution works by issuing:
~~~
host somehost.somedomain.com ns.somedomain.com
~~~
You should see the IP address `192.168.100.200` in the command output. When on the DNS server you can dump the zone data into `/var/named/data/cache_dump.db` file for inspection:
~~~
rndc dumpdb -all
~~~

# Updating DNS after IP acquisition {#dyn_nds_update}

Our virtual machines obtain their IP addresses via DHCP. Whenever the virtual machine obtains a new IP address or renews the lease we'd like it to update the DNS accordingly. This way the DNS is always kept up to date and we're able to access the virtual machine using its host name.

The IP address acquisition is managed by the DHCP client `dhclient` running on the virtual machine. The `dhclient` can be extended by custom hooks. We are going to prepare a script that updates the DNS database whenever the virtual machine acquires an IP address. Our DNS update hook must be saved at ` /etc/dhcp/dhclient-eth0-up-hooks`. The `/sbin/dhclient-script` shell script that comes with the `dhclient` package will execute the hook. Upon execution the hook is passed a `reason` variable describing the event.

To install the update hook on the virtual machine let's make use of Cloud-Init tool that I talked about in the [previous blogpost](/blog/2015/04/26/using-cloud-init-outside-of-cloud/ "Using Cloud-Init Outside of Cloud"). The cloud-config script to be consumed by Cloud-Init looks as follows:

~~~
#cloud-config
fqdn: somehost.somedomain.com
write_files:
  - path: /etc/dhcp/dhclient-eth0-up-hooks
    permissions: '0755'
    content: |
      #!/bin/bash
      INTERFACE=eth0
      LEASE_FILE=/var/lib/dhclient/dhclient-$INTERFACE.leases
      HOST_ADDR=$(sed -n -e 's/.*fixed-address \([0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+\).*/\1/p' $LEASE_FILE | tail -1)
      HOST_NAME=$(hostname)
      NAMESERVER=ns.somedomain.com
      TTL=300

      if host $NAMESERVER 1>/dev/null 2>&1; then
        case $reason in
          BOUND|RENEW|REBIND|REBOOT)
            nsupdate << EOF
              server $NAMESERVER
              update delete $HOST_NAME A
              update add $HOST_NAME $TTL A $HOST_ADDR
              send
      EOF
          ;;
        esac
      fi
runcmd:
  - hostname somehost.somedomain.com # fix the hostname incorrectly set up by cloud-init
  - reason=BOUND /etc/dhcp/dhclient-eth0-up-hooks # DNS registration on first boot
~~~

Upon the very first execution of the hook the machine's network setup is not complete yet. There's no `/etc/resolv.conf` file written yet and the default route is not configured. The condition `if host $NAMESERVER; then ...` skips the DNS update in this case. Later in the initialization process the `runcmd` part of the cloud-config script gets executed. At this time the network configuration is complete and so we execute the update hook manually. This is the first time that the virtual machine registers itself with DNS. Cloud-Init executes the `runcmd` section only on the very first boot. Subsequent boots won't execute the `runcmd` code.

Note that we're parsing the `/var/lib/dhclient/dhclient-eth0.leases` file to obtain the acquired IP address. Should the virtual machine obtain different IP address in the future the DNS entry gets updated accordingly.
