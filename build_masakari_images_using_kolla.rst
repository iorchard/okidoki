Build masakari images using kolla
=====================================

This is a guide to build masakari images from customized sources using kolla.

Kolla is a tool to create a production-ready container images for OpenStack.

I assume the build machine is debian 10 buster.

Install docker-ce
------------------

Reference: https://docs.docker.com/install/linux/docker-ce/debian/.

#. Update the apt package index and upgrade to the latest.::

   $ sudo apt update && sudo apt upgrade -y

#. Install packages to allow apt to use a repository over HTTPS.::

   $ sudo apt install -y \
                    apt-transport-https \
                    ca-certificates \
                    curl \
                    gnupg2 \
                    software-properties-common

#. Add docker's official GPG key.::

   $ curl -fsSL https://download.docker.com/linux/debian/gpg \
                    | sudo apt-key add -

#. Add docker stable repository.::

   $ sudo add-apt-repository \
            "deb [arch=amd64] https://download.docker.com/linux/debian \
            $(lsb_release -cs) \
             stable"

#. Update the apt package index again since we added a new docker repo.::

   $ sudo apt update

#. Install Docker CE and containerd.::

   $ sudo apt install -y docker-ce docker-ce-cli containerd.io

#. Add user to the group docker so user can run docker CLI.::

   $ sudo gpasswd -a <username> docker

#. Log into a new group docker.::

   $ newgrp docker

#. Test to run a container.::

   $ docker run --rm hello-world
   ...
   Hello from Docker!
   This message shows that your installation appears to be working correctly.
   ...

#. Remove a test image.::

   $ docker rmi hello-world


kolla
-------

Build kolla using python virtual environment in debian 10 buster.

Prepare
++++++++

Install git, python3-venv and python3-dev (python3-dev is for compiling pypi
source build.).::

   $ sudo apt install -y git gcc python3-venv python3-dev

Create a virtual environment.::

   $ python3 -m venv .envs/kolla
   $ source .envs/kolla/bin/activate


Install wheel, kolla and tox.::

   (kolla) $ pip install wheel
   (kolla) $ pip install kolla tox

Get kolla train branch source.::

   (kolla) $ git clone -b stable/train https://github.com/openstack/kolla

Configuration
--------------

Copy kolla-build.conf.masakari to kolla source.::

   (kolla) $ cd kolla
   (kolla) $ cp <path/to/okidoki>/src/kolla/kolla-build.conf.masakari \
                  kolla-build.conf

Modify kolla-build.conf to use local masakari sources.::

   (kolla) $ vi kolla-build.conf
   [masakari-base]
   ...
   location = </path/to/okidoki>/src/masakari
   ...
   [masakari-monitors]
   ...
   location = </path/to/okidoki>/src/masakari-monitors

Build
------

Build the masakari images.::

   (kolla) $ python tools/build.py --config-file kolla-build.conf masakari-

Confirm the masakari images are built.::

   (kolla) $ docker images | grep masakri
   jijisa/centos-source-masakari-monitors   train               5a1dcea9255b        2 hours ago         1.01GB
   jijisa/centos-source-masakari-api        train               a435bd6c5f11        2 hours ago         845MB
   jijisa/centos-source-masakari-base       train               ef1ee38348fd        2 hours ago         845MB
   jijisa/centos-source-masakari-engine     train               a4a6c6c989b9        2 hours ago         845MB

