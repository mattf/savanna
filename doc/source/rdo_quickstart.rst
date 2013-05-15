Install and setup Savanna on RDO
================================

Installing RDO - http://openstack.redhat.com
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, create an RDO installation all on localhost.

0. ``sudo yum install -y http://rdo.fedorapeople.org/openstack/openstack-grizzly/rdo-release-grizzly-3.noarch.rpm``
1. ``sudo yum install -y openstack-packstack``
2. ``sudo packstack --allinone``

Second, stash and load the credentials needed to administer the
installation.

3. ``sudo cp /root/keystonerc_admin . ; sudo chmod a+r keystonerc_admin``
4. ``source keystonerc_admin``

At this point you have a functional OpenStack Grizzly installation on
a single machine. The instance network will only be accessible from
this machine, so Savanna must also be run on this single machine. It
is possible to setup floating IPs for the installation and run Savanna
on a separate machine, but that's an advanced topic and out of scope
for these instructions.

These instructions have been verified using RHEL 6.4. A caveat, while
https://bugzilla.redhat.com/show_bug.cgi?id=962605 is outstanding, add
a step after 2. ``sudo iptables -A POSTROUTING -t mangle -p udp
--dport 68 -j CHECKSUM --checksum-fill``

Installing and running Savanna - https://savanna.readthedocs.org/en/latest/quickstart.html
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Savanna has some dependencies that are not provided in RPM form and
some dependencies that are provided as RPMs at a different
version. This introduces complexity, especially since there is overlap
between the python packages needed by RDO and by Savanna
(e.g. pycrypto, sqlalchemy and paramiko). To simplify the setup, we'll
use virtualenv to create an environment for Savanna that is isolated
from the system environment that RDO is using. Note: There is work to
be done here to package Savanna's dependencies and resolve version
conflicts.

First, install virtualenv, python-devel and gcc (used during pip installs)

5. ``sudo yum install -y python-virtualenv python-devel gcc``

Second, setup an environment in which to install and run Savanna

6. ``virtualenv --no-site-packages savanna_env``
7. ``source savanna_env/bin/activate``
8. ``pip install savanna``

Third, setup the Savanna configuration file for your OpenStack
installation. Note, Savanna API will run on port 18080, because Swift
is already running on port 8080. Note, allow_cluster_ops=true is
necessary until https://bugs.launchpad.net/savanna/+bug/1180151 is
resolved. Note, $OS_PASSWORD comes from step 4. TODO: Get Savanna to
use keystonerc_admin env variables and take auth host & port as a
single configuration parameter.

9. ``export SAVANNA_CONF=~/savanna_env/share/savanna/savanna.conf``
10. ``sed -e "s/#port=8080/port=18080/" -e "s/#allow_cluster_ops=false/allow_cluster_ops=true/" -e "s/#os_auth_host=openstack/os_auth_host=127.0.0.1/" -e "s/#os_admin_password=nova/os_admin_password=$OS_PASSWORD/" -e "s/#use_floating_ips=true/use_floating_ips=false/" $SAVANNA_CONF.sample > $SAVANNA_CONF``

Finally, initialize the Savanna database and start the Savanna API

11. ``savanna-manage --config-file $SAVANNA_CONF reset-db --with-gen-templates``
12. ``savanna-api --config-file $SAVANNA_CONF``

At this point you have a running OpenStack Grizzly instance and the
Savanna API service running. However, there are no images in glance
for Savanna to use, and if there were the default security group would
not let you access them as instances.

Setting up Savanna and OpenStack - https://savanna.readthedocs.org/en/latest/quickstart.html
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, get your hands on a working Savanna disk image. TODO: Describe
how to create one. The tar.gz's MD5 hash is *4998e403f559e85be1a5955186a5638e*.

13. ``wget http://savanna-files.mirantis.com/savanna-0.1-hdp-img.tar.gz``

Second, create an image and record its id

14. ``tar xzf savanna-0.1-hdp-img.tar.gz``
15. ``glance image-create --name=hdp.image --disk-format=qcow2 --container-format=bare < savanna-0.1-hdp-img.img``
16. ``export BASE_IMAGE_ID=$(glance image-show hdp.image | grep ' id ' | awk '{print $4}')``

Third, open up the SSH (22) port on the default security group

17. ``nova secgroup-add-rule default tcp 22 22 0.0.0.0/0``

Now, all the necessary pieces are in place to start a Hadoop cluster with
Savanna on OpenStack now. The only thing left to do is try.

Create a Hadoop cluster
~~~~~~~~~~~~~~~~~~~~~~~

First, get an http command-line client

18. ``easy_install httpie``

Second, send a request to Savanna to create a new cluster. Note,
$BASE_IMAGE_ID is from step 16.

19. ``export TOKEN=$(keystone token-get | grep ' id' | awk '{print $4}')``
20. ``export TENANT=$(keystone tenant-get $OS_TENANT_NAME | grep ' id ' | awk '{print $4}')``
21. ``echo "{ \"cluster\": { \"name\": \"hdp-$(date +%s)\", \"node_templates\": { \"jt_nn.small\": 1, \"tt_dn.small\": 3 }, \"base_image_id\": \"$BASE_IMAGE_ID\" } }" | http http://127.0.0.1:18080/v0.2/$TENANT/clusters X-Auth-Token:$TOKEN``

You can now access the Savanna API to interact with your cluster and
discover information, such as the JobTracker & NameNode IP
address. You can SSH to that IP as root, the password on
savanna-0.1-hdp-img is ``swordfish``, and run your expected hadoop
commands.
