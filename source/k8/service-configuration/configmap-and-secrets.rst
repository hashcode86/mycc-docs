**************************************************
Service Configuration With K8S ConfigMap & Secrets
**************************************************

Configuration trong Sping Boot Service
######################################

Cấu hình (Configuration) cho các service được viết dựa trên Sping Boot thường được đặt trong ``application.yml`` hoặc ``application.properties``. Một ví dụ cho cấu hình của một Sping Boot Service thường thấy:

.. code-block:: yaml

  auth:
    loginUrl: /login
    tokenHeader: Authorization
    tokenPrefix: "Bearer "
    tokenType: jwt
    tokenIssuer: hashcode-dev
    tokenExpirationDuration: 86400
    jwtSecret: hashcode@2020000000000000000000000000000000000000000000000000000

Giả sử các cấu hình trên là cho môi trường ``DEV``, và trong môi trường ``Production`` thì sẽ cần cấu hình lại một số tham số cho phù hợp, ví dụ là ``auth.tokenExpirationDuration`` và ``auth.jwtSecret``:

* **auth.tokenExpirationDuration**: thông tin không nhạy cảm, không cần bảo mật
* **auth.jwtSecret**: thông tin không nhạy cảm, cần bảo mật (giống như cấu hình mật khẩu truy cập database)

Với K8S, ta sẽ sử dụng ConfigMap & Secrets để quản lý tập chung và injecting các cấu hình cho tất cả các service chạy trong K8S Cluster.

Put {Key, Value} Pair to ConfigMap & Secrets
############################################

Trong môi trường Production với các cấu hình không yêu cầu tính bảo mật, ta tạo một ConfigMap Object và đặt các cấu hình trong đó. Ta có thể tạo nhiều ConfigMap Object, mỗi Object cho một Sping Boot Service.

Note: Ta chỉ đưa vào trong ConfigMap hoặc Secrets cặp {Key, Value} mà cần ghi đè lại giá trị đã có trong ``application.yml``

.. code-block:: bash

  kubectl create configmap etc-cfgm-auth-svc \
    --from-literal=auth.tokenIssuer="hashcode-prod" \
    --from-literal=auth.tokenExpirationDuration=3600
  
  #review lại object vừa tạo
  kubectl get configmap etc-cfgm-auth-svc -o yaml

Với cấu hình cần bảo mật, ta tạo một secret Object và đặt cấu hình trong đó tương tự như ConfigMap Object

.. code-block:: bash

  kubectl create secret generic etc-secret \
    --from-literal=auth.jwtSecret="hashcode@2020111111111111111111111111111111111111111111111111111"
  
  #review lại object vừa tạo
  kubectl get secret etc-secret -o yaml


Fabric8 Configuration
#####################

Fabric8 là một development platform, ta sử dụng ở đây với mục đích tự động đóng gói một Sping Boot Service và tự động đưa lên môi trường K8S để chạy.

Enable Fabric8 plugin với Mavel Project (pom.xml)
*************************************************

.. code-block:: xml

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>io.fabric8</groupId>
				<artifactId>fabric8-maven-plugin</artifactId>
				<version>3.5.30</version>
				<executions>
					<execution>
						<id>fmp</id>
						<goals>
							<goal>resource</goal>
							<goal>helm</goal>
							<goal>build</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

Cấu hình định nghĩa App Deployment (src/main/fabric8/deployment.yaml)
*********************************************************************

File cấu hình định nghĩa App Deployment dưới đây cho biết, khi Service được chạy, có 3 biến môi trường:

* auth.tokenIssuer 
* auth.tokenExpirationDuration, 
* auth.jwtSecret

Các biến môi trường trên sẽ được lấy từ K8S Configmap (2) và Secrets (1). Các biến này này sẽ ghi đè các giá trị trong ``application.yml``

.. code-block:: yaml

  spec:
    template:
      spec:
        containers:
          - env:
            - name: auth.tokenIssuer
              valueFrom:
               configMapKeyRef:
                  name: etc-cfgm-auth-svc
                  key: auth.tokenIssuer
            - name: auth.tokenExpirationDuration
              valueFrom:
               configMapKeyRef:
                  name: etc-cfgm-auth-svc
                  key: auth.tokenExpirationDuration
            - name: auth.jwtSecret
              valueFrom:
                secretKeyRef:
                  name: etc-secret
                  key: auth.jwtSecret
  #          - name: SPRING_DATASOURCE_USER
  #            valueFrom:
  #              secretKeyRef:
  #                name: etc-secret
  #                key: database-user
  #          - name: SPRING_DATASOURCE_PASSWORD
  #            valueFrom:
  #              secretKeyRef:
  #                name: etc-secret
  #                key: database-password
  
Đóng gói và đưa service vào K8S Cluster
***************************************

Chạy command dưới để build, đóng gói và upload image vào trong K8S Cluster để chạy dịch vụ

.. code-block:: yaml

  mvnw clean fabric8:deploy