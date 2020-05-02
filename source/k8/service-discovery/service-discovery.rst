***************************************
Service-discovery với K8 Service Object
***************************************

Platform-provided service discovery
###################################

.. figure:: /_static/images/k8/service-discovery-1.png
    :align: center
    :figwidth: 600px

Interprocess (Service to Service) communication
###############################################

Vấn đề
******

Solution with K8
****************

Access external Kafka brokers qua service name
##############################################

Vấn đề
******

Kafka cluster là một stateful application, nó có yêu cầu rất cao về resource (Disk I/O, Network, CPU, Memory), do đó không khuyến cáo chạy Kafka trong K8-Cluster do sẽ phải setup rất chuyên sâu về network, shared disk Volume. Ngoài ra khi có vấn đề với Kafka sẽ rất khó để thực hiện troubleshooting. Tham khảo thêm https://www.confluent.io/blog/apache-kafka-kubernetes-could-you-should-you/

Một Kafka cluster tối thiểu sẽ cần 03 nodes (03 brokers). Khi Dev các service, ta sẽ phải có file cấu hình (ví dụ application.yml trong spring boot) để chứa thông tin về Kafka brokers này. Các vấn đề ta có thể dễ nhận thấy:

* Có rất nhiều các service cần phải cấu hình trỏ tới Kafka brokers
* Khi triển khai Application tại các môi trường khác nhau (Dev, Test, Staging, Production), thì mỗi môi trường sẽ có một Kafka Cluster riêng. Ta sẽ phải cấu hình lại Kafka brokers cho tất cả các service trong hệ thống

.. code-block:: yaml

  ---
  spring.profiles: dev
  spring.cloud.stream.kafka:
    streams:
      binder:
        brokers: 10.0.0.1:9092, 10.0.0.2:9092, 10.0.0.3:9092
		
Solution with K8
****************

Đối với mỗi môi trường ta vẫn cần triển khai một Kafka Cluster riêng. Ví dụ với môi trường Dev, ta có một cluster gồm 3 server như sau:

* 10.0.0.1:9092
* 10.0.0.2:9092
* 10.0.0.3:9092

Thực hiện khai báo 03 Service Object trong môi trường của K8-Cluster. Mỗi Service này có đặc điểm là:

* Service Name: ``kafka1``, ``kafka2`` hoặc ``kafka3``
* Các Service để có Port truy cập là ``9092``
* Kết nối ``kafka1:9092`` sẽ tương ứng với kết nối tới ``10.0.0.1:9092`` (cấu hình Endpoits)

.. code-block:: bash

	echo "
	kind: Service
	apiVersion: v1
	metadata:
	 name: kafka1
	spec:
	 ports:
	 - port: 9092
	   targetPort: 9092
	---
	kind: Endpoints
	apiVersion: v1
	metadata:
	 name: kafka1
	subsets:
	 - addresses:
		 - ip: 10.0.0.1
	   ports:
		 - port: 9092
	" | kubectl apply -f -

	echo "
	kind: Service
	apiVersion: v1
	metadata:
	 name: kafka2
	spec:
	 ports:
	 - port: 9092
	   targetPort: 9092
	---
	kind: Endpoints
	apiVersion: v1
	metadata:
	 name: kafka2
	subsets:
	 - addresses:
		 - ip: 10.0.0.2
	   ports:
		 - port: 9092
	" | kubectl apply -f -

	echo "
	kind: Service
	apiVersion: v1
	metadata:
	 name: kafka3
	spec:
	 ports:
	 - port: 9092
	   targetPort: 9092
	---
	kind: Endpoints
	apiVersion: v1
	metadata:
	 name: kafka3
	subsets:
	 - addresses:
		 - ip: 10.0.0.3
	   ports:
		 - port: 9092
	" | kubectl apply -f -

Trong cấu hình của tất cả các service thì khi này có thể truy cập tới Kafka Brokers qua service name thay vì địa chỉ IP:

.. code-block:: yaml

  ---
  spring.profiles: dev
  spring.cloud.stream.kafka:
    streams:
      binder:
        brokers: kafka1:9092, kafka2:9092, kafka3:9092

Khi triển khai tại các môi trường mới, ta sẽ khai thêm các Service Object (kafka1, kafka2, kafka3) tương tự như trên. Còn với các service thì không cần phải cấu hình lại thuộc tính brokers.