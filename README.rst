#############
DevSecOps Lab
#############

Accompanying repository for the courses on Automated Security Testing on
Pluralsight. Please note that these instructions can slightly differ from the
ones being shown during the courses - this repository is leading.

+ https://app.pluralsight.com/library/courses/approaching-automated-security-testing-devsecops/table-of-contents
+ https://app.pluralsight.com/library/courses/performing-devsecops-automated-security-testing/table-of-contents
+ https://app.pluralsight.com/library/courses/integrating-automated-security-testing-tools/table-of-contents

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
  ``docker-compose.override.yml``, and ensure that the group ID is the same as
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

The default Jenkins password can be read from the jenkins container:

.. code-block:: console

   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

The default GitLab password for user ``root`` can be read from the gitlab
container:

.. code-block:: console

   docker exec gitlab cat /etc/gitlab/initial_root_password


