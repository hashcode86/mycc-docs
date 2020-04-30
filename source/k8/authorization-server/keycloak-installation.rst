*********************
KEYCLOAK INSTALLATION
*********************

Database for Keycloak
#####################

DB cho Keycloak có thể sử dụng: Oracle, Mariadb, Mysql, H2, etc. Trong ví dụ này ta sử dụng một Mariadb với các thông tin:

* IP_ADDR : 192.168.8.188
* DB_NAME:  keycloak
* USERNAME: keycloak
* PASSWORD: keycloak

Cấu hình K8 Service Object trỏ tới external endpoints là một Mariadb. Mục đích để các ứng dụng chạy trong K8cluster trỏ tới db này qua service name: ``keycloak-db``:

.. code-block:: bash

  echo "
  kind: Service
  apiVersion: v1
  metadata:
   name: keycloak-db
  spec:
   ports:
   - port: 3306
     targetPort: 3306
  ---
  kind: Endpoints
  apiVersion: v1
  metadata:
   name: keycloak-db
  subsets:
   - addresses:
       - ip: 192.168.8.188
     ports:
       - port: 3306
  " | kubectl apply -f -
  
  
  #view lại các service và endpoints
  kubectl get service
  kubectl get Endpoints

Đưa DB {username, password} lưu trong K8 Secret:

.. code-block:: bash

  kubectl create secret generic keycloak-secret \
    --from-literal=db.user="keycloak" \
    --from-literal=db.password="keycloak"
  
  #view lại các secret
  kubectl get secret

Cấu hình thông tin DB
#####################

.. code-block:: bash

  cd; mkdir keycloak; cd keycloak
  wget https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/9.0.3/kubernetes-examples/keycloak.yaml
  
          #edit keycloak.yaml bổ sung thêm các thông tin cấu hình trong ``env``
  
          - name: DB_VENDOR
            value: "mariadb"
          - name: DB_ADDR
            value: "keycloak-db"
          - name: DB_USER
            valueFrom:
              secretKeyRef:
                name: keycloak-secret
                key: db.user
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: keycloak-secret
                key: db.password
          - name: KEYCLOAK_HOSTNAME
            value: "keycloak.127.0.0.1.nip.io"

Run your owner Keycloak
#######################

.. code-block:: bash

  cd ~/keycloak
  kubectl create -f keycloak.yaml
  
  #To view deployment log:
  kubectl logs deployment.apps/keycloak

  #to delete keycloak app:
  #kubectl delete service/keycloak deployment.apps/keycloak

* Tới đây ta có thể truy cập keycloak qua địa chỉ: **http://keycloak.127.0.0.1.nip.io:8080/auth/**
* Thiết lập ban đầu cho Keycloak :doc:`VISIT THIS <keycloak-initialization>`