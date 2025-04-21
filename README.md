# DNS Lookup Tool

A Python-based DNS lookup tool that uses raw UDP sockets to perform DNS queries. This tool provides insight into how DNS queries work at the protocol level by directly implementing the DNS message format.

## Features

- Performs DNS lookups without relying on high-level DNS libraries
- Supports A (IPv4) and AAAA (IPv6) record queries
- Basic support for other DNS record types
- Configurable DNS server, port, and timeout
- Command-line interface with helpful options

## Requirements

- Python 3.6 or higher
- No external dependencies

## Installation

1. Clone the repository or download the script:

```bash

git clone https://github.com/Prashithshetty/dns-lookup.git

```


## Usage

Basic usage:

```bash
python dns-lookup.py example.com
```

Advanced usage:

```bash
python dns-lookup.py example.com --server 1.1.1.1 --port 53 --timeout 3
```

### Command-line Arguments

- `domain`: The domain name to look up (required)
- `--server`: DNS server to use (default: 8.8.8.8)
- `--port`: DNS server port (default: 53)
- `--timeout`: Query timeout in seconds (default: 5)

## Example Output

```
Looking up example.com using DNS server 8.8.8.8:53

Results for example.com:
A record: 93.184.216.34 (TTL: 86400s)
AAAA record: 2606:2800:220:1:248:1893:25c8:1946 (TTL: 86400s)
```

## How It Works

The tool implements the following steps:

1. **Query Construction**: Creates a DNS query packet with appropriate headers and question section
2. **UDP Communication**: Sends the packet to the specified DNS server using a UDP socket
3. **Response Parsing**: Parses the binary response according to the DNS protocol specification
4. **Result Extraction**: Extracts and displays relevant information from the response

## Implementation Details

The implementation consists of two main classes:

- `DNSQuery`: Builds DNS query packets
- `DNSResponse`: Parses DNS response packets

The main `dns_lookup` function handles the socket communication.

## Limitations

- The implementation is simplified and doesn't support all DNS record types
- CNAME resolution is rudimentary
- The tool doesn't handle DNS recursion or follow referrals

## Further Development

Potential improvements to consider:

- Support for additional DNS record types (MX, TXT, NS, etc.)
- Better handling of CNAME records and record chaining
- Support for DNS-over-HTTPS (DoH) or DNS-over-TLS (DoT)
- Recursive resolution capabilities
- Query caching

## License

MIT License

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
