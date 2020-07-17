COMMON ERRORS HANDLING
######################

Unable to complete user Webphone Registration
*********************************************

When user is logged in to myCC, but Webphone Button failed when performing SIP Registration process. it show like this for ALL USERS:

.. figure:: /_static/images/operations/common-handling/common-errors-handling/sip-registration-is-in-process.png
    :align: center
    :figwidth: 600px

Restart {kazoo-kamailio}
========================

restart module `{kazoo-kamailio}` on `ipcc0`, `ipcc1`, `ipcc20`, `ipcc21`:

.. code-block:: bash

	#su root
	systemctl restart kazoo-kamailio
	
	#check kazoo-kamailio is running (Active: active (running))
	sleep 3; systemctl status kazoo-kamailio

Restart {kazoo}
===============

if `{kazoo-kamailio}` restarting as above does not help, su root and restart `{kazoo}` on `ipcc0`, `ipcc1` as bellow:


* Stop {sip-proxy} to stop receiving all incoming calls from MSC

.. code-block:: bash

	# On [ipcc0, ipcc1] 
	systemctl stop kazoo-kamailio

* Restart {rabbitmq-server} cluster:

.. code-block:: bash
	
	# On [ipcc1]
	systemctl restart rabbitmq-server
	sleep 5; systemctl status rabbitmq-server
	
	# On [ipcc1] check rabbitmq-server cluster status 
	rabbitmqctl -n rabbit@ipcc1 cluster_status
	
Cluster is OK if you can see like this:	

.. code-block:: json

	Cluster status of node rabbit@ipcc1
	...
	
* Restart {kazoo}:

.. code-block:: bash
	
	# On [ipcc0, ipcc1]
	
	systemctl stop kazoo-applications;
	sleep 5
	systemctl restart kazoo-applications;
	
	sleep 60
	HOSTNAME=$(hostname -s)
	if [[ $HOSTNAME == *"ipcc0"* ]]; then
	  sup kapps_controller start_app acdc;
	fi

* Let agent logged in, and start {kazoo-kamailio} to start receiving new incoming calls from MSC

.. code-block:: bash

	# On [ipcc0, ipcc1]
	systemctl start kazoo-kamailio




