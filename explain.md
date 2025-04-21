# DNS Lookup Script Explanation

This document provides a detailed explanation of the DNS lookup script that uses the UDP protocol. The script is broken down into functional components with code snippets and explanations.

## 1. Imports and Setup

```python
import socket
import struct
import random
import argparse
```

- `socket` - Provides low-level networking interface to create UDP sockets
- `struct` - Used for packing and unpacking binary data according to DNS protocol format
- `random` - Generates random transaction IDs for DNS queries
- `argparse` - Parses command-line arguments for the script

## 2. DNS Query Construction

```python
class DNSQuery:
    def __init__(self, domain):
        self.domain = domain
        self.transaction_id = random.randint(0, 65535)  # Random transaction ID
    
    def build_query(self):
        """Build a DNS query packet."""
        # Header section
        header = struct.pack(
            "!HHHHHH",
            self.transaction_id,  # Transaction ID
            0x0100,  # Flags: standard query (0x0100)
            1,       # Questions: 1
            0,       # Answer RRs: 0
            0,       # Authority RRs: 0
            0        # Additional RRs: 0
        )
```

- The `DNSQuery` class is responsible for constructing binary DNS query packets
- Each query gets a random transaction ID (16-bit integer) to match responses to queries
- The `build_query` method starts by creating the DNS header using `struct.pack`
  - `!` - Network byte order (big-endian)
  - `H` - Unsigned short (2 bytes) for each header field
  - `0x0100` represents standard query flags (RD bit set)
  - We specify 1 question and 0 records for answers, authority, and additional sections

```python
        # Question section - domain name
        query_parts = []
        for part in self.domain.split('.'):
            query_parts.append(bytes([len(part)]))
            query_parts.append(part.encode('utf-8'))
        query_parts.append(b'\x00')  # Terminating null byte
        
        # Question section - type and class
        query_parts.append(struct.pack('!HH', 1, 1))  # Type: A (1), Class: IN (1)
        
        return header + b''.join(query_parts)
```

- DNS protocol requires domain names to be encoded as a sequence of labels
- Each label is prefixed with its length (e.g., "example.com" becomes `\x07example\x03com\x00`)
- The domain name is terminated with a null byte (`\x00`)
- The question section ends with the query type (1 for A record) and class (1 for Internet)
- The complete packet is returned as a binary string

## 3. DNS Response Parsing

```python
class DNSResponse:
    def __init__(self, data):
        self.data = data
        self.answers = []
        self.parse_response()
    
    def parse_response(self):
        """Parse the DNS response packet."""
        # Parse header
        header = struct.unpack('!HHHHHH', self.data[:12])
        transaction_id, flags, questions, answers, authority, additional = header
        
        # Check if response is valid
        if flags & 0x8000 == 0:  # Check QR bit
            raise Exception("Not a valid DNS response")
        
        if flags & 0x000F != 0:  # Check RCODE
            error_codes = {
                1: "Format Error",
                2: "Server Failure",
                3: "Name Error (Domain does not exist)",
                4: "Not Implemented",
                5: "Refused"
            }
            rcode = flags & 0x000F
            raise Exception(f"DNS Error: {error_codes.get(rcode, f'Unknown Error ({rcode})')}")
```

- The `DNSResponse` class handles parsing of binary DNS response packets
- `struct.unpack` extracts the 6 header fields from the first 12 bytes of the response
- The code checks if this is a valid DNS response by checking the QR bit (bit 15 in the flags)
- If the response contains an error code (RCODE), an exception is raised with a descriptive message

```python
        # Skip the question section
        offset = 12  # Header size
        
        # Skip the domain name in the question
        while True:
            length = self.data[offset]
            offset += 1
            if length == 0:
                break
            offset += length
        
        # Skip QTYPE and QCLASS (4 bytes total)
        offset += 4
```

- After processing the header, we need to skip over the question section
- We start at byte 12 (after the header) and follow the domain name format
- Each label is prefixed with its length, and we continue until we hit a zero length
- We then skip 4 more bytes (2 for query type, 2 for query class)

```python
        # Parse answers
        for _ in range(answers):
            # Domain name can be a pointer or a sequence of labels
            if (self.data[offset] & 0xC0) == 0xC0:  # Compressed domain name pointer
                offset += 2  # Skip pointer
            else:
                # Skip the domain name labels
                while True:
                    length = self.data[offset]
                    offset += 1
                    if length == 0:
                        break
                    offset += length
            
            # Extract record type, class, TTL, and data length
            rec_type, rec_class, ttl, data_len = struct.unpack('!HHIH', self.data[offset:offset+10])
            offset += 10
```

