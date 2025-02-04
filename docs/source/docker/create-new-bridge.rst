Create a new bridge network and connect with container
=======================================================

You can isolate containers, such as all the containers running the same microservice, on a shared network bridge.  This is a great security practice, as all the containers can easily have the same network security rules applied to them at the bridge level.

Create a new Bridge Network
---------------------------

Use ``docker network create -d bridge NETWORK_NAME`` command to create a new bridge network [#f1]_.

.. code-block:: bash

  $ docker network ls
  NETWORK ID          NAME                DRIVER              SCOPE
  326ddef352c5        bridge              bridge              local
  1ca18e6b4867        host                host                local
  e9530f1fb046        none                null                local
  $ docker network create -d bridge my-bridge
  e0fc5f7ff50e97787a7b13064f12806232dcc88bafa9c2eb07cec5e81cefd886
  $ docker network ls
  NETWORK ID          NAME                DRIVER              SCOPE
  326ddef352c5        bridge              bridge              local
  1ca18e6b4867        host                host                local
  e0fc5f7ff50e        my-bridge           bridge              local
  e9530f1fb046        none                null                local
  $
  $ ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 02:30:c1:3e:63:3a brd ff:ff:ff:ff:ff:ff
      inet 172.31.29.93/20 brd 172.31.31.255 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::30:c1ff:fe3e:633a/64 scope link
         valid_lft forever preferred_lft forever
  4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
      link/ether 02:42:a7:88:bd:32 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 scope global docker0
         valid_lft forever preferred_lft forever
      inet6 fe80::42:a7ff:fe88:bd32/64 scope link
         valid_lft forever preferred_lft forever
  56: br-e0fc5f7ff50e: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
      link/ether 02:42:c0:80:09:3c brd ff:ff:ff:ff:ff:ff
      inet 172.18.0.1/16 scope global br-e0fc5f7ff50e
         valid_lft forever preferred_lft forever
  $ brctl show
  bridge name bridge id   STP enabled interfaces
  br-e0fc5f7ff50e   8000.0242c080093c no
  docker0   8000.0242a788bd32 no
  $


Create a Container connected with new Bridge
---------------------------------------------

Create a container connected with the ``my-bridge`` network.  This container will also run a sleep routine to keep it from exiting.

.. code-block:: bash
  
  $ docker stop test1
  test1
  $ docker stop test2
  test2
  $ docker rm test1
  test1
  $ docker rm test2
  test2
  $ docker run -d --name test1 --network my-bridge busybox sh -c "while true;do sleep 3600;done"
  $ docker exec -it test1 sh
  / # ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
  57: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.18.0.2/16 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::42:acff:fe12:2/64 scope link
         valid_lft forever preferred_lft forever

  / # exit
  $ brctl show
  bridge name bridge id   STP enabled interfaces
  br-e0fc5f7ff50e   8000.0242c080093c no    veth2f36f74
  docker0   8000.0242a788bd32 no
  $

The new container will connect to the ``my-bridge`` bridge.

Change a Container's network
-----------------------------

Create two containers which connect with the default ``docker0`` bridge.

.. code-block:: bash

  $ docker run -d --name test2  busybox sh -c "while true;do sleep 3600;done"
  73624dd5373b594526d73a1d6fb68a32b92c1ed75e84575f32e4e0f2e1d8d356
  $ docker run -d --name test3  busybox sh -c "while true;do sleep 3600;done"
  33498192d489832a8534fb516029be7fbaf0b58e665d3e4922147857ffbbc10b

Create a new bridge network

.. code-block:: bash

  $ docker network create -d bridge demo-bridge
  be9309ebb3b3fc18c3d43b0fef7c82fe348ce7bf841e281934deccf6bd6e51eb

Use ``docker network connect demo-bridge test2`` command to connect container ``test2`` to bridge ``demo-bridge``.  After connecting the container to the bridge, inspect the docker network information to verify.

.. code-block:: bash

  $ docker network connect demo-bridge test2
  $ brctl show
  bridge name bridge id   STP enabled interfaces
  br-be9309ebb3b3   8000.02423906b898 no    vethec7dc1d
  docker0   8000.0242a788bd32 no    veth3238a5d
                veth7b516dd
  $ docker network inspect demo-bridge
  [
      {
          "Name": "demo-bridge",
          "Id": "be9309ebb3b3fc18c3d43b0fef7c82fe348ce7bf841e281934deccf6bd6e51eb",
          "Created": "2017-02-23T06:16:28.251575297Z",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": {},
              "Config": [
                  {
                      "Subnet": "172.18.0.0/16",
                      "Gateway": "172.18.0.1"
                  }
              ]
          },
          "Internal": false,
          "Attachable": false,
          "Containers": {
              "73624dd5373b594526d73a1d6fb68a32b92c1ed75e84575f32e4e0f2e1d8d356": {
                  "Name": "test1",
                  "EndpointID": "b766bfcc7fc851620b63931f114f5b81b5e072c7ffd64d8f1c99d9828810f17a",
                  "MacAddress": "02:42:ac:12:00:02",
                  "IPv4Address": "172.18.0.2/16",
                  "IPv6Address": ""
              }
          },
          "Options": {},
          "Labels": {}
      }
  ]

