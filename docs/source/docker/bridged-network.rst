Bridge Networking Deep Dive
===========================

The bridge network represents the docker0 network present in all Docker installations. Unless you specify otherwise with
the ``docker run --network=<NETWORK> option``, the Docker daemon connects containers to this network by default.

There are four important concepts about bridged networking:

- Docker0 Bridge
- Network Namespace
- Veth Pair
- External Communication


Docker0 bridge
--------------

Through the ``docker network`` command we can get more details about the docker0 bridge, and from the output, we can see there is no container
connected to the bridge now.


.. code-block:: bash

  $ docker network ls
  NETWORK ID     NAME      DRIVER    SCOPE
  09a656f355b7   bridge    bridge    local
  262c9060a1d3   host      host      local
  a2d74fb15378   none      null      local

The `network inspect` docker command shows you details on a specifc network, selected by the `NETWORK ID`.  For this example, use the ID of your `bridge` network:

.. code-block:: bash

  $ docker network inspect 09a656f355b7
  [
      {
          "Name": "bridge",
          "Id": "32b93b141baeeac8bbf01382ec594c23515719c0d13febd8583553d70b4ecdba",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": null,
              "Config": [
                  {
                      "Subnet": "172.17.0.0/16",
                      "Gateway": "172.17.0.1"
                  }
              ]
          },
          "Internal": false,
          "Containers": {},
          "Options": {
              "com.docker.network.bridge.default_bridge": "true",
              "com.docker.network.bridge.enable_icc": "true",
              "com.docker.network.bridge.enable_ip_masquerade": "true",
              "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
              "com.docker.network.bridge.name": "docker0",
              "com.docker.network.driver.mtu": "1500"
          },
          "Labels": {}
      }
  ]

This is your first exposure to Docker configuration.  Docker stores configuration internally in the `YAML` format.
You can also see this bridge as a part of a host’s network stack by using the `ip` command on the host.

.. code-block:: bash

  $ ip link
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
      link/ether 06:95:4a:1f:08:7f brd ff:ff:ff:ff:ff:ff
  3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT
      link/ether 02:42:d6:23:e6:18 brd ff:ff:ff:ff:ff:ff

Because there are no containers running, the bridge ``docker0`` status is down.

You can also use ``brctl`` command to get bridge docker0 information.  First install the command...

.. code-block:: bash

  $ yum install bridge-utils net-utils
  Loaded plugins: fastestmirror, ovl
  Loading mirror speeds from cached hostfile
  * base: nl.mirrors.clouvider.net
  * extras: nl.mirrors.clouvider.net
  * updates: nl.mirrors.clouvider.net
  Resolving Dependencies
  --> Running transaction check
  ---> Package bridge-utils.x86_64 0:1.5-9.el7 will be installed
  --> Finished Dependency Resolution

  Dependencies Resolved

  ================================================================================
  Package               Arch            Version              Repository     Size
  ================================================================================
  Installing:
  bridge-utils          x86_64          1.5-9.el7            base           32 k

  Transaction Summary
  ================================================================================
  Install  1 Package

  Total download size: 32 k
  Installed size: 56 k
  Is this ok [y/d/N]: y
  Downloading packages:
  bridge-utils-1.5-9.el7.x86_64.rpm                          |  32 kB   00:00     
  Running transaction check
  Running transaction test
  Transaction test succeeded
  Running transaction
    Installing : bridge-utils-1.5-9.el7.x86_64                                1/1 
    Verifying  : bridge-utils-1.5-9.el7.x86_64                                1/1 

  Installed:
    bridge-utils.x86_64 0:1.5-9.el7                                               

  Complete!

And then run the command:

.. code-block:: bash

  $ brctl show
  bridge name     bridge id               STP enabled     interfaces
  docker0         8000.024250c5107b       no

Veth Pair
---------

To test neworking, we create and run a centos7 container.  This container will build and start, then the `sleep` command prevents the container from exiting.

.. code-block:: bash

  $ docker run -d --name test1 centos:7 /bin/bash -c "while true; do sleep 3600; done"
  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  4fea95f2e979        centos:7            "/bin/bash -c 'while "   6 minutes ago       Up 6 minutes                            test1

