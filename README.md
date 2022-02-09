# python-sockets-chat-and-file-send
Simple CLI chat app running on LAN. It can send messages and files, also discover other clients.


* To run: `python3 chat_pro.py`
* You can use it on computers on same LAN. 
* Clients will automatically discover each other.
* TCP and UDP sockets are used to send messages.
* File sending is achieved with flow control over UDP.
* Users have to send a normal chat message before sending a file. So please say "hi" to the target user first :)
