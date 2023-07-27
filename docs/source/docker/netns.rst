Linux Network Namespace Introduction
=====================================

In this tutorial, we will learn ahout the Linux network namespace and how to use it.

Docker uses many Linux namespace technologies for isolation, there are user namespace, process namespace, etc. For network isolation
docker uses Linux network namespace technology: each docker container has its own network namespace, which means it has its own IP address,
routing table, etc.

First, let's see how to create and check a network namespace.

Create and List Network Namespace
----------------------------------

Use ``ip netns add <network namespace name>`` to create a network namespace, and ``ip netns list`` to list all network namepaces on the host.

.. code-block:: bash

    $ ip netns add test1
    $ ip netns list
    test1
    $


Delete Network Namespace
-------------------------

Use ``ip netns delete <network namespace name>`` to delete a network namespace.

.. code-block:: bash

    $ ip netns delete test1
    $ ip netns list
    $

Execute commands within a Network Namespace
-------------------------------------

To check interfaces in a particular network namespace, we can use command ``ip netns exec <network namespace name> <command>`` like:

.. code-block:: bash

    $ ip netns add test1
    $ ip netns exec test1 ip a
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    $

``ip a`` will list all ip interfaces within this ``test1`` network namespaces. From the output we can see that the ``lo`` inteface is ``DOWN``, 
we can run a command to set it up.

.. code-block:: bash

    $ ip netns exec test1 ip link
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    $ ip netns exec test1 ip link set dev lo up
    $ ip netns exec test1 ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

The status of lo became ``UNKNOWN``, you can ignore that and continue on.

Add Interface to a Network Namespace
------------------------------------

We will create a virtual interface pair: it has two virtual interfaces which are connected by a virtual cable

.. code-block:: bash

    $ ip link add veth-a type veth peer name veth-b
    $ ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 02:30:c1:3e:63:3a brd ff:ff:ff:ff:ff:ff
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:a7:88:bd:32 brd ff:ff:ff:ff:ff:ff
    27: veth-b: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 52:58:31:ef:0b:98 brd ff:ff:ff:ff:ff:ff
    28: veth-a: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 3e:89:92:ac:ef:10 brd ff:ff:ff:ff:ff:ff
    $

These two interfaces are located on localhost default network namespace. what we will do is move one of them to ``test1`` network namespace,
we can do this through:

.. code-block:: bash

    $ ip link set veth-b netns test1
    $ ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 02:30:c1:3e:63:3a brd ff:ff:ff:ff:ff:ff
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:a7:88:bd:32 brd ff:ff:ff:ff:ff:ff
    28: veth-a: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 3e:89:92:ac:ef:10 brd ff:ff:ff:ff:ff:ff
    
    $ ip netns exec test1 ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    27: veth-b: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 52:58:31:ef:0b:98 brd ff:ff:ff:ff:ff:ff
    $

Notice how the ``veth-b`` is missing from the first output?  That's because you mover it to the network namespace ``test1``.

Assign IP address to veth interface
------------------------------------

In the localhost to set ``veth-a``

.. code-block:: bash

    $ ip addr add 192.168.1.1/24 dev veth-a
    $ ip link set veth-a up
    $ ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 02:30:c1:3e:63:3a brd ff:ff:ff:ff:ff:ff
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:a7:88:bd:32 brd ff:ff:ff:ff:ff:ff
    28: veth-a: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
        link/ether 3e:89:92:ac:ef:10 brd ff:ff:ff:ff:ff:ff

``veth-a`` has an IP address, but its status is DOWN. Now let's set ``veth-b`` in ``test1``.

.. code-block:: bash

    $ ip netns exec test1 ip addr add 192.168.1.2/24 dev veth-b
    $ ip netns exec test1 ip link set dev veth-b up
    $ ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 02:30:c1:3e:63:3a brd ff:ff:ff:ff:ff:ff
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:a7:88:bd:32 brd ff:ff:ff:ff:ff:ff
    28: veth-a: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 3e:89:92:ac:ef:10 brd ff:ff:ff:ff:ff:ff
    $ ip netns exec test1 ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    27: veth-b: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 52:58:31:ef:0b:98 brd ff:ff:ff:ff:ff:ff

After we configured ``veth-b`` and brought it up, both ``veth-a`` and ``veth-b`` are UP. Now we can use ``ping`` to check their connectivity. (Use control-c to stop the ping.)

.. code-block:: bash

    $ ping 192.168.1.2
    PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
    64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.047 ms
    64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=0.046 ms
    64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=0.052 ms
    ^C
    --- 192.168.1.2 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 1998ms
    rtt min/avg/max/mdev = 0.046/0.048/0.052/0.006 ms
    $


Please go to http://www.opencloudblog.com/?p=66 to learn more.
