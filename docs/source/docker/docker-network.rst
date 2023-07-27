Docker Network Overview
=======================

.. image:: _image/docker-turtles-communication.jpg

When you install Docker, it creates three networks automatically. You can list these networks using the docker network ls command:

.. code-block:: bash

  $ docker network ls
  NETWORK ID          NAME                DRIVER
  32b93b141bae        bridge              bridge
  c363d9a92877        host                host
  88077db743a8        none                null
  