Now the container ``test2`` has connected with the default ``docker0`` bridge and ``demo-bridge``. We can do them same action
to connect container ``test3`` to ``demo-bridge`` network. 

.. code-block:: bash

  $ docker network connect demo-bridge test3
  $ brctl show
  bridge name bridge id   STP enabled interfaces
  br-be9309ebb3b3   8000.02423906b898 no    veth67bd1b0
                vethec7dc1d
  docker0   8000.0242a788bd32 no    veth3238a5d
                veth7b516dd
  $ docker network inspect demo-bridge
  [
      {
          "Name": "demo-bridge",
          "Id": "be9309ebb3b3fc18c3d43b0fef7c82fe348ce7bf841e281934deccf6bd6e51eb",
          "Created": "2017-02-23T06:16:28.251575297Z",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": {},
              "Config": [
                  {
                      "Subnet": "172.18.0.0/16",
                      "Gateway": "172.18.0.1"
                  }
              ]
          },
          "Internal": false,
          "Attachable": false,
          "Containers": {
              "33498192d489832a8534fb516029be7fbaf0b58e665d3e4922147857ffbbc10b": {
                  "Name": "test2",
                  "EndpointID": "26d6bdc1c1c0459ba49718e07d6983a9dda1a1a96db3f1beedcbc5ea54abd163",
                  "MacAddress": "02:42:ac:12:00:03",
                  "IPv4Address": "172.18.0.3/16",
                  "IPv6Address": ""
              },
              "73624dd5373b594526d73a1d6fb68a32b92c1ed75e84575f32e4e0f2e1d8d356": {
                  "Name": "test3",
                  "EndpointID": "b766bfcc7fc851620b63931f114f5b81b5e072c7ffd64d8f1c99d9828810f17a",
                  "MacAddress": "02:42:ac:12:00:02",
                  "IPv4Address": "172.18.0.2/16",
                  "IPv6Address": ""
              }
          },
          "Options": {},
          "Labels": {}
      }
  ]

Now, if we go into ``test2``, we can ping ``test3`` directly by container name:

.. code-block:: bash

  $ docker exec -it test2 sh
  / # ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
  78: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.2/16 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::42:acff:fe11:2/64 scope link
         valid_lft forever preferred_lft forever
  83: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.18.0.2/16 scope global eth1
         valid_lft forever preferred_lft forever
      inet6 fe80::42:acff:fe12:2/64 scope link
         valid_lft forever preferred_lft forever
  / # ping test3
  PING test3 (172.18.0.3): 56 data bytes
  64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.095 ms
  64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.077 ms
  ^C
  --- test2 ping statistics ---
  2 packets transmitted, 2 packets received, 0% packet loss
  round-trip min/avg/max = 0.077/0.086/0.095 ms

Also, we can use ``docker network disconnect demo-bridge test2`` to disconnect container ``test2`` from
network ``demo-bridge``.

Reference
----------

.. [#f1] https://docs.docker.com/engine/reference/commandline/network_create/
