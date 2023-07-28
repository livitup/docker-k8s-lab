Container Port Mapping in Bridge networking
===========================================

Through :doc:`bridged-network` we learned that, by default, Docker containers can make connections to the outside world,
but the outside world cannot connect to containers. Each outgoing connection will appear to originate from one of
the host machine’s own IP addresses thanks to an iptables masquerading rule on the host machine that the Docker
server creates when it starts: [#f1]_ You can check the rules on your server with the `iptables` command:

.. code-block:: bash

  $ iptables -t nat -L -n
  ...
  Chain POSTROUTING (policy ACCEPT)
  target     prot opt source               destination
  MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
  ...
  $ ifconfig docker0
  docker0   Link encap:Ethernet  HWaddr 02:42:58:22:4c:30
            inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
            UP BROADCAST MULTICAST  MTU:1500  Metric:1
            RX packets:0 errors:0 dropped:0 overruns:0 frame:0
            TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:0
            RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

  $

The Docker server creates a ``masquerade`` rule that let containers connect to IP addresses in the outside world.

Bind Container port to the host
--------------------------------

Nginx is an open source proxy server that, by default, accepts connections on the HTTP and HTTPS TCP (layer 4) ports 80 and 443.  Start a nginx container which exports port 80 and 443. We can access the port from inside of the docker host.  We'll use the `curl` command, which acts as a web browser to download web pages, to see that it's working.

.. code-block:: bash

  $ docker run -d --name demo nginx
  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
  b5e53067e12f        nginx               "nginx -g 'daemon off"   8 minutes ago       Up 8 minutes        80/tcp, 443/tcp     demo
  $ docker inspect --format {{.NetworkSettings.IPAddress}} demo
  172.17.0.2
  $ curl 172.17.0.2
  <!DOCTYPE html>
  <html>
  <head>
  <title>Welcome to nginx!</title>
  <style>
      body {
          width: 35em;
          margin: 0 auto;
          font-family: Tahoma, Verdana, Arial, sans-serif;
      }
  </style>
  </head>
  <body>
  <h1>Welcome to nginx!</h1>
  <p>If you see this page, the nginx web server is successfully installed and
  working. Further configuration is required.</p>

  <p>For online documentation and support please refer to
  <a href="http://nginx.org/">nginx.org</a>.<br/>
  Commercial support is available at
  <a href="http://nginx.com/">nginx.com</a>.</p>

  <p><em>Thank you for using nginx.</em></p>
  </body>
  </html>

If we want to access the nginx web from outside of the docker host, we must bind the port to docker host.  The `-p` argument tells docker to map a port on the host to a container. First find the IP address of your local server, looking for the IP assigned to the `eth0` interface.
Then use the `docker ps` command to see what port on the local host is mapped to the container:

.. code-block:: bash
  $ docker stop demo
  $ docker rm demo
  $ docker run -d  -p 80 --name demo nginx
  0fb783dcd5b3010c0ef47e4c929dfe0c9eac8ddec2e5e0470df5529bfd4cb64e
  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
  0fb783dcd5b3        nginx               "nginx -g 'daemon off"   5 seconds ago       Up 5 seconds        443/tcp, 0.0.0.0:32768->80/tcp   demo
  $ ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
  2: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:50:c5:10:7b brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
        valid_lft forever preferred_lft forever
  12: veth-a@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
      link/ether 16:82:cd:de:59:ed brd ff:ff:ff:ff:ff:ff link-netnsid 1
      inet 192.168.1.1/24 scope global veth-a
        valid_lft forever preferred_lft forever
  14: veth39f61b9@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
      link/ether f2:61:3d:37:b4:e7 brd ff:ff:ff:ff:ff:ff link-netnsid 2
  16: veth9e2e055@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
      link/ether 7e:36:02:97:bc:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 3
  20: veth3a53e04@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
      link/ether ee:ee:e5:f6:83:63 brd ff:ff:ff:ff:ff:ff link-netnsid 4
  65491: eth0@if65492: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether e6:35:10:69:55:36 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 192.168.0.13/23 scope global eth0
        valid_lft forever preferred_lft forever
  65495: eth1@if65496: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:ac:12:00:0d brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 172.18.0.13/16 scope global eth1
        valid_lft forever preferred_lft forever
    $
    $ curl 192.168.0.13:32768
  <!DOCTYPE html>
  <html>
  <head>
  <title>Welcome to nginx!</title>
  <style>
      body {
          width: 35em;
          margin: 0 auto;
          font-family: Tahoma, Verdana, Arial, sans-serif;
      }
  </style>
  </head>
  <body>
  <h1>Welcome to nginx!</h1>
  <p>If you see this page, the nginx web server is successfully installed and
  working. Further configuration is required.</p>

  <p>For online documentation and support please refer to
  <a href="http://nginx.org/">nginx.org</a>.<br/>
  Commercial support is available at
  <a href="http://nginx.com/">nginx.com</a>.</p>

  <p><em>Thank you for using nginx.</em></p>
  </body>
  </html>
  
  $

If we want to specify which port on host want to bind:

.. code-block:: bash

  $ docker run -d  -p 80:80 --name demo1 nginx
  4f548139a4be6574e3f9718f99a05e5174bdfb62d229ea656d35a979b5b0507d
  $ docker ps
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
  4f548139a4be        nginx               "nginx -g 'daemon off"   5 seconds ago       Up 4 seconds        0.0.0.0:80->80/tcp, 443/tcp      demo1
  0fb783dcd5b3        nginx               "nginx -g 'daemon off"   2 minutes ago       Up 2 minutes        443/tcp, 0.0.0.0:32768->80/tcp   demo
  $ curl 192.168.0.13:80
  <!DOCTYPE html>
  <html>
  <head>
  <title>Welcome to nginx!</title>
  <style>
      body {
          width: 35em;
          margin: 0 auto;
          font-family: Tahoma, Verdana, Arial, sans-serif;
      }
  </style>
  </head>
  <body>
  <h1>Welcome to nginx!</h1>
  <p>If you see this page, the nginx web server is successfully installed and
  working. Further configuration is required.</p>

  <p>For online documentation and support please refer to
  <a href="http://nginx.org/">nginx.org</a>.<br/>
  Commercial support is available at
  <a href="http://nginx.com/">nginx.com</a>.</p>

  <p><em>Thank you for using nginx.</em></p>
  </body>
  </html>
  
  $

What happened
--------------

Check iptables to show the rules that Docker installs in the Linux firewall:

.. code-block:: bash


  $ iptables -t nat -L -n
  Chain PREROUTING (policy ACCEPT)
  target     prot opt source               destination
  DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

  Chain INPUT (policy ACCEPT)
  target     prot opt source               destination

  Chain OUTPUT (policy ACCEPT)
  target     prot opt source               destination
  DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

  Chain POSTROUTING (policy ACCEPT)
  target     prot opt source               destination
  MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
  MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80
  MASQUERADE  tcp  --  172.17.0.3           172.17.0.3           tcp dpt:80

  Chain DOCKER (2 references)
  target     prot opt source               destination
  RETURN     all  --  0.0.0.0/0            0.0.0.0/0
  DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:32768 to:172.17.0.2:80
  DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.3:80
  $

  $ iptables -t nat -nvxL
  Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
      pkts      bytes target     prot opt in     out     source               destination
         1       44 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

  Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
      pkts      bytes target     prot opt in     out     source               destination

  Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
      pkts      bytes target     prot opt in     out     source               destination
         4      240 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

  Chain POSTROUTING (policy ACCEPT 2 packets, 120 bytes)
      pkts      bytes target     prot opt in     out     source               destination
         0        0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
         0        0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80
         0        0 MASQUERADE  tcp  --  *      *       172.17.0.3           172.17.0.3           tcp dpt:80

  Chain DOCKER (2 references)
      pkts      bytes target     prot opt in     out     source               destination
         0        0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
         1       60 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:32768 to:172.17.0.2:80
         2      120 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.3:80
  $


References
----------

.. [#f1] https://docs.docker.com/engine/userguide/networking/default_network/binding/
