These are initial thoughts on what a DNS over HTTP should look like if we want
multiple message formats. A server that supports this is called a "DNS API
server" to differentiate from a "DNS server" that uses the regular DNS protocol.

The initial design focuses on stub-to-resolver, but needs to be expandable later
for resolver-to-authoritative.

Discovery of a DNS API resolver is the same as for a DNS resolver: DHCPv4,
DHCPv6, IPv6 RA, and configuration. The response of discovery is a URI template,
most likely HTTPS.

* Why even allow HTTP?
* Why even allow other schemes if they don't have good prefix defintions?
* If a stub has a DNS resolver address, there should be a HTTPS request that gets the prefix back.
* Need to think about SNI for IP addresses

Likely template would /queriedname/recordtype where *queriedname* is an FQDN
(trailing dot is allowed, not required) and *recordtype* is numeric.

Define GET semantics initially, say that other verbs might be defined in the
future.

GET with no body does simple request with no EDNS. GET must be sent with Accept
header to say what Content-Types can be returned.

Any extensions or modifications to a simple request require extensions.
Extensions appear in the body. If a body is sent, the GET must have a
Content-Type header. Define at least one content type for the body (JSON, as a
dict) so others can model it.

Need to think about HTTP authorization mechanisms. This would allow user
tracking, but could also free resolvers from having to use IP address ranges for
filtering. (The Bad Idea Fairy says: define username of "anon" to be a universal
name so that an ISP only specifies a password so all of its clients are lumped.)

Define a JSON content type for responses, but say that others (such as "wire
format") are expected.

