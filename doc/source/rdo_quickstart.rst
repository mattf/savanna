Install and setup Savanna on RDO
================================

Installing RDO - http://openstack.redhat.com
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, create an RDO installation all on localhost.

0. ``sudo yum install -y http://rdo.fedorapeople.org/openstack/openstack-grizzly/rdo-release-grizzly.rpm``
1. ``sudo yum install -y openstack-packstack``
2. ``sudo packstack --allinone --os-quantum-install=n``

START UNDER DEVELOPMENT
Alternatively, create a multi-host installation. You'll need to make sure you
have two network interfaces available and connecting all the hosts you
want in your installation. You will also need password-less ssh access
to all nodes, or have the root account password available when prompted.

2. ``sudo packstack --install-hosts=$(host $(hostname) | cut -f4 -d\
),node2's ip, node3's ip, etc`` --os-quantum-install=n --novanetwork-auto-assign-floating-ip=y

Note, --os-quantum-install=n, i.e. use nova-networking, is required
until Savanna supports quantum/neutron. The missing feature is
auto assignment of floating IPs. See https://blueprints.launchpad.net/savanna/+spec/add-neutron-support.
END UNDER DEVELOPMENT

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

START UNDER DEVELOPMENT
All set, unless you did a multi-host install. You'll need to setup
floating IPs. The floating IPs you get out of the box likely will not
work. You should delete them,

``nova floating-ip-bulk-delete 10.3.4.0/22``

And add the correct IPs, likely given to you by your network admins,

``nova floating-ip-bulk-create CORRECT-RANGE``
END UNDER DEVELOPMENT

These instructions have been verified using RHEL 6.4. A caveat, if you did an
--allinone install and while
https://bugzilla.redhat.com/show_bug.cgi?id=962605 is outstanding, you
must also run ``sudo iptables -A POSTROUTING -t mangle -p udp
--dport 68 -j CHECKSUM --checksum-fill``

Installing and running Savanna - https://savanna.readthedocs.org/en/latest/quickstart.html
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Until Savanna is available in EPEL, see
https://apps.fedoraproject.org/packages/s/savanna, you will need to
install using virtualenv and pip.

First, install virtualenv, python-devel and gcc (used during pip installs)

5. ``sudo yum install -y python-virtualenv python-devel gcc``

Second, setup an environment in which to install and run Savanna

6. ``virtualenv --no-site-packages savanna_env``
7. ``source savanna_env/bin/activate``
8. ``pip install savanna``

Third, setup the Savanna configuration file for your OpenStack
installation. Note, $OS_PASSWORD comes from step 4. TODO: Get Savanna to
use keystonerc_admin env variables and take auth host & port as a
single configuration parameter. Note, if you did a multi-host install,
you will want to use_floating_ips, so remove it from the sed line below.

9. ``export SAVANNA_CONF=~/savanna_env/share/savanna/savanna.conf``
10. ``sed -e "s/#os_auth_host=openstack/os_auth_host=127.0.0.1/" -e "s/#os_admin_password=nova/os_admin_password=$OS_PASSWORD/" -e "s/#use_floating_ips=true/use_floating_ips=false/" $SAVANNA_CONF.sample > $SAVANNA_CONF``

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
how to create one. The qcow2's MD5 hash is *ff14810d7b7ded3dc734dc163834fcf0*.

13. ``export IMAGE_URL=http://savanna-files.mirantis.com/savanna-0.1.2-hadoop.qcow2
; export IMAGE_NAME=$(basename $IMAGE_URL)``
14. ``wget $IMAGE_URL``

Second, create an image and record its id

15. ``glance image-create --name=$IMAGE_NAME --disk-format=qcow2 --container-format=bare < $IMAGE_NAME``
16. ``export BASE_IMAGE_ID=$(glance image-show $IMAGE_NAME | grep ' id ' | awk '{print $4}')``

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
savanna-0.1.2-hadoop.qcow2 is ``swordfish``, and run your expected hadoop
commands.