- We process each answer record based on the count in the header
- DNS uses name compression - if the first two bits of a byte are set (0xC0), it's a pointer
- For each answer, we extract:
  - Record type (A=1, AAAA=28, etc.)
  - Record class (IN=1)
  - TTL (Time-to-Live in seconds)
  - Data length (how many bytes of data follow)

```python
            # Process record data based on type
            if rec_type == 1:  # A record (IPv4)
                ip = socket.inet_ntoa(self.data[offset:offset+data_len])
                self.answers.append(('A', ip, ttl))
            elif rec_type == 5:  # CNAME record
                # This is simplified - proper CNAME parsing would require handling compression
                cname = "CNAME data"  # Placeholder
                self.answers.append(('CNAME', cname, ttl))
            elif rec_type == 28:  # AAAA record (IPv6)
                ip = socket.inet_ntop(socket.AF_INET6, self.data[offset:offset+data_len])
                self.answers.append(('AAAA', ip, ttl))
            
            offset += data_len
```

- For A records (IPv4): We convert 4 bytes to a dotted-decimal string using `inet_ntoa`
- For AAAA records (IPv6): We convert 16 bytes to IPv6 notation using `inet_ntop`
- The script has basic support for CNAME records but doesn't fully parse them
- Results are stored in the `answers` list as tuples of (record_type, data, ttl)

## 4. Network Communication

```python
def dns_lookup(domain, dns_server="8.8.8.8", port=53, timeout=5):
    """Perform a DNS lookup using raw UDP sockets."""
    # Create UDP socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(timeout)
    
    try:
        # Create and send query
        query = DNSQuery(domain)
        packet = query.build_query()
        sock.sendto(packet, (dns_server, port))
        
        # Receive response
        response_data, _ = sock.recvfrom(1024)
        response = DNSResponse(response_data)
        
        return response
    finally:
        sock.close()
```

- The `dns_lookup` function handles the actual UDP communication
- We create a UDP socket with `socket.SOCK_DGRAM` (DGRAM = datagram = UDP)
- We set a timeout to avoid hanging if the DNS server doesn't respond
- We build the query packet, send it to the specified DNS server, and wait for a response
- The response is parsed and returned as a `DNSResponse` object
- The socket is always closed with `finally` to ensure resources are released

## 5. Command-Line Interface

```python
def main():
    parser = argparse.ArgumentParser(description='DNS Lookup Tool using UDP Protocol')
    parser.add_argument('domain', help='Domain name to lookup')
    parser.add_argument('--server', default='8.8.8.8', help='DNS server to query (default: 8.8.8.8)')
    parser.add_argument('--port', type=int, default=53, help='DNS server port (default: 53)')
    parser.add_argument('--timeout', type=int, default=5, help='Query timeout in seconds (default: 5)')
    
    args = parser.parse_args()
    
    try:
        print(f"Looking up {args.domain} using DNS server {args.server}:{args.port}")
        response = dns_lookup(args.domain, args.server, args.port, args.timeout)
        
        if not response.answers:
            print("No records found")
        else:
            print(f"\nResults for {args.domain}:")
            for record_type, data, ttl in response.answers:
                print(f"{record_type} record: {data} (TTL: {ttl}s)")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

- The `main` function provides a command-line interface using `argparse`
- It defines parameters for the domain name, DNS server, port, and timeout
- The function performs the lookup and displays the results in a human-readable format
- Any errors are caught and displayed to the user
- The script can be run directly as a command-line tool when executed as a script

## Why UDP for DNS?

DNS traditionally uses UDP on port 53 for several reasons:

1. **Efficiency**: UDP is connectionless, meaning there's no handshake overhead (like in TCP).
2. **Low latency**: No connection setup means faster query resolution.
3. **Small payload size**: Most DNS queries and responses fit in a single UDP packet.
4. **Statelessness**: DNS is designed to be stateless, matching UDP's stateless nature.
5. **Retries**: If a query fails, clients can simply retry the query.

While DNS can also use TCP (especially for zone transfers or when responses exceed UDP size limits), UDP is the primary protocol for standard queries. This implementation follows that standard approach by using UDP sockets for communication.