After that we can check the ip interface in the docker host.

.. code-block:: bash

  $ ip link
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
      link/ether 06:95:4a:1f:08:7f brd ff:ff:ff:ff:ff:ff
  3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
      link/ether 02:42:d6:23:e6:18 brd ff:ff:ff:ff:ff:ff
  15: vethae2abb8@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT
      link/ether e6:97:43:5c:33:a6 brd ff:ff:ff:ff:ff:ff link-netnsid 0

The bridge ``docker0`` is up, and there is a veth pair created, one is in localhost, and another is in container's network namspace.


Network Namespace
------------------

If we add a new network namespace from command line:

.. code-block:: bash

  $ ip netns add demo
  $ ip netns list
  demo
  $ ip netns exec demo ip a
  1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

Typically Linux systems store configuration data in `run files` or `proc files`.  Linux will create run files for network namespaces, but when docker creates networks, it deletes them from the run file.  Docker stores internal namespace information in a system file.  We can get all the docker container network namespaces from ``/var/run/docker/netns``.

.. code-block:: bash

  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  4fea95f2e979        centos:7            "/bin/bash -c 'while "   2 hours ago         Up About an hour                        test1
  $ ls -l /var/run/docker/netns
  total 0
  -rw-r--r--. 1 root root 0 Nov 28 05:51 572d8e7abcb2

The filename (`572d8e7abcb2` in this case) is the Network ID.

How to get the detail information (like veth) about the container network namespace?  First we should get the pid of this container process, and then check the /proc filesystem to get all namespaces about this container.

.. code-block:: bash

  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  4fea95f2e979        centos:7            "/bin/bash -c 'while "   2 hours ago         Up 2 hours                              test1
  $ docker inspect --format '{{.State.Pid}}' 4f
  3090
  $ ls -l /proc/3090/ns
  total 0
  lrwxrwxrwx. 1 root root 0 Nov 28 05:52 ipc -> ipc:[4026532156]
  lrwxrwxrwx. 1 root root 0 Nov 28 05:52 mnt -> mnt:[4026532154]
  lrwxrwxrwx. 1 root root 0 Nov 28 05:51 net -> net:[4026532159]
  lrwxrwxrwx. 1 root root 0 Nov 28 05:52 pid -> pid:[4026532157]
  lrwxrwxrwx. 1 root root 0 Nov 28 08:02 user -> user:[4026531837]
  lrwxrwxrwx. 1 root root 0 Nov 28 05:52 uts -> uts:[4026532155]

.. note::
  This is the first example of a "Docker Shortcut".  When specifying Container (and many other types of) IDs, you only need to type enough to match a single ID.

Finally, we can restore the network namespace:

.. code-block:: bash
  $ ln -s /proc/3090/ns/net /var/run/netns/3090
  $ ip netns list
  3090
  demo
  $ ip netns exec 3090 ip link
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  26: eth0@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0



After all is done, please remove ``/var/run/netns/3090``.

.. code-block:: bash
  rm /var/run/netns/3090

External Communication
----------------------

All containers connected with bridge ``docker0`` can communicate with the external network or other containers which are connected to the same bridge.

Let's start two containers:

.. code-block:: bash

  $ docker run -d --name test2 centos:7 /bin/bash -c "while true; do sleep 3600; done"
  8975cb01d142271d463ec8dac43ea7586f509735d4648203319d28d46365af2f
  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  8975cb01d142        centos:7            "/bin/bash -c 'while "   4 seconds ago       Up 4 seconds                            test2
  4fea95f2e979        centos:7            "/bin/bash -c 'while "   27 hours ago        Up 26 hours                             test1

And from the bridge ``docker0``, we can see two interfaces connected.

