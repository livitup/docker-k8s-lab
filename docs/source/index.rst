.. Docker Kubernetes Lab documentation master file, created by
   sphinx-quickstart on Fri Nov 25 23:35:48 2016.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Docker Kubernetes Lab Handbook
==============================

This handbook contains some docker and kubernetes lab tutorials. It will be useful if you are learning docker or kubernetes now.
The labs in this tutorial are all well documented, include the required environments, steps, detailed input and output.

This lab works with "Play-with-k8s" instances from https://labs.play-with-k8s.com/ 

Preview Docker concepts with this video: https://www.youtube.com/watch?v=rOTqprHv1YE

.. warning::

  This is just a lab guide, not documentation for docker or kubernetes. Please go to their online documentation sites for more details about
  what docker or kubernetes are and how they work.

Table of Contents
-----------------

.. toctree::
   :maxdepth: 2

   docker

Kubernetes
==================

To do the kubernetes labs, first install kubernetes on your node and start a single-node cluster:
.. code-block:: bash

   $ curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
   $ k3d cluster create k8s -p "30000-30040:30000-30040@server:0"

Next, download the lab files to your node:
.. code-block:: bash
   
   $ git clone https://github.com/courselabs/kubernetes.git

Once your cluster is up and running, start with the "Core Kubernetes" lab here: https://kubernetes.courselabs.co/#core-kubernetes

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
