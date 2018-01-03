| Title  | External module repository |
|--------|----------------------------|
| Author | @CurlyMoo                  |
| Status | Draft                      |
| Date   | 31-12-2017                 |

Proposal
========
For my own software `pilight` I have developed various modules that I use on top of libuv. The reason for me to develop this stuff myself and not use existing libraries, is that I can use a single libuv loop for all functionality I need. Using external libraries require additional threads, additional non-blocking io loops etc. The downside of this, is that my code is not as complete as some existing libraries, and the code is probably of less quality. Despite all of this, what I have written so far is:

- NTP synchronisation (IPv6 supported)
- HTTP post and get (IPv6 and SSL supported)
- SSDP server and client
- MAIL sent through smtp (IPv6 and SSL supported)
- Webserver with websockets (SSL supported)
- Eventpool based on the observer pattern.
- ARP discovery.
- ICMP.

The SSL library used is mbedtls.

All code has been written in pure C and is mostly modular. The biggest downside is that all code uses my eventpool for handling SSL connections. I therefor also use the pure `uv_poll_t` functionality a lot to keep close to socket programming.

However, that of course doesn't mean that things cannot be improved further with the help of others.

So, my proposal is to create a repository that hosts all kinds external components built on top of libuv. Other developers can then help maintain the code and ensure its functionality and quality. By having these modules stored close to libuv, other developers can more easily find existing implementations on top of libuv, and make use of it.
