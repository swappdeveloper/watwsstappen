1. Maak een lege 'ASP,.NET Core Web Application' aan en selecteer de 'API' template.
-------------------------------------------------------------------------------------------------------------------------------------
2. Maak een 'WebSocketManager.cs' aan, en voeg het onderstaande stuk code toe:
-------------------------------------------------------------------------------------------------------------------------------------
```c#
using System;
using System.Collections.Concurrent;
using System.Linq;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

namespace WebsocketWithMiddleware
{
    public class WebSocketManager
    {
        private ConcurrentDictionary<string, WebSocket> _sockets = new ConcurrentDictionary<string, WebSocket>();
        public WebSocket GetSocketById(string id)
        {
            return _sockets.FirstOrDefault(p => p.Key == id).Value;
        }
        public ConcurrentDictionary<string, WebSocket> GetAll()
        {
            return _sockets;
        }
        public string GetId(WebSocket socket)
        {
            return _sockets.FirstOrDefault(p => p.Value == socket).Key;
        }
        public string AddSocket(WebSocket socket)
        {
            var socketId = CreateConnectionId();
            _sockets.TryAdd(socketId, socket);
            return socketId;
        }
        public async Task RemoveSocket(string id)
        {
            WebSocket socket;
            _sockets.TryRemove(id, out socket);
            await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closed by the WebSocketManager", CancellationToken.None);
        }
        public async Task SendMessageToAllAsync(string message)
        {
            foreach (var pair in _sockets)
            {
                if (pair.Value.State == WebSocketState.Open)
                    await SendMessageAsync(pair.Value, message);
            }
        }
        // Private Methods
        private async Task SendMessageAsync(WebSocket socket, string message)
        {
            if (socket.State != WebSocketState.Open)
                return;
            await socket.SendAsync(
                buffer: new ArraySegment<byte>(
                    array: Encoding.ASCII.GetBytes(message),
                    offset: 0,
                    count: message.Length),
                messageType: WebSocketMessageType.Text,
                endOfMessage: true,
                cancellationToken: CancellationToken.None);
        }
        private string CreateConnectionId()
        {
            return Guid.NewGuid().ToString();
        }
    }
}
```
-------------------------------------------------------------------------------------------------------------------------------------
3. Maak een folder aan en geef het volgende naam 'Middelwares'

-------------------------------------------------------------------------------------------------------------------------------------
4. In de 'Middelwares' map maak een nieuwe bestand aan en geeft de volgende naam 'WebSocketMiddleware.cs', vervolgens voeg de onderstaande code toe:
-------------------------------------------------------------------------------------------------------------------------------------
using Microsoft.AspNetCore.Http;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

namespace WebsocketWithMiddleware.Middlewares
{
    public class WebSocketMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly WebSocketManager _webSocketManager;
        public WebSocketMiddleware(RequestDelegate next, WebSocketManager webSocketManager)
        {
            _next = next;
            _webSocketManager = webSocketManager;
        }
        public async Task Invoke(HttpContext context)
        {
            if (!context.WebSockets.IsWebSocketRequest)
            {
                await _next.Invoke(context);
                return;
            }
            var socket = await context.WebSockets.AcceptWebSocketAsync();
            var id = _webSocketManager.AddSocket(socket);
            await ReceiveAsync(socket, async (result, buffer) =>
            {
                if (result.MessageType == WebSocketMessageType.Close)
                {
                    await _webSocketManager.RemoveSocket(id);
                    return;
                }
            });
        }
        private async Task ReceiveAsync(WebSocket socket, Action<WebSocketReceiveResult, byte[]> handleMessage)
        {
            var buffer = new byte[1024 * 4];
            while (socket.State == WebSocketState.Open)
            {
                var result = await socket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
                var message = $"{ _webSocketManager.GetId(socket) } said: {Encoding.UTF8.GetString(buffer, 0, result.Count)}";
                await _webSocketManager.SendMessageToAllAsync(message);
                handleMessage(result, buffer);
            }
        }
    }
}

-------------------------------------------------------------------------------------------------------------------------------------

5. Open de 'Startup.cs' bestand en voeg het volgende toe voor 'app.UseMvc()' in ConfigureServices method. Wel de juiste using gebruiken.
-------------------------------------------------------------------------------------------------------------------------------------
services.AddSingleton<WebSocketManager>();
  
-------------------------------------------------------------------------------------------------------------------------------------

6. Open de 'Startup.cs' bestand en voeg het volgende toe voor 'app.UseMvc()' in Configure method. Wel de juiste using gebruiken.
-------------------------------------------------------------------------------------------------------------------------------------
app.UseStaticFiles();
app.UseWebSockets();
app.Map("/websocket", a =>
{
    a.UseMiddleware<WebSocketMiddleware>();
});

-------------------------------------------------------------------------------------------------------------------------------------

7. Maak een 'wwwroot' map aan.
-------------------------------------------------------------------------------------------------------------------------------------

8. In de 'wwwroot' map, maak een index.html bestand aan en voeg de volgende stuk code toe:

-------------------------------------------------------------------------------------------------------------------------------------

<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8" />
    <title>Websocket example with middelware</title>
</head>

<body>
    <h1>This should be mapped to "/websocket"</h1>
    <input type=text id="textInput" placeholder="Enter your text" />
    <button id="sendButton">Send</button>
    <ul id="messages"></ul>
    <script language="javascript" type="text/javascript">
        var uri = "ws://" + window.location.host + "/websocket";
        function connect() {
            socket = new WebSocket(uri);
            socket.onopen = function(event) {
                console.log("opened connection to " + uri);
            };
            socket.onclose = function(event) {
                console.log("closed connection from " + uri);
            };
            socket.onmessage = function (event) {
                appendItem(list, event.data);
                console.log(event.data);
            };
            socket.onerror = function(event) {
                console.log("error: " + event.data);
            };
        }
        connect();
        var list = document.getElementById("messages");
        var button = document.getElementById("sendButton");
        button.addEventListener("click", function() {
            var input = document.getElementById("textInput");
            sendMessage(input.value);
            input.value = "";
        });
        function sendMessage(message) {
            console.log("Sending: " + message);
            socket.send(message);
        }
        function appendItem(list, message) {
            var item = document.createElement("li");
            item.appendChild(document.createTextNode(message));
            list.appendChild(item);
        }
    </script>
</body>
</html>