.. code-block:: bash

  $ brctl show
  bridge name     bridge id               STP enabled     interfaces
  docker0         8000.0242d623e618       no              veth6a5ae6f
                                                          vethc16e6c8
  $ ip link
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
      link/ether 06:95:4a:1f:08:7f brd ff:ff:ff:ff:ff:ff
  3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
      link/ether 02:42:d6:23:e6:18 brd ff:ff:ff:ff:ff:ff
  27: veth6a5ae6f@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT
      link/ether 02:7d:eb:4e:85:99 brd ff:ff:ff:ff:ff:ff link-netnsid 0
  31: vethc16e6c8@if30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT
      link/ether d2:9f:2e:ca:22:a5 brd ff:ff:ff:ff:ff:ff link-netnsid 1

The two containers can be reached by each other.  You can use the `docker exec` command to run a command within a container.

.. code-block:: bash

  $  docker inspect --format '{{.NetworkSettings.IPAddress}}' test1
  172.17.0.2
  $  docker inspect --format '{{.NetworkSettings.IPAddress}}' test2
  172.17.0.3
  $ docker exec test1 bash -c 'ping 172.17.0.3'
  PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
  64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.051 ms
  64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.058 ms
  64 bytes from 172.17.0.3: icmp_seq=3 ttl=64 time=0.053 ms
  ^C

The basic network would be like below:

.. image:: _image/two-container-network.png


CNM
~~~~

To understand how a container get its ip address, you should understand the CNM (Container Network Model) [#f2]_.

`Libnetwork` implements the Container Network Model (CNM) in Docker, which formalizes the steps required to provide networking for
containers while providing an abstraction that can be used to support multiple network drivers.

During the Network and Endpoints lifecycle, the CNM model controls the IP address assignment for network
and endpoint interfaces via the IPAM driver(s) [#f1]_.

When creating the bridge ``docker0``,  libnetwork will make a request to the IPAM driver (which is acting like a network gateway) for an address
pool. When creating a container, in the network sandbox, and endpoint was created, libnetwork will request an IPv4 address from
the IPv4 pool and assign it to the endpoint interface IPv4 address.

.. image:: _image/cnm-model.jpg

NAT
~~~

Containers in bridge network mode can access the external network through ``NAT`` (network address translation) which is configured by ``iptables``.

Inside the container:

.. code-block:: bash

  $ docker exec test1 bash -c 'ping www.google.com'
  PING www.google.com (172.217.27.100) 56(84) bytes of data.
  64 bytes from ams17s12-in-f4.1e100.net (142.251.36.36): icmp_seq=1 ttl=110 time=2.21 ms
  64 bytes from ams17s12-in-f4.1e100.net (142.251.36.36): icmp_seq=2 ttl=110 time=2.38 ms
  64 bytes from ams17s12-in-f4.1e100.net (142.251.36.36): icmp_seq=3 ttl=110 time=2.29 ms
  ^C
  --- www.google.com ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2004ms
  rtt min/avg/max/mdev = 99.073/106.064/110.400/4.990 ms

From the docker host, we can see the `Chain Docker` that allows traffic from anywhere, to anywhere:

.. code-block:: bash

  $ iptables --list -t nat
  Chain PREROUTING (policy ACCEPT)
  target     prot opt source               destination
  DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

  Chain INPUT (policy ACCEPT)
  target     prot opt source               destination

  Chain OUTPUT (policy ACCEPT)
  target     prot opt source               destination
  DOCKER     all  --  anywhere            !loopback/8           ADDRTYPE match dst-type LOCAL

  Chain POSTROUTING (policy ACCEPT)
  target     prot opt source               destination
  MASQUERADE  all  --  172.17.0.0/16  anywhere

  Chain DOCKER (2 references)
  target     prot opt source               destination
  RETURN     all  --  anywhere             anywhere


It's a good idea to lock down network access for production Docker containers.  For further information on NAT with iptables, you can reference [#f3]_ [#f4]_


Reference
----------

.. [#f1] https://github.com/docker/libnetwork/blob/master/docs/ipam.md
.. [#f2] https://github.com/docker/libnetwork/blob/master/docs/design.md
.. [#f3] http://www.karlrupp.net/en/computer/nat_tutorial
.. [#f4] https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Security_Guide/s1-firewall-ipt-fwd.html
