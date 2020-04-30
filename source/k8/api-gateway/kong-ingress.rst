API Gateway & Demo backend service setup
########################################

Deploy Kong Ingress Controller
******************************
.. code-block:: bash

	kubectl create -f https://bit.ly/k4k8s

Check các k8 object đã được tạo trong namespace ``kong``

.. code-block:: bash

	kubectl get all -n kong
	
.. code-block:: bash

	NAME                                READY   STATUS    RESTARTS   AGE
	pod/ingress-kong-794c58c6c9-r2m6j   2/2     Running   1          57s

	NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)
	service/kong-proxy                LoadBalancer   10.101.241.58    localhost     80:32426/TCP,443:32008/TCP
	service/kong-validation-webhook   ClusterIP      10.107.221.249   <none>        443/TCP

	NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/ingress-kong   1/1     1            1           57s

	NAME                                      DESIRED   CURRENT   READY   AGE
	replicaset.apps/ingress-kong-794c58c6c9   1         1         1       57s
	
Như thông tin của ``service/kong-proxy`` để truy cập thử tới địa chỉ dịch vụ mà kong đưa ra bên ngoài ta thử với

.. code-block:: bash
	
	export PROXY_IP=127.0.0.1
	
	curl $PROXY_IP
	#{"message":"no Route matched with those values"}

Tại đây, cơ bản ta đã có một Kong Ingress Proxy hoạt động trên cổng 80

Setup httpbin service 
******************

httpd service là một demo cho rest service, nhận http request và trả về json response. Vai trò của nó dùng ví dụ cho các setup chi tiết của API gateway

.. code-block:: bash

  kubectl apply -f https://bit.ly/k8s-httpbin


Basic proxy
***********

Cài đặt dưới đây sẽ thiết lập rule cho proxy, để request gửi tới ``http://localhost/get`` hoặc ``http://localhost/post`` sẽ được forward tới httpbin-svc

.. code-block:: bash

  echo '
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: demo-get
    annotations:
      konghq.com/strip-path: "false"
  spec:
    rules:
    - http:
        paths:
        - path: /get
          backend:
            serviceName: httpbin
            servicePort: 80
  ' | kubectl apply -f -

  echo '
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: demo-post
    annotations:
      konghq.com/strip-path: "false"
  spec:
    rules:
    - http:
        paths:
        - path: /post
          backend:
            serviceName: httpbin
            servicePort: 80
  ' | kubectl apply -f -


Check proxy & rule đã hoạt động:

.. code-block:: bash

  curl -i $PROXY_IP/get
  
.. code-block:: bash

	HTTP/1.1 200 OK
	Content-Type: application/json
	Content-Length: 252
	Connection: keep-alive
	Server: gunicorn/19.9.0
	Date: Tue, 28 Apr 2020 03:16:43 GMT
	Access-Control-Allow-Origin: *
	Access-Control-Allow-Credentials: true
	X-Kong-Upstream-Latency: 3
	X-Kong-Proxy-Latency: 1
	Via: kong/2.0.3

	{
	  "args": {},
	  "headers": {
		"Accept": "*/*",
		"Connection": "keep-alive",
		"Host": "127.0.0.1",
		"User-Agent": "curl/7.58.0",
		"X-Forwarded-Host": "127.0.0.1"
	  },
	  "origin": "192.168.65.3",
	  "url": "http://127.0.0.1/get"
	}

Add JWT authentication
######################

Kong Gateway có thể setup để thực hiện JWT authentication tất cả các request trước khi gửi tới backend service (httpbin-svc). Trước tiên mô tả về JWT được sử dụng trong ETC

Keycloak JWT
************

Hệ thống ETC sử dụng Keycloak với vai trò Authorization Server; Sau khi login, client sẽ được cấp một access token (JWT) như ví dụ dưới đây:

.. code-block:: bash

  eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJsTUViQ1U4eW5sY3hyZFBEUU5VY01rcGhaUkc1YjBRNmVfb282bU1lSmNZIn0.eyJleHAiOjE1ODgwMzg3MTQsImlhdCI6MTU4ODAzODQxNCwianRpIjoiZDVhYTA3MzgtMjE3MC00NGE1LThjNzgtY2YyZTk5N2M3MjA5IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL2F1dGgvcmVhbG1zL2V0YyIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiIzZTgyNzM4Ni1hYWRiLTRlYWEtYjE4MS1jNGY4MWNmODMwZjkiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJpbSIsInNlc3Npb25fc3RhdGUiOiIwYmNiZTI1YS0yZGM1LTQ3YzUtODI1OC0yNmMxNjI5ZGQyM2UiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6InByb2ZpbGUgZW1haWwiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJUaGFuZyBEbyIsInByZWZlcnJlZF91c2VybmFtZSI6InRoYW5nZGQ4IiwibG9jYWxlIjoidmktdm4iLCJnaXZlbl9uYW1lIjoiVGhhbmciLCJmYW1pbHlfbmFtZSI6IkRvIiwiZW1haWwiOiJ0aGFuZ2RkOEB2aWV0dGVsLmNvbS52biJ9.otXvsOAWIHqZ9B9eWGG3aDqt5nhiSBebkTehI2jSShx4X210AOqrIm7JU01QAiVmRrx6Beuc0NnY8b856fqtOXbttWHmXCQjagwmloB1TsstpB02H_QcyYJnDXjpzP_Y04dkhaTh0xM7jqGcjDi8JV_NQDfIyhFy9dNttHqOaNdqo__Mlyl0KKjVPtGUpL1tOiRWmmjxQ9OX5yjEPM9AoYtb6RH9IzI2HU_99ux40dZ1LIcvhPiagZQJNG02-HEQ6xHwSFdCmCqrhHEP3rPGrTJ0mPEBVt_HQ5X8gZLDoDYAM3-jgctgKMGwdFZ26NqXHA_Yb_OvEqXWqqB48s6tig


Decode của token này qua: https://jwt.io/

HEADER:

.. code-block:: javascript

	{
	  "alg": "RS256",
	  "typ": "JWT",
	  "kid": "lMEbCU8ynlcxrdPDQNUcMkphZRG5b0Q6e_oo6mMeJcY"
	}

PAYLOAD:

.. code-block:: javascript

	{
	  "exp": 1588038714,
	  "iat": 1588038414,
	  "jti": "d5aa0738-2170-44a5-8c78-cf2e997c7209",
	  "iss": "http://localhost:8080/auth/realms/etc",
	  "realm_access": {
		"roles": [
		  "offline_access",
		  "uma_authorization"
		]
	  },
	  "preferred_username": "thangdd8",
	  "locale": "vi-vn",
	  "given_name": "Thang",
	  "family_name": "Do",
	  "email": "thangdd8@viettel.com.vn"
	  ...
	}

Đặc điểm của jwt access token này:

