Install and setup Savanna on RDO
================================

Installing RDO - http://openstack.redhat.com
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, create an RDO installation all on localhost.

0. ``sudo yum install -y http://rdo.fedorapeople.org/openstack/openstack-grizzly/rdo-release-grizzly.rpm``
1. ``sudo yum install -y openstack-packstack``
2. ``sudo packstack --allinone --os-quantum-install=n``

Note, make sure your systems have access to updates, the installation
requires selinux-policy >= 3.7.19-195.el6_4.2.

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
installation. Note, $OS_PASSWORD comes from step 4. Note, if you did a
multi-host install, you will want to use_floating_ips, so remove it
from the sed line below.

9. ``export SAVANNA_CONF=~/savanna_env/share/savanna/savanna.conf``
10. ``sed -e "s/#os_admin_password=nova/os_admin_password=$OS_PASSWORD/" -e "s/#use_floating_ips=true/use_floating_ips=false/" $SAVANNA_CONF.sample > $SAVANNA_CONF``

Finally, initialize the Savanna database and start the Savanna API

11. ``savanna-api --config-file $SAVANNA_CONF``

At this point you have a running OpenStack Grizzly instance and the
Savanna API service running. However, there are no images in glance
for Savanna to use, and if there were the default security group would
not let you access them as instances.

Setting up Savanna and OpenStack - https://savanna.readthedocs.org/en/latest/quickstart.html
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, get your hands on a working Savanna disk image. TODO: Describe
how to create one. The qcow2's MD5 hash is *10856243711041fac360b1840ccb51dd*.

12. ``export IMAGE_URL=http://savanna-files.mirantis.com/savanna-0.2-vanilla-1.1.2-ubuntu-12.10.qcow2 ; export IMAGE_NAME=$(basename $IMAGE_URL)``
13. ``wget $IMAGE_URL``

Second, create an image and record its id

14. ``glance image-create --name=$IMAGE_NAME --disk-format=qcow2 --container-format=bare < $IMAGE_NAME``
15. ``export BASE_IMAGE_ID=$(glance image-show $IMAGE_NAME | grep ' id ' | awk '{print $4}')``

Third, open up the SSH (22) port on the default security group

16. ``nova secgroup-add-rule default tcp 22 22 0.0.0.0/0``

Now, all the necessary pieces are in place to start a Hadoop cluster with
Savanna on OpenStack now. The only thing left to do is try.

Create a Hadoop cluster
~~~~~~~~~~~~~~~~~~~~~~~

First, get an http command-line client

17. ``easy_install httpie``

Second, register and tag the image. Note, $BASE_IMAGE_ID is from step 15.

18. ``export TOKEN=$(keystone token-get | grep ' id' | awk '{print $4}')``
19. ``export TENANT=$(keystone tenant-get $OS_TENANT_NAME | grep ' id ' | awk '{print $4}')``
20. ``export SAVANNA_URL="http://localhost:8386/v1.0/$TENANT"``
21. ``http POST $SAVANNA_URL/images/$BASE_IMAGE_ID X-Auth-Token:$TOKEN username=ubuntu``
22. ``http POST $SAVANNA_URL/images/$BASE_IMAGE_ID/tag X-Auth-Token:$TOKEN tags:='["vanilla", "1.1.2"]'``

Third, create node group templates, one for the master node and one for
the worker nodes, and record their IDs.

23. ``echo '{"name": "master-tmpl", "flavor_id": "2", "plugin_name": "vanilla", "hadoop_version": "1.1.2", "node_processes": ["jobtracker", "namenode"] }' | http POST $SAVANNA_URL/node-group-templates X-Auth-Token:$TOKEN``
24. ``export MASTER=*id from step 23*``
25. ``echo '{"name": "worker-tmpl", "flavor_id": "2", "plugin_name": "vanilla", "hadoop_version": "1.1.2", "node_processes": ["tasktracker", "datanode"] }' | http POST $SAVANNA_URL/node-group-templates X-Auth-Token:$TOKEN``
26. ``export WORKER=*id from step 25*``

Fourth, create a cluster template consisting of one master and 2
workers. Also, record the cluster template's ID.

27. ``echo "{\"name\": \"cluster-template\", \"plugin_name\": \"vanilla\", \"hadoop_version\": \"1.1.2\", \"node_groups\": [ { \"name\": \"master\", \"node_group_template_id\": \"$MASTER\", \"count\": 1 }, { \"name\": \"workers\", \"node_group_template_id\": \"$WORKER\", \"count\": 2 } ] }" | http $SAVANNA_URL/cluster-templates X-Auth-Token:$TOKEN``
28. ``export CLUSTER=*id from step 27*``

Fifth, upload a keypair to use with the cluster.

29. ``nova keypair-add keypair0 --pub-key ~/.ssh/id_rsa.pub``

Finally, create the cluster.

30. ``echo "{ \"name\": \"cluster-1\", \"plugin_name\": \"vanilla\", \"hadoop_version\": \"1.1.2\", \"cluster_template_id\" : \"$CLUSTER\", \"user_keypair_id\": \"keypair0\", \"default_image_id\": \"$BASE_IMAGE_ID\" }" | http $SAVANNA_URL/clusters X-Auth-Token:$TOKEN``

You can now access the Savanna API to interact with your cluster and
discover information, such as the JobTracker & NameNode IP
address. You can SSH to that IP as ubuntu using your ssh keypair, and
run your expected hadoop commands.
