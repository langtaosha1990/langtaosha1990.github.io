---
# title: "通过Django搭建WebSocket聊天服务"
header:
  #   image: /assets/images/bio-photo.jpg
  overlay_image: /assets/images/bio-photo.jpg
# og_image: /assets/images/bio-photo.png
# caption: "Photo credit: [**Unsplash**](https://unsplash.com)"
last_modified_at: 2024-08-25 10:00:00 +0800
---

<!--  -->

### settings.py 中的配置

django 中 settings.py 的配置

```
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'chatapp',	# 添加创建的app
    'channels'	# 添加channels
]

添加
ASGI_APPLICATION = 'ChatProject.asgi.application'

#添加 配置 Django 应用中使用的 Channel Layer， 它提供了一种在多个进程之间传递消息的机制
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
    },
}
```

代码实现：

- ChatProject/asgi.py 代码

  ```
  import os
  from django.core.asgi import get_asgi_application
  from channels.auth import AuthMiddlewareStack
  from channels.routing import ProtocolTypeRouter, URLRouter
  import chatapp.routings


  os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'ChatProject.settings')

  application = ProtocolTypeRouter({
      "http": get_asgi_application(),
      "websocket": AuthMiddlewareStack(
          URLRouter(
              chatapp.routings.websocket_urlpatterns
          )
      ),
  })
  ```

- ChatProject/url.py

  ```
  from django.contrib import admin
  from django.urls import path, include

  urlpatterns = [
      path('admin/', admin.site.urls),
      path('chat/', include('chatapp.urls')),
  ]
  ```

- chatapp/templates/room.html 聊天界面代码

  ```
  <!DOCTYPE html>
  <html lang="en">
  <head>
      <meta charset="UTF-8">
      <title>Chat Room</title>
      <style>
          .chat-log {
              border: 1px solid #ddd;
              height: 300px;
              overflow-y: scroll;
              padding: 10px;
              margin-bottom: 10px;
          }
          .message {
              margin-bottom: 10px;
          }
          .message img,
          .message video,
          .message audio {
              max-width: 100%;
              height: auto;
              display: block;
          }
      </style>
  </head>
  <body>
      <div class="chat-log" id="chat-log"></div>

      <input type="text" id="chat-message-input" placeholder="Enter your message">
      <input type="file" id="chat-media-input" style="display:none;">
      <button id="send-text">Send Text</button>
      <button id="send-image">Send Image</button>
      <button id="send-video">Send Video</button>
      <button id="send-audio">Send Audio</button>

      <script>
          const roomName = "{{ room_name }}";
          const chatSocket = new WebSocket(
              'ws://' + window.location.host + '/ws/chat/' + roomName + '/'
          );

          chatSocket.onmessage = function(e) {
              const data = JSON.parse(e.data);
              const chatLog = document.getElementById('chat-log');
              const messageElement = document.createElement('div');
              messageElement.className = 'message';

              console.log(data);
              if (data.type === 'text') {
                  messageElement.textContent = data.username + ': ' + data.message;
              } else if (data.type === 'image') {
                  const img = document.createElement('img');
                  // 上传到文件存储服务器后返回的地址，这里用本地另一个服务中的文件替代
                  img.src = 'http://127.0.0.1:8000/media/avatars/user_14_image01.jpg';
                  messageElement.textContent = data.username + ': ';
                  messageElement.appendChild(img);
              } else if (data.type === 'video') {
                  const video = document.createElement('video');
                  video.src = data.message;
                  video.controls = true;
                  messageElement.textContent = data.username + ': ';
                  messageElement.appendChild(video);
              } else if (data.type === 'voice') {
                  const audio = document.createElement('audio');
                  audio.src = data.message;
                  audio.controls = true;
                  messageElement.textContent = data.username + ': ';
                  messageElement.appendChild(audio);
              }

              chatLog.appendChild(messageElement);
              chatLog.scrollTop = chatLog.scrollHeight;
          };

          chatSocket.onclose = function(e) {
              console.error('Chat socket closed unexpectedly');
          };

          document.getElementById('send-text').onclick = function(e) {
              const inputDom = document.getElementById('chat-message-input');
              const message = inputDom.value;
              chatSocket.send(JSON.stringify({
                  'type': 'text',
                  'content': message,
                  'user_id': "{{ user.id }}",
                  'user_name':'web'
              }));
              inputDom.value = '';
          };

          function sendMediaMessage(type, file) {
              const reader = new FileReader();
              reader.onload = function(e) {
                  chatSocket.send(JSON.stringify({
                      'type': type,
                      'content': e.target.result,
                       'user_id': "{{ user.id }}"  // 确保这里传递的是实际的用户ID
                  }));
              };
              reader.readAsDataURL(file);
          }

          document.getElementById('send-image').onclick = function(e) {
              document.getElementById('chat-media-input').accept = 'image/*';
              document.getElementById('chat-media-input').click();
          };

          document.getElementById('send-video').onclick = function(e) {
              document.getElementById('chat-media-input').accept = 'video/*';
              document.getElementById('chat-media-input').click();
          };

          document.getElementById('send-audio').onclick = function(e) {
              document.getElementById('chat-media-input').accept = 'audio/*';
              document.getElementById('chat-media-input').click();
          };

          document.getElementById('chat-media-input').onchange = function(e) {
              const file = e.target.files[0];
              if (file) {
                  let type = 'image';
                  if (file.type.startsWith('video/')) {
                      type = 'video';
                  } else if (file.type.startsWith('audio/')) {
                      type = 'voice';
                  }
                  sendMediaMessage(type, file);
              }
              e.target.value = '';  // Clear the input
          };
      </script>
  </body>
  </html>

  ```

