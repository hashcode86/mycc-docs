*************
Tích hợp myCC
*************

.. meta::
   :description lang=en: Get started writing technical documentation with Sphinx and publishing to Read the Docs.

.. _extensions: http://www.sphinx-doc.org/en/master/ext/builtins.html#builtin-sphinx-extensions


Nhúng myCC WebPhone trực tiếp vào CRM
#####################################

Để loại bỏ sự phiền toái trong việc phải chuyển đổi qua lại giữa các màn hình (myCC, CRM, ...). Doanh nghiệp có thể nhúng trực tiếp myCC WebPhone vào CRM của mình. 
Việc này đem đến khả năng như:

* Điều khiển cuộc gọi (như: Trả lời/ kết thúc/ chuyển/ hold/ mute)
* Xem thông tin chi tiết cuộc gọi hiện tại ( Tên người gọi, số điện thoại, thời gian gọi)
* Bấm số để gọi ra trên myCC WebPhone
* Agent đặt trạng thái của mình
* ...

myCC WebPhone cần được đặt trong một iframe và nên được đặt ở ``phía trên bên phải`` màn hình CRM. Dưới đây là hình chụp một ví dụ: 

.. figure:: ../_static/images/integrate/webphone-integrate-1.png
    :align: center
    :figwidth: 300px
    :target: ../_static/images/integrate/webphone-integrate-1.png

WebPhone nên được đặt trong một Button và ở ``phía trên bên phải`` của màn hình CRM. Các tính năng với Button này cần phía implement tại CRM:

* Show/ Hide WebPhone bên trong khi click vào Button.
* Show/ Hide WebPhone thông qua API 
* CRM đăng ký nhận sự kiện do WebPhone (IFRAME) gửi ra khi có các sự kiện liên quan tới cuộc gọi (New Incomming Call, Call Answer, Call End, ...)
* Với sự kiện, ví dụ như sự kiện có cuộc gọi mới đang RING, sau khi CRM nhận sự kiện này từ WebPhone, cần gọi API ở trên để Show WebPhone cho EndUser;
  Đồng thời có thể tự động bật thông tin khách hàng (theo số điện thoại) cho EndUser.
* Gửi sự kiện yêu cầu WebPhone gọi ra cho một số điện thoại (click to call trên CRM)
* Tự động đăng nhập WebPhone khi user đăng nhập CRM


Nhúng WebPhone vào CRM
**********************

myCC WebPhone: https://mycc.vn/webphone

Mã để đặt WebPhone vào một iframe, ví dụ:

.. code-block:: html

	<iframe id="iframeWebPhone" src="https://mycc.vn/webphone" allow="microphone; camera"
			style="height: 100%; width: 100%" frameborder="0" scrolling="auto"></iframe>

Chú ý cần cấp quyền ``microphone; camera`` cho iframe này.

Nhận sự kiện gửi từ WebPhone
****************************

Khi WebPhone được load, bạn cần đăng ký một hàm để nhận và xử lý sự kiện do WebPhone gửi ra. 

.. code-block:: javascript

	var handleWebPhoneMessage = function(event) {
		var msg = event.data;
		console.log(msg);
	};
	
	window.onload = (event) => {
	  console.log('page is fully loaded');
	  window.addEventListener("message", handleWebPhoneMessage, false);
	};

Câu lệnh ``console.log(msg);`` trên sẽ in ra sự kiện do WebPhone gửi. Cấu trúc dữ liệu của biến msg này sẽ gồm 2 trường:

.. code-block:: json

	{
		type: Cho biết sự kiện đang được gửi là gì. Ví dụ: OnCallRinging, OnCallAnswer
		data: Dữ liệu chi tiết của sự kiện
	}

Các sự kiện WebPhone sẽ gửi ra tưng ứng với các trường hợp:

+------------------+------------------------------------------------------------------------------------------------------------------------------+
| Type             | Ý nghĩa                                                                                                                      |
+==================+==============================================================================================================================+
| OnRegistered     | WebPhone đã sẵn sàng (nhận hoặc gọi đi)                                                                                      |
+------------------+------------------------------------------------------------------------------------------------------------------------------+
| OnCallRinging    | Có cuộc gọi đến đang RING hoặc                                                                                               |
|                  | Cuộc gọi đi cho khách hàng đang được đổ chuông                                                                               |
+------------------+------------------------------------------------------------------------------------------------------------------------------+
| OnCallAnswer     | Có cuộc gọi đến được EndUser tiếp nhận hoặc                                                                                  |
|                  | Cuộc gọi đi được số đích bấm trả lời                                                                                         |
+------------------+------------------------------------------------------------------------------------------------------------------------------+
	
