Docker Compose Networking Deep Dive
===================================

.. note::

  We suggest that you should complete the lab :doc:`bridged-network` firstly before going to this lab.

This lab will use ``example-voting-app`` as the demo application run by docker-compose, you can find the source code of the project in
https://github.com/DaoCloud

Using Compose is basically a three-step process. [#f1]_

1. Define your app’s environment with a Dockerfile so it can be reproduced anywhere.
2. Define the services that make up your app in docker-compose.yml so they can be run together in an isolated environment.
3. Lastly, run docker-compose up and Compose will start and run your entire app.

.. note::
  Unfortunatley, at the end of all this, the app will start, but won't respond to web requests.  Sorry, I'm a Docker/K8s guy, not a javascript developer.  The lab is still informative from a DevOps perspective.

Build APP
----------

Clone the ``example-voting-app`` repository to docker host. The app defines five containers: ``voting-app``, ``result-app``, ``worker``, ``redis``, ``db``.
and two networks: ``front-tier``, ``back-tier`` through ``docker-compose.yml``.

.. code-block:: bash

  $ cd ~
  $ git clone https://github.com/livitup/example-voting-app
  $ cd example-voting-app/

The app's configuration file looks like this:

.. code-block:: bash

  version: "2"

  services:
    voting-app:
      build: ./voting-app/.
      volumes:
       - ./voting-app:/app
      ports:
        - "5000:80"
      links:
        - redis
      networks:
        - front-tier
        - back-tier

    result-app:
      build: ./result-app/.
      volumes:
        - ./result-app:/app
      ports:
        - "5001:80"
      links:
        - db
      networks:
        - front-tier
        - back-tier

    worker:
      build: ./worker
      links:
        - db
        - redis
      networks:
        - back-tier

    redis:
      image: redis
      ports: ["6379"]
      networks:
        - back-tier

    db:
      image: postgres:9.4
      volumes:
        - "db-data:/var/lib/postgresql/data"
      networks:
        - back-tier

  volumes:
    db-data:

  networks:
    front-tier:
    back-tier:

Run ``docker-compose build`` to build required docker images. This will take some time, and a lot of status information will scroll past.  You'll also see some warning messages, but those can be safely ignored.  The final line should show:

.. code-block:: bash

  $ docker-compose build
  ... lots of status will scroll by ...
  Successfully tagged example-voting-app_result-app:latest

Run APP
----------

Next we will use docker-compose to start the application.  We use the ``--detach`` argument to return to the command prompt when the containers are all started.  Without ``-detach`` the system would wait for the conatiners to exit before returning to the command prompt.

.. code-block:: bash

  $ docker-compose up --detach
  Starting example-voting-app_redis_1 ... done
  Starting example-voting-app_db_1         ... done
  Starting example-voting-app_voting-app_1 ... done
  Starting example-voting-app_result-app_1 ... done
  Starting example-voting-app_worker_1     ... done
  $

There will be five containers, two bridge networks and seven veth interfaces created.  We can inspect them with the commands we learned earlier in the lab.

.. code-block:: bash

  $ docker ps
  CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                     NAMES
  c9c4e7fe7b6c        example-voting-app_worker       "/usr/lib/jvm/java-7-"   About an hour ago   Up 5 seconds                                  example-voting-app_worker_1
  4213167049aa        example-voting-app_result-app   "node server.js"         About an hour ago   Up 4 seconds        0.0.0.0:5001->80/tcp      example-voting-app_result-app_1
  8711d687bda9        example-voting-app_voting-app   "python app.py"          About an hour ago   Up 5 seconds        0.0.0.0:5000->80/tcp      example-voting-app_voting-app_1
  b7eda251865d        redis                         "docker-entrypoint.sh"   About an hour ago   Up 5 seconds        0.0.0.0:32770->6379/tcp   example-voting-app_redis_1
  7d6dbb98ce40        postgres:9.4                  "/docker-entrypoint.s"   About an hour ago   Up 5 seconds        5432/tcp                  example-voting-app_db_1
  $ docker network ls
  NETWORK ID          NAME                          DRIVER              SCOPE
  3b5cfe4aafa1        bridge                        bridge              local
  69a019d00603        example-voting-app_back-tier    bridge              local
  6ddb07377c35        example-voting-app_front-tier   bridge              local
  b1670e00e2a3        host                          host                local
  6006af29f010        none                          null                local
  $ brctl show
  bridge name	bridge id		STP enabled	interfaces
  br-69a019d00603		8000.0242c780244f	no		veth2eccb94
  							veth374be12
  							veth57f50a8
  							veth8418ed3
  							veth91d724d
  br-6ddb07377c35		8000.02421dac7490	no		veth156c0a9
  							vethaba6401

Through ``docker network inspect``, we can know which container connnects with the bridge.

There are two containers connected to the docker network ``example-voting-app_front-tier``.

.. code-block:: bash

  $ docker network inspect example-voting-app_front-tier
  [
      {
          "Name": "example-voting-app_front-tier",
          "Id": "6ddb07377c354bcf68542592a8c6eb34d334ce8515e64832b3c7bf2af56274ca",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": null,
              "Config": [
                  {
                      "Subnet": "172.18.0.0/16",
                      "Gateway": "172.18.0.1/16"
                  }
              ]
          },
          "Internal": false,
          "Containers": {
              "4213167049aa7b2cc1b3096333706f2ef0428e78b2847a7c5ddc755f5332505c": {
                  "Name": "example-voting-app_result-app_1",
                  "EndpointID": "00c7e1101227ece1535385e8d6fe9210dfcdc3c58d71cedb4e9fad6c949120e3",
                  "MacAddress": "02:42:ac:12:00:03",
                  "IPv4Address": "172.18.0.3/16",
                  "IPv6Address": ""
              },
              "8711d687bda94069ed7d5a7677ca4c7953d384f1ebf83c3bd75ac51b1606ed2f": {
                  "Name": "example-voting-app_voting-app_1",
                  "EndpointID": "ffc9905cbfd5332b9ef333bcc7578415977a0044c2ec2055d6760c419513ae5f",
                  "MacAddress": "02:42:ac:12:00:02",
                  "IPv4Address": "172.18.0.2/16",
                  "IPv6Address": ""
              }
          },
          "Options": {},
          "Labels": {}
      }
  ]

