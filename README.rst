=============================
Database as a Service (Trove)
=============================

General information about the Openstack Trove component can be found at:

https://wiki.openstack.org/wiki/Trove

This repository provides the following::

    - `System Architecture and Bill of Materials <https://github.com/open-power-ref-design/dbaas/blob/master/dbaas.pdf>`_
    - `Deployment configuration <https://github.com/open-power-ref-design/dbaas/blob/master/dbaas.xml>`_

The Architecture file provides a description and representation of Database
as a Service that is tuned for OpenPOWER servers.  It provides information
such as model numbers and feature codes to simplify the ordering process
and it provides racking and cabling rules for the preferred layout of
servers, switches, and cables.

The Deployment Configuration file provides a mapping of servers and switches
to software for the purposes of deployment.  Each server is mapped to a set
of OpenStack based software roles constituting the control plane, compute
plane, and storage plane.  Each role is defined in terms of operating system
system based resources such as users and networks that need to be configured
to satisfy that role.

The Deployment Configuration file needs to be edited so that it reflects the
configuration that is to be installed.  This is mostly a matter of making sure
that the numbers of servers represented match the number of servers to be 
installed and that IP addresses are allocated so that the installation is 
properly integrated into the data center.   

The installation process is split into two parts::

    > Bare metal installation of Linux 
    > Installation of OpenStack, Ceph storage, and DBaaS

The Deployment configuration file is fed into the bare metal installation
process which is performed by cluster-genesis.  The operating system is loaded
and configured as specified in the configuration file.  Users, networks, and
switches are installed during this step.  The last step is to invoke a small
script that installs OpenStack and DBaaS which is performed by os-services.

More properly, in the bare metal installation step, only the installation tools
for OpenStack and Ceph were installed, not the actual services.  The next step
is to configure these tools, so that they install the actual services in a
prescribed manner so that they fit properly in the data center.  The two 
projects are os-services and ceph-services.  See the README files of each project
to determine what is required here.

hint..: Only os-services needs to be configured

The final step is to include cluster-create.sh in the os-services
repository to install and configure the cluster.  This includes the formation of
the storage cluster by ceph-services and the installation and 
configuration of operational management by opsmgr.  os-services orchestrates
the installation process of OpenStack, Ceph, and Operational Management.  It is
loaded into the first controller that is setup by cluster-genesis.

The OpenStack dashboard may be reached through your browser:

http://<ipaddr of the first OpenStack control node>

This recipe also includes an operational management console which is
integrated into the openstack dashboard.  It monitors the cloud infrastructure
itself and is a valuable tool that shows capacity and utilization metrics
and provides an indication of health and may be configured to receive alerts
when components fail.  It is provided through the opsmgr repository.

Getting Started
---------------

The toolkit runs on an Ubuntu 16.04 OpenPOWER server or VM that is connected
to the internet and management switch in the cluster to be configured.

#. Read `dbaas.pdf <https://github.com/open-power-ref-design/dbaas/blob/master/bom/dbaas.pdf>`_

#. Rack and cable hardware as indicated

#. Get a local copy of this repository::

   $ git clone git://github.com/open-power-ref-design/dbaas
   $ cd dbaas
   $ TAG=$(git describe --tags $(git rev-list --tags --max-count=1))
   $ git checkout $TAG
   $ CFG=$(pwd)/config.yml

#. Edit the configuration file:

   Instructions for editing the file are included in the file itself.

   Additional information may be found in the
   *TBD-cluster-genesis-user-guide*.

#. Validate the configuration file::

   $ git clone git://github.com/open-power-ref-design/os-services
   $ cd os-services
   $ git checkout $TAG
   $ ./scripts/validate_config.py --file $CFG
   
#. Place the configuration file::

   $ git clone git://github.com/open-power-ref-design-toolkit/cluster-genesis
   $ cd cluster-genesis
   $ git checkout 1.0.1
   $ cp $CFG .

#. Invoke cluster-genesis to perform the bare metal installation process:

   Instructions may be found in cluster-genesis.

#. Wait for cluster-genesis to complete, ~3 hours:

