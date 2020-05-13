Swagger Mock Service
####################

Ngoài việc cho phép mô tả đặc tả về API, Swagger còn cho phép thực hiện API Mocking. Ví một một API Mocking chạy trên Swagger:

.. figure:: /_static/images/k8/api-gateway/mock-service-1.png
    :align: center
    :figwidth: 800px

Service trên (là một trong các Rest API của Etc Category service) dùng để ví dụ cho việc tích hợp các service chạy trong K8s với một mock service chạy tại môi trường bên ngoài:

* Service request:

.. code-block:: bash

	curl -X GET "https://virtserver.swaggerhub.com/thangdd7/ETCCategory/1.0.0/customers/genders" -H "accept: application/json"

* Service response:

.. code-block:: json

	[
	  {
		"code": "MAN",
		"value": "Nam giới"
	  },
	  {
		"code": "WOMAN",
		"value": "Nữ giới"
	  },
	  {
		"code": "MULTI",
		"value": "Đa giới tính"
	  }
	]

Kubernetes Services for Egress Traffic
**************************************

Script dưới đây để tạo một K8s ExternalName Service. ExternalName được sử dụng để tạo một local DNS alias (`etc-mock`) trỏ tới một external service (`virtserver.swaggerhub.com`):

.. code-block:: bash

  kubectl delete svc etc-mock
  kubectl apply -f - <<EOF
  kind: Service
  apiVersion: v1
  metadata:
    name: etc-mock
  spec:
    type: ExternalName
    externalName: virtserver.swaggerhub.com
    ports:
    - name: http
      protocol: TCP
      port: 80
  EOF

Thử truy cập tới API thông qua K8s service name: `etc-mock`

.. code-block:: bash

  kubectl run curl --image=radial/busyboxplus:curl -i --tty
  curl etc-mock.default.svc.cluster.local/thangdd7/ETCCategory/1.0.0/customers/genders

Category Service Ingress
************************

Để các Frontend UI truy cập được tới Service (`etc-mock`) chạy trong K8s ta tạo một Kong Ingress như dưới đây:

.. code-block:: bash

  kubectl delete kongplugin.configuration.konghq.com/etc-category-rewrite
  kubectl apply -f - <<EOF
  apiVersion: configuration.konghq.com/v1
  kind: KongPlugin
  metadata:
    name: etc-category-rewrite
  config:
    replace:
      uri: "/thangdd7/ETCCategory/1.0.0/\$(uri_captures[1])"
  plugin: request-transformer
  EOF
  
  ##
  
  kubectl describe ing etc-category-mock
  kubectl delete ingress etc-category-mock
  kubectl apply -f - <<EOF
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: etc-category-mock
    annotations:
      plugins.konghq.com: etc-category-rewrite
      kubernetes.io/ingress.class: "kong"
  spec:
    tls:
      - hosts:
        - api.127.0.0.1.nip.io
    rules:
    - host: api.127.0.0.1.nip.io
      http:
        paths:
        - backend:
            serviceName: etc-mock
            servicePort: 80
          path: /category/1.0.0/(.*)
    - host: etc-api.default.svc.cluster.local
      http:
        paths:
        - backend:
            serviceName: etc-mock
            servicePort: 80
          path: /category/1.0.0/(.*)
  EOF

Với Ingress trên:

* Client gửi request tới: http://api.127.0.0.1.nip.io/category/1.0.0/{THE_REST_REQUEST_PATH} 
* Hoặc tới: http://etc-api.default.svc.cluster.local/category/1.0.0/{THE_REST_REQUEST_PATH}
* Kong Ingress sẽ xử lý để gửi request tới backend service: etc-mock/thangdd7/ETCCategory/1.0.0/{THE_REST_REQUEST_PATH}


K8s Internal Services Communication
***********************************

Với ingress rules trên, mục đích để external client có thể giao tiếp tới Category service. Tuy nhiên để một K8s internal service gọi được tới service trên qua service name. Ta tạo một service cho mục đích giao tiếp nội bộ như dưới:

.. code-block:: bash

  kubectl delete svc etc-api
  kubectl apply -f - <<EOF
  kind: Service
  apiVersion: v1
  metadata:
    name: etc-api
  spec:
    type: ExternalName
    externalName: api.10.61.231.9.nip.io
    ports:
    - name: http
      protocol: TCP
      port: 80
  EOF

Thử truy cập tới API của Category service thông qua K8s service name: `etc-api`

.. code-block:: bash

  kubectl run curl --image=radial/busyboxplus:curl -i --tty
  curl etc-api.default.svc.cluster.local/category/1.0.0/customers/genders

