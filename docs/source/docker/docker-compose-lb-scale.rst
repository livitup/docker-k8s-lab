Docker Compose Load Blancing and Scaling
=========================================

Please finish :doc:`docker-compose` before starting this lab.

In this lab, we will create a web service, try to scale this service, and add a load blancer.

In our ``docker-compose.yml`` file, we just use two images...

.. code-block:: bash

  $ cd ~
  $ mkdir scaling
  $ cd scaling
  $ cat <<EOF > docker-compose.yml
  web:
    image: 'jwilder/whoami'
  lb:
    image: 'dockercloud/haproxy:latest'
    links:
      - web
    ports:
      - '80:80'
  EOF

Start and check the service.

.. code-block:: bash

  $ docker-compose up --detach
  Creating ubuntu_web_1
  Creating ubuntu_lb_1
  $ docker-compose ps
      Name                  Command               State                   Ports
  ---------------------------------------------------------------------------------------------
  ubuntu_lb_1    /sbin/tini -- dockercloud- ...   Up      1936/tcp, 443/tcp, 0.0.0.0:80->80/tcp
  ubuntu_web_1   /bin/sh -c php-fpm -d vari ...   Up      80/tcp

Open the browser and check the hostname.

.. note::
  At this moment the play-with-k8s application is not allowing users to access the applications running on their hosts from the outside world.  The browser will probably time out, but the app is running.

Scale the web service to 2 and check:

.. code-block:: bash

  $ docker-compose up --scale web=3
  Creating and starting ubuntu_web_2 ... done
  Creating and starting ubuntu_web_3 ... done
  ubuntu@aws-swarm-manager:~$ docker-compose ps
      Name                  Command               State                   Ports
  ---------------------------------------------------------------------------------------------
  ubuntu_lb_1    /sbin/tini -- dockercloud- ...   Up      1936/tcp, 443/tcp, 0.0.0.0:80->80/tcp
  ubuntu_web_1   /bin/sh -c php-fpm -d vari ...   Up      80/tcp
  ubuntu_web_2   /bin/sh -c php-fpm -d vari ...   Up      80/tcp
  ubuntu_web_3   /bin/sh -c php-fpm -d vari ...   Up      80/tcp
