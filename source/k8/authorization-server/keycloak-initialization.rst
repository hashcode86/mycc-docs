***********************
KEYCLOAK INITIALIZATION
***********************

Khai báo Realm
##############

Khái niệm Realm trong keycloak tương đương với khái niệm tenant trong các dịch vụ cloud khác. Khi cần phải setup Authorization Service cho một tổ chức, ta khai báo mới một keycloak realm mới.

Giả sử với ETC, ta tổ chức 2 realm:
* etc-internal: Định danh và quản lý truy cập cho các người dùng nội bộ cho hệ thống ETC (người dùng thuộc công ty ETC và các BOT)
* etc-external: Định danh và quản lý truy cập cho các người dùng là khách hàng cá nhân, khách hàng doanh nghiệp của công ty ETC (dùng truy cập vào các hệ thống portal, mobile app)

Trong nội dung này, ta focus vào khai báo ví vụ một realm là ``etc-internal`` với mục đích hướng dẫn về keycloak

Khai báo một realm mới (etc-internal) ta setup các nội dung sau:

* Realm name: etc-internal
* Default Signature Algorithm: RS256 (xem thêm về thực hiện jwt authentication tại API gateway)

.. figure:: /_static/images/k8/authorization-server/keycloak-initial-1.png
    :align: center
    :figwidth: 800px

Lấy public key  (Dùng khi thực hiện jwt authentication tại API gateway)

.. figure:: /_static/images/k8/authorization-server/keycloak-initial-2.png
    :align: center
    :figwidth: 800px

Khai báo các clients của etc-internal
#####################################

Client trong keycloak đại diện cho một resource mà user có thể truy cập. Client (resource) này sẽ yêu cầu user phải thực hiện authenticate với keycloak. Dễ hiểu hơn, clients có thể là:

* Một Application - Khi User truy cập Application thì Application này ủy quyền cho keycloak xác thực User --> Application đóng vai trò là một cient của keycloak
* Một Webservice - Một Webservice muốn cung cấp cơ chế authentication cho các client của nó (webservice khác hoặc ứng dụng khác gọi tới nó). Client của webservice sẽ được cấp cặp key+secret, nó sẽ dùng để authenticate với keycloak, xin cấp access token để có thể gọi tới Webservice.

Trong ETC, các client có thể có là:

* CRM Web Application
* IM Web Application
* Mobile Application
* ...

Cụ thể hơn, với phân hệ CRM, ta sẽ khai báo một client gồm các thông tin như ảnh dưới:

* Trong đó ``Root URL`` là đường dẫn truy cập của ứng dụng CRM. Ở đây ta ví dụ đường dẫn CRM là: ``https://www.keycloak.org/app/``

.. figure:: /_static/images/k8/authorization-server/keycloak-initial-3.png
    :align: center
    :figwidth: 800px


Sau khi tạo xong client, ta có thể view lại chi tiết hơn về cài đặt mặc định ban đầu client application này như ảnh dưới:

* Standard Flow Enabled: ON - Hỗ trợ ``Authorization Code Flow``
* Direct Access Grants Enabled: ON - Hỗ trợ ``Resource Owner Password Credentials Grant Flow``
* Xem them chi tiết về các Flow trong Oauth2: https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2

.. figure:: /_static/images/k8/authorization-server/keycloak-initial-4.png
    :align: center
    :figwidth: 800px

Tại đây, sau khi kết thúc việc tạo client trên keycloak, tại phía ứng dụng CRM, khi user truy cập, nếu xác định user chưa login, CRM cần redirect trình duyệt của user qua trang đăng nhập của keycloak:

.. code-block:: bash

	http://localhost:8080/auth/realms/etc-internal/protocol/openid-connect/auth?client_id=crm&redirect_uri={CRM_CALLBACK_URL}&response_type=code&scope=openid

Các nội dung cần chú ý:

* ``...realms/etc-internal/...``: authenticate cho client của realm etc-internal
* ``client_id=crm``: authenticate cho client ``crm``
* ``redirect_uri={CRM_CALLBACK_URL}``: sau khi login thành công, chuyển người dùng về lại trang này
* ``response_type=code``: sau khi login thành công client sẽ nhận được một Authorization Code. Client dùng code này để lấy về JWT Access Token (xem thêm OAuth2 Authorization Code Flow)

Tuy nhiên tại đây etc-internal vẫn chưa có bất cứ user nào được khai báo nên chưa thể đăng nhập được.

Ta có thể kết nối keycloak tới một LDAP server để thực hiện xác thực user.

Khai báo thủ công các users của etc-internal
############################################

Như đã nói ở trên, Ta có thể kết nối keycloak tới một LDAP server để thực hiện xác thực user. Tuy nhiên trong case này, ta sẽ khai báo thủ công một user trên keycloak thông qua chức năng quản lý users.

.. figure:: /_static/images/k8/authorization-server/keycloak-initial-5.png
    :align: center
    :figwidth: 800px

Sau khi khai báo & đặt mật khẩu cho user, ta sử dụng user này để truy cập ứng dụng CRM. theo luồng Authorization Code Flow cuối cùng ta sẽ lấy được JWT Access Token như dưới:

.. code-block:: javascript

	{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJJZEtidUFnT3dQc2ZNVFFKdlBRRk9IT05UWEo3UDI1NU55MG5ueVh3c2ZVIn0.eyJleHAiOjE1ODgwOTM1NTQsImlhdCI6MTU4ODA5MzI1NCwiYXV0aF90aW1lIjoxNTg4MDkzMjUyLCJqdGkiOiI3ZWMzZDU2NC1iMDg1LTQ3YzMtODRkOC1hZGFmNWE0ZjkyMmYiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvYXV0aC9yZWFsbXMvZXRjLWludGVybmFsIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjU5MWJlZTM1LTVjMDMtNDg1Ni05YThhLWFkZTIxYmNhNDJmYSIsInR5cCI6IkJlYXJlciIsImF6cCI6ImNybSIsIm5vbmNlIjoiYjFkNjdkZTEtNzE3YS00OWU2LTk1YzAtMzAzOThmZmRiOTY3Iiwic2Vzc2lvbl9zdGF0ZSI6IjdjNzZlZDQ0LTQxMWEtNDljOC05NDZlLWZiNTc5OTZiZDEyMiIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiaHR0cHM6Ly93d3cua2V5Y2xvYWsub3JnIl0sInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsiYWNjb3VudCI6eyJyb2xlcyI6WyJtYW5hZ2UtYWNjb3VudCIsIm1hbmFnZS1hY2NvdW50LWxpbmtzIiwidmlldy1wcm9maWxlIl19fSwic2NvcGUiOiJvcGVuaWQgcHJvZmlsZSBlbWFpbCIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwibmFtZSI6IlRoYW5nIERvIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiaGFzaGNvZGUiLCJnaXZlbl9uYW1lIjoiVGhhbmciLCJmYW1pbHlfbmFtZSI6IkRvIn0.KGhw6c4T2UlUXmKxD_-vJAANpzNDuQwfju0GcxfWt3BoGDbfvx0s0vy6Toygbb2T1AqRSr68bM9FpTcIzdJE1zVYoq5xoZXZObNC6NLo8Am2WGTcBgT-kPNDGgOQ21mQnV63nhrLwt32adPB-nEL7Q_tcSml_x1Lh5V8zD4kENQNBywahE25dp1N4ktr7u_kp_uEFiX7ZbNXxh3cLG1KukavqSfSs59H6lqE7OWZCKfnLBWMf08m-2MUyLIoOOwEVPiuPSYub8nc6FRvgnTaacL4sGZnfYFzI83raESkSb2PfxeWXTcdtk5K6qrK1d5QAZ_nJMQjRhaQ0TndAgbbiw","expires_in":300,"refresh_expires_in":1800,"refresh_token":"eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI1YWVjNGM0Mi04ZmE5LTQ3NDMtYTFhNi00MDA4NjBmMmE4ZTcifQ.eyJleHAiOjE1ODgwOTUwNTQsImlhdCI6MTU4ODA5MzI1NCwianRpIjoiYzdiZjc3YzYtMzExMC00YzBjLWE0MTMtOTM4ZTcyMTgzZDVlIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwL2F1dGgvcmVhbG1zL2V0Yy1pbnRlcm5hbCIsImF1ZCI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9hdXRoL3JlYWxtcy9ldGMtaW50ZXJuYWwiLCJzdWIiOiI1OTFiZWUzNS01YzAzLTQ4NTYtOWE4YS1hZGUyMWJjYTQyZmEiLCJ0eXAiOiJSZWZyZXNoIiwiYXpwIjoiY3JtIiwibm9uY2UiOiJiMWQ2N2RlMS03MTdhLTQ5ZTYtOTVjMC0zMDM5OGZmZGI5NjciLCJzZXNzaW9uX3N0YXRlIjoiN2M3NmVkNDQtNDExYS00OWM4LTk0NmUtZmI1Nzk5NmJkMTIyIiwic2NvcGUiOiJvcGVuaWQgcHJvZmlsZSBlbWFpbCJ9.pNCub9QnSAOlTwdfO0nbpxo2kmb-OVCF2ocQOHSSpSI","token_type":"bearer","id_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJJZEtidUFnT3dQc2ZNVFFKdlBRRk9IT05UWEo3UDI1NU55MG5ueVh3c2ZVIn0.eyJleHAiOjE1ODgwOTM1NTQsImlhdCI6MTU4ODA5MzI1NCwiYXV0aF90aW1lIjoxNTg4MDkzMjUyLCJqdGkiOiJkZGZhZjAzOC1mNzgyLTQ2NzQtYTcxYi00YWIyYmFkYzEzYjUiLCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAvYXV0aC9yZWFsbXMvZXRjLWludGVybmFsIiwiYXVkIjoiY3JtIiwic3ViIjoiNTkxYmVlMzUtNWMwMy00ODU2LTlhOGEtYWRlMjFiY2E0MmZhIiwidHlwIjoiSUQiLCJhenAiOiJjcm0iLCJub25jZSI6ImIxZDY3ZGUxLTcxN2EtNDllNi05NWMwLTMwMzk4ZmZkYjk2NyIsInNlc3Npb25fc3RhdGUiOiI3Yzc2ZWQ0NC00MTFhLTQ5YzgtOTQ2ZS1mYjU3OTk2YmQxMjIiLCJhY3IiOiIxIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJuYW1lIjoiVGhhbmcgRG8iLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJoYXNoY29kZSIsImdpdmVuX25hbWUiOiJUaGFuZyIsImZhbWlseV9uYW1lIjoiRG8ifQ.GNjTYcvQF_JHZTKLZqILUMc17lgORLA_vU4q1qYO67f2GX0YKse3cEquIboQ1vHSikT841qa2y76px4wndi2i0GyoJPdqZxnmupwy6TOonB9aVMaIWu7lADcVUn-VSivudIPEryzdlIM8R7UEXoMbq7Z0JyahTb-0TZsxKYOCKXkWlY40u9_ZKIo58cMLttXOoxBbRn-o6pBbpMHZW7eRz6_oME5TgHSed4EIrQcMoUKqklhrSDOebinrKXkpS9m_aoYVNzHFBAPK9fMqFq3S3whr5zRWmMUbtLOLpqWFsNyMyowsRDCmfQ59bZSTympnRfR05eSwRbDpqiMbbUOOg","not-before-policy":0,"session_state":"7c76ed44-411a-49c8-946e-fb57996bd122","scope":"openid profile email"}

Khai báo client IM, thực hiện SSO giữa CMR & IM
###############################################