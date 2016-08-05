These are initial thoughts on what a DNS over HTTP should look like if we want
multiple message formats. A server that supports this is called a "DNS API
server" to differentiate from a "DNS server" that uses the regular DNS protocol.

The initial design focuses on stub-to-resolver, but needs to be expandable later
for resolver-to-authoritative.

## Use cases

* Application wants to avoid network path involvement with DNS.
  * Can be implemented in the app (e.g. browser) if configured, hard-coded,
    or app-discoverable (e.g. DHCP/RA fail for config here, but dns-sd
    might work)
  * Note: this breaks 6to4 DNS
* OS wants to help applications
  * DHCP/RA work
  * getaddrinfo(3) and friends get updated with support
  * OS config knobs allow entering URIs as well as IP addresses
* Small device that has COAP and CBOR already doesn't want to have a DNS
  parser, etc.
* Eventually: DETH-style editing
* DNS stapling in HTTP context - this is a bit different, an h2
  connection is allowed to push (authenticated) records at the client
  that it knows it is going to lookup.

## Discovery

Discovery of a DNS API resolver is the same as for a DNS resolver: DHCPv4,
DHCPv6, IPv6 RA, and configuration. The response of discovery is a URI template,
most likely HTTPS.

* Don't allow insecure URI templates, like HTTP
* Multiple templates might be configured or discovered.  Some of the URI types
  might not be supported by the resolver, that's ok.  For example, imagine
  getting both https: and coaps: URI templates.
* If a stub has a DNS resolver address, there should be both DNS and HTTP
  requests that gets the template back.
* Need to think about how to get both an IP address and domain name for
  the SNI for HTTPS. Related: RFC 7858 (DNS over TLS).

## Templates

Likely template would be
`/{queriedname}/{recordtype}[/?name1=value1&name2=value2&...]`
where `{queriedname}` is an FQDN (trailing dot is allowed, not required),
`{recordtype}` is numeric, and the optional name/value pair list are extensions. 
`{recordtype}` might also be allowed to be text like "AAAA" for
well-known types.
Any request that has extensions or modifications requires an extensions list.

Paul's alternative: No template as such, just `/?name1=value1&name2=value2&...`.
There must be one name/value pair with name "qname", where the value is
an FQDN (trailing dot is allowed, not required). If name "rtype" is not
given, it defaults to 1 ("A"). All other name/value pairs are optional.

## Protocol

Decide what to do about multiple in-flight requests and ordering.
Maybe require H2. 

Patrick's idea: leave it up to the application. H2 is a great approach, but
parallel H1 might work too. It is worth discussing for performance.

### Verbs

Define GET semantics initially, say that other verbs will be defined in the
future.  Other verbs will likely need different authorization semantics as well
as different discoverability for the template.

GET with no body does simple request with no EDNS, but a newly-defined set of
defaults. GET must be sent with `Accept:` header to say what Content-Types can be
returned. Patrick says: Maybe Accept: is not a requirement.

### Response

Define a JSON content type for responses, but say that others (such as "wire
format" and CBOR) are expected.

Decide how much (if any) of the existing header is to be retained in the JSON.
Retaining none of the header would be nice if possible, to encourage adoption.
If the header is wanted by some, put it in wherever the extension point is in
the response so it can be more easily ignored.  Each response record needs an
separate ID, for later editing.

Decide what to do with the other sections, such as authority and additional.

### Caching

Set HTTP cache headers according to shortest DNS TTL in the response.
Set etags, do If-None-Match.

## DNSSEC

Joe's idea: Always get all related DNSSEC records by default, or just turn off
DNSSEC for the Internet. Never return information about whether the recursive
resolver thought the DNSSEC was right; I can't trust that anyway, so it's
useless.  Consider what effect this has on response size, perhaps require
separate requests for names higher in the tree in order to get better caching.

Paul's idea: Don't pull DNSSEC data by default because few stubs can use it and
it causes a huge increase in response sizes. Of course, allow a stub to request
the DNSSEC records with the CD and DO bits. Return the AD bit in the response
just as it is done today. Don't use this protocol as a way of forcing DNSSEC
down the throats of stub resolvers, particularly in IoT.

Patrick's idea: provide a JavaScript library that can take the additional
DNSSEC data and validate it locally. I would return it by default but
should be configurable. (For HTTPS, DNS is not actually part of
the security model: it uses a different root for that. But if we
were going to staple cross-orgin DNS resolutions it would make sense
to require them to be DNSSEC verified. Thus, two different modes.)

## Security

Need to think about HTTP authorization mechanisms. This would allow user
tracking, but could also free resolvers from having to use IP address ranges for
filtering.  Several bad ideas are likely here, so let's think about it early.

Define what to do with cookies; one approach: always ignore, make sure not
to send.  Regardless, DO NOT allow * cookies.

Patrick says: It's kind of weird to say do not allow normal http mechanisms
like cookies in a foo-over-http spec. A lot of the libraries used for
implementation won't have great visibility into letting you do
that. It's tempting to just say do whatever you want for HTTP auth
(which could be digest challenges, it could be bearer cookies, it
could be bearer auth headers, it could be some wacky SAML thing.)

Patrick says: Require secure (HTTPS) templates. Paul agrees.

Patrick says: DNS API servers should understand CORS, especially to the extent that
they are deployed behind firewalls.

Patrick says: Does a redirect for a recursive resolver make any sense?
Paul says: Yes, definitely.
