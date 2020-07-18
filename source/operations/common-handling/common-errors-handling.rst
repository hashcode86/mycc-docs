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
	
	# On [ipcc0, ipcc1]
	systemctl restart rabbitmq-server
	sleep 5; systemctl status rabbitmq-server
	
	# On [ipcc0] check rabbitmq-server cluster status 
	rabbitmqctl -n rabbit@ipcc0 cluster_status
	
Cluster is OK if you can see like this:	

.. code-block:: json

	Cluster status of node rabbit@ipcc0 ...
	[{nodes,[{disc,[rabbit@ipcc0,rabbit@ipcc1]}]},
	 {running_nodes,[rabbit@ipcc1,rabbit@ipcc0]},
	 {cluster_name,<<"rabbit@ipcc0.local">>},
	 {partitions,[]},
	 {alarms,[{rabbit@ipcc1,[]},{rabbit@ipcc0,[]}]}]
	
After that, init some rabbitmq exchanges & queues
	
.. code-block:: bash

	# [ipcc0, ipcc1] init exchange & queue
	
	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=kapps type=topic durable=false
	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=callevt type=topic durable=false
	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=callmgr type=topic durable=false
	
	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=acdc_status_stat_for_mycc type=topic durable=true
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="kapps" destination_type="exchange" destination="acdc_status_stat_for_mycc" routing_key="acdc_stats.status.*.*"

	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=acdc_member_call_for_mycc type=topic durable=true
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callmgr" destination_type="exchange" destination="acdc_member_call_for_mycc" routing_key="acdc.member.call.*.*"

	/usr/local/bin/rabbitmqadmin declare exchange --vhost=/ name=callevt_for_mycc type=topic durable=true
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callmgr" destination_type="exchange" destination="callevt_for_mycc" routing_key="acdc.member.call.*.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_ANSWER.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_BRIDGE.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_CREATE.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_DESTROY.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.DTMF.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_HOLD.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_TRANSFEREE.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_TRANSFEROR.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_UNBRIDGE.*"
	/usr/local/bin/rabbitmqadmin --vhost="/" declare binding source="callevt" destination_type="exchange" destination="callevt_for_mycc" routing_key="call.CHANNEL_UNHOLD.*"
	
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




