#############
DevSecOps Lab
#############

Accompanying repository for 3 courses on Automated Security Testing on
Pluralsight

**************
What is this ?
**************

The files in this repository allow you to quickly spin up a lab, consisting of

+ Jenkins, a build and automation server
+ GitLab, a git server
+ Sonarqube, a code quality server
+ Docker registry, a Docker registry server

************
How to start
************

+ Ensure you have ``docker`` and ``docker-compose`` installed, and that you have
  the correct permissions to execute the binaries

  .. code-block:: console

     docker version
     docker-compose version

+ Clone this repository

  .. code-block:: console

     git clone https://github.com/PeterMosmans/devsecops-lab
     cd devsecops-lab

+ Find out the group ID of ``docker``:

  .. code-block:: console

     getent group docker

+ Copy the file ``docker-compose.override.example.yml`` to
  ``docker-compose-override.yml``, and ensure that the group ID is the same as
  your local ``docker`` group ID. In the example, the group ID is ``999``:

    .. code-block:: yaml

       ---
       version: "3.5"

       services:
         jenkins:
          user: ":999"

That's it! Now you can execute ``docker-compose up --detach`` in this
directory - this will spin up the servers in the background.

By default, Jenkins will listen on port 8080 (http), Gitlab on port 80 (http)
and 7722 (ssh), and Sonarqube on port 8080 (http). You can override the port
numbers in ``docker-compose.override.yml``. See the ``docker-compose.yml`` file
for the correct syntax.