Sự kiện cho cuộc gọi vào hàng đợi và tới WebPhone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Cuộc gọi vào hàng đợi (QUEUE) và đang RING tới Agent
----------------------------------------------------

.. code-block:: json

	{
		"type":"OnCallRinging",
		"data":{
			"interactionID":"63729795363-220b673c",
			"callId":"26329cfa6c6c7803aabe294bcf9d2c41-b57f2aaf538cef33c05413c97c90ea8e-afc28ad5",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
			"callerIdNumber":"0900000101",
			"direction":"outbound",
			"request":"thangdd8@acc01.mycc.vn",
			"calleeIdNumber":"+84869517833",
			"resourceType":null,
			"queueAgentInteraction":true,
			"state":{
				"@c":".QueueAgentConnecting",
				"timestamp":1562576163000,
				"agentId":"b57f2aaf538cef33c05413c97c90ea8e",
				"queueId":"d57d2bc2a5acc43a7dfd80d9a5c36b10",
				"memberCallId":"c772f9ca-1c00-1238-a9bd-0050560123b5"
			}
		}
	}

Cuộc gọi vào hàng đợi (QUEUE) và được Agent tiếp nhận
-----------------------------------------------------

.. code-block:: json

	{
		"type":"OnCallAnswer",
		"data":{
			"interactionID":"63729795808-3778afd6",
			"callId":"d648e037a096707c5fd09e18d823d652-b57f2aaf538cef33c05413c97c90ea8e-bc2dab04",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
			"callerIdNumber":"0900000101",
			"direction":"outbound",
			"request":null,
			"calleeIdNumber":"+84869517833",
			"resourceType":null,
			"queueAgentInteraction":true,
			"state":{
				"@c":".QueueAgentConnected",
				"timestamp":1562576619000,
				"agentId":"b57f2aaf538cef33c05413c97c90ea8e",
				"queueId":"d57d2bc2a5acc43a7dfd80d9a5c36b10",
				"memberCallId":"d15e2fac-1c01-1238-a9bd-0050560123b5"
			}
		}
	}
	
Trong đó:

* data.callId
	id cuộc gọi đang ring tới Agent
	
* data.state.memberCallId
	id cuộc gọi của khách hàng

* data.ownerId
	userId của tài khoản đang đăng nhập WebPhone và trả lời cuộc gọi

* data.callerIdNumber
	Số điện thoại người gọi
	
* data.state.queueId
	Id hàng đợi mà cuộc gọi đã vào đó
	
Sự kiện cho cuộc gọi ring thằng WebPhone (không qua QUEUE)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Cuộc gọi đang RING
------------------

.. code-block:: json

	{
		"type":"OnCallRinging",
		"data":{
			"interactionID":"63729796945-eb43c3a4",
			"callId":"79e78adb-1c04-1238-a9bd-0050560123b5",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":null,
			"callerIdNumber":"0900000101",
			"direction":"inbound",
			"request":"0869517833@171.244.49.129",
			"calleeIdNumber":null,
			"resourceType":"offnet-origination",
			"queueAgentInteraction":false,
			"state":{
				"@c":".CreateOtherLeg",
				"timestamp":1562577750000,
				"callId":"e9000d84-a161-11e9-a7f0-6fe8469ff495",
				"request":"thangdd8@acc01.mycc.vn",
				"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
				"direction":"outbound",
				"resourceType":null
			}
		}
	}

Cuộc gọi được trả lời
---------------------

.. code-block:: json

	{
		"type":"OnCallAnswer",
		"data":{
			"interactionID":"63729797116-ba83f7c9",
			"callId":"e04e4b96-1c04-1238-a9bd-0050560123b5",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":null,
			"callerIdNumber":"0900000101",
			"direction":"inbound",
			"request":null,
			"calleeIdNumber":"+84869517833",
			"resourceType":"offnet-origination",
			"queueAgentInteraction":false,
			"state":{
				"@c":".AnsweringWithOtherLeg",
				"timestamp":1562577929000,
				"callId":"4f452bec-a162-11e9-a8eb-6fe8469ff495",
				"request":"thangdd8@acc01.mycc.vn",
				"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
				"direction":"outbound",
				"resourceType":null
			}
		}
	}
	