- chatapp/consumer.py

  ```
  import json
  from channels.generic.websocket import AsyncWebsocketConsumer

  class ChatConsumer(AsyncWebsocketConsumer):
      async def connect(self):
          self.room_name = self.scope['url_route']['kwargs']['room_name']
          self.room_group_name = f'chat_{self.room_name}'

          # 加入房间组
          await self.channel_layer.group_add(
              self.room_group_name,
              self.channel_name
          )
          await self.accept()

      async def disconnect(self, close_code):
          # 离开房间组
          await self.channel_layer.group_discard(
              self.room_group_name,
              self.channel_name
          )

      # 接收消息并广播到房间组
      async def receive(self, text_data):
          data = json.loads(text_data)
          print(text_data)
          message_type = data.get('type', 'text')
          user_id = data.get('user_id')
          content = data.get('content', '')
          media_url = data.get('media_url', '')
          user_name = data.get('user_name', '')

          await self.channel_layer.group_send(
              self.room_group_name,
              {
                  'type': 'chat_message',
                  'message': content if message_type == 'text' else media_url,
                  'username': user_name,
                  'message_type': message_type
              }
          )

      # 处理来自房间组的消息
      async def chat_message(self, event):
          message = event['message']
          username = event['username']
          message_type = event['message_type']

          # 发送消息到 WebSocket
          await self.send(text_data=json.dumps({
              'message': message,
              'username': username,
              'type': message_type
          }))
  ```

- Chat app/routings.py

  ```
  from django.urls import re_path
  from . import consumers

  websocket_urlpatterns = [
      re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer.as_asgi()),
  ]
  ```

- chatapp

  ```
  from django.urls import path
  from . import views
  urlpatterns = [
      path('<str:room_name>/', views.room, name='room'),
  ]
  ```

文件结构：

![image-20240816110929159](/assets/images/image-20240816110929159.png)

- 通过 8070 端口启动服务

```
python manage.py runserver 8070
```

本地服务运行成功后可以通过http://127.0.0.1:8070/chat/room/进入聊天室

### 如何通过 postman 进行 socket 连接

![image-20240816111637125](/assets/images/image-20240816111637125.png)

![image-20240816111659940](/assets/images/image-20240816111659940.png)

![image-20240816111822297](/assets/images/image-20240816111822297.png)

webview 和 postman 聊天测试：

![image-20240816112023721](/assets/images/image-20240816112023721.png)
