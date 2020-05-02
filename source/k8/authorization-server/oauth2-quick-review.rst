***************
OAuth2 Overview
***************

About OAuth2
############

OAuth2 là một authorization framework cho phép hạn chế được quyền truy cập của từng user tới các HTTP Services. Cơ chế hoạt động đơn giản là user phải xác thực với một authorization server (quản lý các user account) trước khi có thể gửi request tới các Backend Service.

OAuth Roles
###########

OAuth định nghĩa 4 roles sau:

* **Client**
* **Resource Owner** 
* **Resource Server**
* **Authorization Server**

Client: Application
*******************

Client là các Application (ví dụ Mobile Application hoặc Angular Single Page Application):

* Client khi chạy phải gọi các API của các RESTful service (Resource Server). 
* Client gửi gửi kèm các ACCESS_TOKEN trong các request gửi tới Resource Server để Resource Server xác định được Client được phép truy cập.

Ta sẽ có nhiều loại client:

* **Public Clients**: Ví dụ Mobile Application hoặc Angular Single Page Application. Các client này được coi là hoạt động trong môi trường public, không an toàn (client chạy trên trình duyệt hoạt mobile device của user, thông tin dễ bị lộ lọt)
* **Confidential clients**: Ví dụ ứng dụng dựa trên Struts, và Struts Action (chạy trên server) đóng vai trò là client, cần access tới các RESTful service khác của hệ thống. 

Tùy theo loại client mà ta có thể có các FLOW phù hợp cho client đó, để client lấy được ACCESS_TOKEN một cách an toàn và đảm bảo nhất. Với **Public Clients** ta cần một FLOW phức tạp, còn với **Confidential clients** ta có thể cấp cho client một cặp {client_id, client_secret}, client dùng cặp này để xác thực với Authorization Server và nhận về ACCESS_TOKEN.

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-1.png
    :align: center
    :figwidth: 800px

Resource Owner: User
********************

**Resource Owner** thường chính là một người dùng trong hệ thống:

* Trong quá trình User sử dụng Client Application , Trước tiên Client Application sẽ yêu cầu User thực hiện xác thực (Authentication). 
* User thực hiện việc xác thực này với **Authorization Server**. 
* Sau khi xác thực xong ACCESS_TOKEN sẽ được cấp cho Client Application.

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-2.png
    :align: center
    :figwidth: 800px

Authorization Server
********************

**Authorization Server** thực hiện quản lý client application, user, role. Thực hiện xác thực user và sau đó cấp ACCESS_TOKEN. 

Tuy theo việc implement của **Authorization Server** mà ta có những feaute phong phú khác nhau. Trong ETC, ta chọn **Keycloak Authorization Server** (với hướng dẫn cài đặt :doc:`TẠI ĐÂY <keycloak-installation>` ):

* Multi tenant
* Hỗ trợ SSO cho nhiều client application
* Kết nối LDAP, AD là các user directories được dùng phổ biến bởi doanh nghiệp
* Standard Protocols: OpenID Connect, OAuth 2.0 và SAML 2.0
* Hỗ trợ Social Login
* Password Policies
* High Performance, Clustering For scalability and availability
* Opensouce, được phát triển bởi Redhat
* ...

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-3.png
    :align: center
    :figwidth: 800px

Resource Server
***************

Dễ hình dung nhất thì **Resource Server** chính là các RESTful service (backend service). Resource Server thực hiện kiểm tra tính hợp lệ của ACCESS_TOKEN trước khi thực hiện trả về response cho reqeust gửi tới nó. 

Các OAuth2 Flow dùng cho hệ thống ETC
####################################

Chi tiết về các OAuth2 Flows và trường hợp sử dụng của chúng xem `TẠI ĐÂY <https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2/>`_

* Code (Authorization Code)
* Implicit
* Resource Owner Password Credentials
* Client Credentials

Angular SPA Client: Code Flow with PKCE
***************************************

Tại sao Code Flow with PKCE thay vì Implicit Flow? Implement cho Angular Client xem `TẠI ĐÂY <https://ordina-jworks.github.io/security/2019/08/22/Securing-Web-Applications-With-Keycloak.html#implicit-flow-versus-code-flow--pkce/>`_. Các đặc điểm chính:

* Bảo mật
* SSO giữa các Angular SPA (CRM, IM, ...)

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-4.png
    :align: center
    :figwidth: 800px

Mobile App Client: Resource Owner Password Credentials
******************************************************

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-5.png
    :align: center
    :figwidth: 800px

* Các Mobile APP do chính Viettel, ETC làm chủ nên không có vấn đề khi user nhập username, password trực tiếp vào ứng dụng.
* Không cần SSO giữa các Mobile APP nên sử dụng Resource Owner Password Credentials Flow cho đơn giản, thay vì Code Flow with PKCE.

M2M Access: Client Credentials
******************************

* Các Third party system (ví dụ như các hệ thống bên ViettelPay) cần gọi các API của hệ thống ETC sẽ sử dụng cơ chế xác thực qua Client Credentials Flow

.. figure:: /_static/images/k8/authorization-server/oauth2-quick-review-6.png
    :align: center
    :figwidth: 800px