* alg: RS256 - Được ký số bằng algorithm: RS256 (thuật toán bất đối xứng sử dụng cặp khóa public key & private key)
* kid: id của cặp khóa, được quản lý trong Keycloak
* iss: token được cấp bởi http://localhost:8080/auth/realms/etc - là một định doanh của một tennent trên keycloak
* public key dùng để xác thực jwt này (có thể thử trên https://jwt.io/):

.. code-block:: bash

	-----BEGIN PUBLIC KEY-----
	MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArH3ZToJNBvodbk008mW7
	bU8phz91Ao6UVaa12JmI7+b3m2ym7qxS4XdSLNOqEiC9FEC5AiYZTecbmwZXNR0q
	tvKfOcTrpFBwcI/Vv/oqHEreg5Db0Ee/3Hwry771vtWvYVcHspK2yFf9HyTCEKXu
	2oTny556fE2QxkM0hYF/MAoF+f4K5DEFKXkBcp0GqkKk1A1TsBO2as0x3YH4tOpJ
	qjNQtkeeRSEXL90xXwjaGlCfS88OWkEgtD0+Y634u2z02ensbdPA6EuvDuiI9fNS
	MYGubGjwBL8PNutj5XXSIef/SiCJBitWkF7xdLEtYIbbzw/J2bljRA5P0mHKvuoS
	pQIDAQAB
	-----END PUBLIC KEY-----

Các setup tiếp theo đây sẽ yêu cầu API Gateway thực hiện JWT Authentication. Với mỗi Token được gửi tới, token được decode để:

* Lấy thông tin kid hoặc iss (ở đây chọn kid)
* Từ kid ở trên tìm trong cấu hình gateway để lấy ra public key (được setup từ trước) + algorithm RS256 --> xác định jwt có hợp lệ hay không
* Nếu Hợp lệ, mới cho phép gửi request vào httpbin-svc bên trong.

Enable JWT authentication
*************************

.. code-block:: bash

  echo "
  apiVersion: configuration.konghq.com/v1
  kind: KongPlugin
  metadata:
    name: app-jwt
  config:
    key_claim_name: kid
  plugin: jwt
  " | kubectl apply -f -

Liên kết ``plugin jwt`` với 2 Ingress rule đã tạo trước đó:

.. code-block:: bash

  echo '
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: demo-get
    annotations:
      plugins.konghq.com: app-jwt
      konghq.com/strip-path: "false"
  spec:
    rules:
    - http:
        paths:
        - path: /get
          backend:
            serviceName: httpbin
            servicePort: 80
  ' | kubectl apply -f -

  echo '
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: demo-post
    annotations:
      plugins.konghq.com: app-jwt
      konghq.com/strip-path: "false"
  spec:
    rules:
    - http:
        paths:
        - path: /post
          backend:
            serviceName: httpbin
            servicePort: 80
  ' | kubectl apply -f -

Tới đây backend service httpbin-svc đã được bảo vệ và yêu cầu JWT Authentication bởi API Gateway

.. code-block:: bash

  curl -i $PROXY_IP/get
  
  #{"message":"Unauthorized"}
  

Ta sẽ nhận được một ``401 response`` kèm theo message ``{"message":"Unauthorized"}`` cho biết rằng request không được cho phép. 

Tới đây tất cả các request gửi tới ``http://localhost/get`` hoặc ``http://localhost/post`` đều phải kèm theo ``access token (JWT)``. Tuy nhiên ta cần setup thêm cho API Gateway để nó có thể verify được token là hợp lệ thông qua chữ ký số bên trong nó (https://auth0.com/docs/tokens/guides/validate-jwts). Token phải được check hợp lệ mới có thể được gửi tới backend service.

Provision Consumers
*******************

.. code-block:: bash

  echo "
  apiVersion: configuration.konghq.com/v1
  kind: KongConsumer
  metadata:
    name: etckongconsumer
  username: etckongconsumer
  " | kubectl apply -f -

Secrets
*******

Secret cho phép API Gateway validate tính hợp lệ của chữ ký số trong jwt

.. code-block:: bash

  kubectl create secret generic etc-jwt-secret  \
    --from-literal=kongCredType=jwt  \
    --from-literal=key='lMEbCU8ynlcxrdPDQNUcMkphZRG5b0Q6e_oo6mMeJcY' \
    --from-literal=algorithm=RS256 \
    --from-literal=rsa_public_key="-----BEGIN PUBLIC KEY-----
  MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArH3ZToJNBvodbk008mW7
  bU8phz91Ao6UVaa12JmI7+b3m2ym7qxS4XdSLNOqEiC9FEC5AiYZTecbmwZXNR0q
  tvKfOcTrpFBwcI/Vv/oqHEreg5Db0Ee/3Hwry771vtWvYVcHspK2yFf9HyTCEKXu
  2oTny556fE2QxkM0hYF/MAoF+f4K5DEFKXkBcp0GqkKk1A1TsBO2as0x3YH4tOpJ
  qjNQtkeeRSEXL90xXwjaGlCfS88OWkEgtD0+Y634u2z02ensbdPA6EuvDuiI9fNS
  MYGubGjwBL8PNutj5XXSIef/SiCJBitWkF7xdLEtYIbbzw/J2bljRA5P0mHKvuoS
  pQIDAQAB
  -----END PUBLIC KEY-----"
  
  #Nếu jwt được ký bằng HS512 
  #kubectl create secret generic etc-jwt-secret-4  \
    #--from-literal=kongCredType=jwt  \
    #--from-literal=key="hashcode" \
    #--from-literal=algorithm=HS256 \
    #--from-literal=secret="hashcode@2020000000000000000000000000000000000000000000000000000"

.. code-block:: bash

  echo "
  apiVersion: configuration.konghq.com/v1
  kind: KongConsumer
  metadata:
    name: etckongconsumer
  username: etckongconsumer
  credentials:
    - etc-jwt-secret
  " | kubectl apply -f -

Rate limit
##########

Others
######

Tham khảo
#########
https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/getting-started.md
https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/configure-acl-plugin.md