Trong đó:

* data.callId
	id cuộc gọi của khách hàng
	
* data.state.callId
	id cuộc gọi ring tới WebPhone

* data.state.ownerId
	userId của tài khoản đang đăng nhập WebPhone và trả lời cuộc gọi

* data.callerIdNumber
	Số điện thoại người gọi
	
* data.calleeIdNumber
	Số khách hàng đã gọi
	
Sự kiện cho cuộc gọi khi user bấm gọi ra cho số khách hàng
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Cuộc gọi đang RING
------------------

.. code-block:: json

	{
		"type":"OnCallRinging",
		"data":{
			"interactionID":"63729797336-930be5fa",
			"callId":"re2113maho8s8n7rqos7",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
			"callerIdNumber":"thangdd8",
			"direction":"inbound",
			"request":"0982861121@acc01.mycc.vn",
			"calleeIdNumber":null,
			"resourceType":null,
			"queueAgentInteraction":false,
			"state":{
				"@c":".CreateOtherLeg",
				"timestamp":1562578136000,
				"callId":"cf5b0a7c-a162-11e9-aa99-6fe8469ff495",
				"request":"0982861121@acc01.mycc.vn",
				"ownerId":null,
				"direction":"outbound",
				"resourceType":"offnet-termination"
			}
		}
	}


Cuộc gọi được trả lời
---------------------

.. code-block:: json

	{
		"type":"OnCallAnswer",
		"data":{
			"interactionID":"63729797336-930be5fa",
			"callId":"re2113maho8s8n7rqos7",
			"accountId":"77f4d2fa62cf5f9a97c620aa86438952",
			"ownerId":"b57f2aaf538cef33c05413c97c90ea8e",
			"callerIdNumber":"100",
			"direction":"inbound",
			"request":"0982861121@acc01.mycc.vn",
			"calleeIdNumber":"982861121",
			"resourceType":null,
			"queueAgentInteraction":false,
			"state":{
				"@c":".AnsweringWithOtherLeg",
				"timestamp":1562578145000,
				"callId":"cf5b0a7c-a162-11e9-aa99-6fe8469ff495",
				"request":"0982861121@acc01.mycc.vn",
				"ownerId":null,
				"direction":"outbound",
				"resourceType":"offnet-termination"
			}
		}
	}

Trong đó:

* data.callId
	id cuộc gọi WebPhone đang gọi đi cho KH
	
* data.state.callId
	id cuộc gọi ring tới máy KH

* data.ownerId
	userId của tài khoản đang đăng nhập WebPhone và bấm gọi đi

* data.callerIdNumber
	Số điện thoại người gọi
	
* data.calleeIdNumber
	Số khách hàng
	
Gửi sự kiện gửi tới WebPhone
****************************

Trường hợp bạn cần click vào một số điện thoại trên CRM và mong muốn WebPhone thực hiện cuộc gọi ra cho số điện thoại đó, khi đó bạn sẽ cần gửi sự kiện tới WebPhone. Dưới đây là một ví dụ:

.. code-block:: javascript

	var makeCall = function(target) {
		var iframe = document.getElementById('iframeWebPhone');
		
		var outCallRequest = {'type': 'OutboundCallRequest', data: {'target': target}};
		iframe.contentWindow.postMessage(outCallRequest, '*');
	}
	
FULL EXAMPLE CODE
#################

.. code-block:: html

	<!DOCTYPE html>
	<html>
		<head>
		<title>myCC WebPhone Example</title>
		</head>
		<body>

			<button onclick="makeCall('0982861121')">CALL TO 0982861121</button>

			<iframe id="iframeWebPhone" src="http://localhost:9999" allow="microphone; camera"
							style="height: 600px; width: 50%" frameborder="0" scrolling="auto"></iframe>

			<script>
				var handleWebPhoneMessage = function(event) {
					var msg = event.data;
					console.log(''+JSON.stringify(msg));
				};

				window.onload = (event) => {
				  console.log('page is fully loaded');
				  window.addEventListener("message", handleWebPhoneMessage, false);
				};

				var makeCall = function(target) {
					var iframe = document.getElementById('iframeWebPhone');
					
					var outCallRequest = {'type': 'OutboundCallRequest', data: {'target': target}};
					iframe.contentWindow.postMessage(outCallRequest, '*');
				}
			</script>
							
		</body>
	</html>