#. Edit the OpenStack Installer configuration file:

   OpenStack installation is performed by openstack-ansible.  Instructions 
   for editing the user configuration files of openstack is described in 
   general terms in os-services.  Instructions specifically for DBaaS service
   are described below in the Trove Installation section.

#. Invoke the toolkit again to complete the installation::

   $ ./scripts/create-cluster

   Note this command is invoked on the first controller node.  The commands
   listed above are invoked on the deployer node.  When cluster-genesis completes,
   it displays on the screen instructions for invoking the command above. 

Trove Installation
------------------

The Openstack Trove component provides the DBaaS feature. 

The following files are installed for Trove:

+-------------------+-----------------------------------------------------------+
| Primary installer | ``/opt/openstack-ansible/playbooks/os-trove-install.yml`` |
+-------------------+-----------------------------------------------------------+
| Ansible role      | ``/etc/ansible/roles/power_trove/``                       |
+-------------------+-----------------------------------------------------------+
| Passwords         | ``/etc/openstack_deploy/user_secrets_trove.yml``          |
+-------------------+-----------------------------------------------------------+
| Container defns   | ``/etc/openstack_deploy/env.d/trove.yml``                 |
+-------------------+-----------------------------------------------------------+

See README.rst in os-services for more details. 

Customization
-------------

The following parameters can be customized:

* ``/etc/openstack_deploy/user_variables_trove.yml`` (required)

  ``trove_infra_subnet_alloc_start: "172.29.236.100"
  trove_infra_subnet_alloc_end: "172.29.236.110"``

  Trove requires access to the infrastructure network shared by other Openstack
  components. The above variables need to be set to limit the set of IP addresses
  that Trove will use from that network. The addresses must belong to the
  container infrastructure network defined in the inventory file
  ``/etc/openstack_deploy/openstack_user_config.yml``. The definition of that
  network is of the form::

   cidr_networks:
     container: 172.29.236.0/22

  NOTE that the ``openstack_user_config.yml`` file **must** contain a
  ``used-ips`` section that contains the same address range.

* ``/etc/openstack_deploy/user_secrets_trove.yml`` (optional)

  This contains passwords which are generated during the create-cluster phase.
  Any fields that are manually filled in after the bootstrap-cluster phase will
  not be touched by the automatic password generator during the create-cluster
  phase.

Verifying an install
--------------------
After successful installation, verify that Trove services are running correctly.

* Check for existence of Trove container(s) using ``lxc-ls -f`` on the
  controller nodes.

* Attach Trove container using ``lxc-attach -n <container name>``

* Check for existence of 3 Trove processes::

  - trove-api
  - trove-conductor
  - trove-taskmanager

* Source the environment file::

  $ source /root/openrc

* Run some sample trove commands and ensure they run without any errors::

  $ trove list
  $ trove datastore-list
  $ trove flavor-list

Using Trove
-----------

The next step is to build Trove guest images containing database software and
Trove guest agent software, upload them to Glance, and update the Trove
datastore list to map the Glance images to the database versions. Further
details of this process can be found at:
http://docs.openstack.org/developer/trove/#installation-and-deployment

Related projects
----------------

Recipes for OpenPOWER servers are located here:

    - `Recipe directory <https://github.com/open-power-ref-design/>`_

Here, you will find several OpenStack based recipes:

    - `Private cloud w/ and w/o Swift Object Storage <https://github.com/open-power-ref-design/private-compute-cloud/blob/master/README.rst>`_
    - `Database as a Service (OpenStack Trove) <https://github.com/open-power-ref-design/dbaas/blob/master/README-dbaas.rst>`_
    - `Standalone Swift Clusters (OpenStack Swift) <https://github.com/open-power-ref-design/standalone-swift/blob/master/README-swift.rst>`_
    - `Standalone Ceph Clusters <https://github.com/open-power-ref-design/standalone-ceph/blob/master/README-swift.rst>`_

The following projects provides services that are used as major building blocks in
recipes:

    - `cluster-genesis <https://github.com/open-power-ref-design-toolkit/cluster-genesis>`_
    - `ceph-services <https://github.com/open-power-ref-design-toolkit/ceph-services>`_
    - `opsmgr <https://github.com/open-power-ref-design-toolkit/opsmgr>`_

