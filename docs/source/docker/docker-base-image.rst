Build a Base Image from Scratch
================================

we will build a ``hello world`` base image from Scratch.


System Environment
-------------------

Verfiy the docker and server OS versions.

.. code-block:: sh

    $ docker version
Client: Docker Engine - Community
 Version:           24.0.2
 API version:       1.43
 Go version:        go1.20.4
 Git commit:        cb74dfc
 Built:             Thu May 25 21:55:21 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.2
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.4
  Git commit:       659604f
  Built:            Thu May 25 21:54:24 2023
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          1.6.21
  GitCommit:        3dce8eb055cbb6872793272b4f20ed16117344f8
 runc:
  Version:          1.1.7
  GitCommit:        v1.1.7-0-g860f061
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

Before building our container, we have to install a prerequisite set of libraries on the base system:

.. code-block:: sh

    $ yum install -y gcc glibc-static


Create a Hello world
---------------------


create a ``hello.c`` file (this is the source code for your application).

.. code-block:: bash

    $ cd ~
    $ mkdir hello-world
    $ cd hello-world
    $ cat <<EOF > hello.c
    #include<stdio.h>

    int main()
    {
    printf("hello docker\n");
    }
    EOF

Compile the ``hello.c`` source file to an binary file, and run it to test your code.

.. code-block:: bash

    $ gcc -o hello -static  hello.c
    $ ls
    hello  hello.c
    $ ./hello
    hello docker


Build Docker image
-------------------

Create a Dockerfile:

.. code-block:: bash

    $ cat <<EOF > Dockerfile
    FROM scratch
    ADD hello /
    CMD ["/hello"]
    EOF

Build the docker image and verify that the image built correctly:

.. code-block:: bash

    $ docker build -t hello-world .
    $ docker image ls
    REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
    hello-world       latest    c4f15764e67b   21 seconds ago   861kB
    redis_localdemo   0.1       41ac630484a6   57 minutes ago   215MB
    ubuntu            14.04     13b66b487594   2 years ago      197MB

Run the hello world container
------------------------------

.. code-block:: bash

    $ docker run hello-world
    hello docker

Congratulations!  You have just built, deployed, and run a docker container!