There are five containers connected to the docker network ``example-voting-app_back-tier``.

.. code-block:: bash

  $ docker network inspect example-voting-app_back-tier
  [
      {
          "Name": "example-voting-app_back-tier",
          "Id": "69a019d00603ca3a06a30ac99fc0a2700dd8cc14ba8b8368de4fe0c26ad4c69d",
          "Scope": "local",
          "Driver": "bridge",
          "EnableIPv6": false,
          "IPAM": {
              "Driver": "default",
              "Options": null,
              "Config": [
                  {
                      "Subnet": "172.19.0.0/16",
                      "Gateway": "172.19.0.1/16"
                  }
              ]
          },
          "Internal": false,
          "Containers": {
              "4213167049aa7b2cc1b3096333706f2ef0428e78b2847a7c5ddc755f5332505c": {
                  "Name": "example-voting-app_result-app_1",
                  "EndpointID": "cb531eb6deb08346d1dbcfa65ea67d43d4c2f244f002b195fc4dadd2adb0b47d",
                  "MacAddress": "02:42:ac:13:00:06",
                  "IPv4Address": "172.19.0.6/16",
                  "IPv6Address": ""
              },
              "7d6dbb98ce408c1837f42fdf743e365cc9b0ee2b7dffd108d97e81b172d43114": {
                  "Name": "example-voting-app_db_1",
                  "EndpointID": "67007a454f320d336c13e30e028cd8e85537400b70a880eabdd1f0ed743b7a6a",
                  "MacAddress": "02:42:ac:13:00:03",
                  "IPv4Address": "172.19.0.3/16",
                  "IPv6Address": ""
              },
              "8711d687bda94069ed7d5a7677ca4c7953d384f1ebf83c3bd75ac51b1606ed2f": {
                  "Name": "example-voting-app_voting-app_1",
                  "EndpointID": "d414b06b9368d1719a05d527500a06fc714a4efae187df32c1476385ee03ae67",
                  "MacAddress": "02:42:ac:13:00:05",
                  "IPv4Address": "172.19.0.5/16",
                  "IPv6Address": ""
              },
              "b7eda251865d824de90ebe0dfefa3e4aab924d5030ccfb21a55e79f910ff857a": {
                  "Name": "example-voting-app_redis_1",
                  "EndpointID": "9acc267d3e6b41da6fe3db040cff964c91037df215a0f2be2155b94be3bb87d0",
                  "MacAddress": "02:42:ac:13:00:02",
                  "IPv4Address": "172.19.0.2/16",
                  "IPv6Address": ""
              },
              "c9c4e7fe7b6c1508f9d9d3a05e8a4e66aa1265f2a5c3d33f363343cd37184e6f": {
                  "Name": "example-voting-app_worker_1",
                  "EndpointID": "557e978eaef18a64f24d400727d396431d74cd7e8735f060396e3226f31ab97b",
                  "MacAddress": "02:42:ac:13:00:04",
                  "IPv4Address": "172.19.0.4/16",
                  "IPv6Address": ""
              }
          },
          "Options": {},
          "Labels": {}
      }
  ]

Container information summary:


==============================  ============================
Container Name                  IP Address
==============================  ============================
example-voting-app_result-app_1   172.19.0.6/16, 172.18.0.3/16
example-voting-app_voting-app_1   172.19.0.3/16, 172.18.0.2/16
example-voting-app_redis_1        172.19.0.2/16
example-voting-app_worker_1       172.19.0.4/16
example-voting-app_db_1           172.19.0.3/16
==============================  ============================

Docker network information summary:

==============================  ============= ============= =========================================================================================================================================
Docker Network Name             Gateway       Subnet        Containers
==============================  ============= ============= =========================================================================================================================================
example-voting-app_front-tier     172.18.0.1/16 172.18.0.0/16 example-voting-app_result-app_1, example-voting-app_voting-app_1
example-voting-app_back-tier      172.19.0.1/16 172.19.0.0/16 example-voting-app_result-app_1, example-voting-app_voting-app_1, example-voting-app_db_1, example-voting-app_redis_1, example-voting-app_worker_1
==============================  ============= ============= =========================================================================================================================================

Network Topology
-----------------

.. image:: _image/docker-compose.png

For bridge network connection details, please reference lab :doc:`bridged-network`

Cleaning up
-------------

Use the ``docker-compose down`` command to terminate your containers.

.. code-block:: bash
  
  $ docker-compose down
  Stopping example-voting-app_worker_1     ... done
  Stopping example-voting-app_voting-app_1 ... done
  Stopping example-voting-app_result-app_1 ... done
  Stopping example-voting-app_db_1         ... done
  Stopping example-voting-app_redis_1      ... done
  Removing network example-voting-app_back-tier
  Removing network example-voting-app_front-tier

Reference
---------

.. [#f1] https://docs.docker.com/compose/overview/
.. [#f2] https://docs.docker.com/compose/install/
