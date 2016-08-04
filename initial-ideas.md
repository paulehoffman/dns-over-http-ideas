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
* Need to think about SNI for IP addresses; how do you ensure you're talking
  to the DNS API server you think you're talking to, if that thing doesn't
  have a name?  If it does have a name, how do you bootstrap
  resolution? [mcmanus - it needs a name to do
  authentication. probably need to discover a full origin and
  bootstrap address]

## Templates

Likely template would be `/{queriedname}/{recordtype}` where `{queriedname}`
is an FQDN (trailing dot is allowed, not required) and `{recordtype}` is
numeric.  `{recordtype}` might also be allowed to be text like "AAAA" for
well-known types.

## Protocol

Decide what to do about multiple in-flight requests and ordering.
Example: require H2.
[mcmanus - leave it up to the http app. H2 is a great approach, but
parallel H1 might work too. worth discussing for perf, but they have
the same http semantics]

### Verbs

Define GET semantics initially, say that other verbs will be defined in the
future.  Other verbs will likely need different authorization semantics as well
as different discoverability for the template.

GET with no body does simple request with no EDNS, but a newly-defined set of
defaults. GET must be sent with `Accept:` header to say what Content-Types can be
returned. [mcmanus - http has some things to say about
accept. Generally its very good advice to honor it, but its not a requirement]

### Parameterized requests

Any extensions or modifications to a simple request require extensions.
Extensions appear in the body. If a body is sent, the GET must have a
Content-Type header. Define at least one content type for the body (JSON, as a
dict) so others can model it.

[mcmanus - don't send GET with body. Its technically allowed but in
practice will give you problems by surprising other elements. The only
reason you might entertain it would be to allow caching, but no cache
in its right mind is going to save and compare request bodies as cache
keys anyhow. Go POST imo if this can't be encoded in the URI (which
would be preferable).]

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

McManus's idea: provide a js library that can take the json with
dnssec data and validate it locally. I would return it by default but
should be configurable. (e.g. for HTTPS DNS is not actually part of
the security model - it uses a different root for that.. but if we
were going to staple cross-orgin DNS resolutions it would make sense
to require them to be DNSSEC verified.. so two different modes).

## Security

Need to think about HTTP authorization mechanisms. This would allow user
tracking, but could also free resolvers from having to use IP address ranges for
filtering.  Several bad ideas are likely here, so let's think about it early.

Define what to do with cookies; one approach: always ignore, make sure not
to send.  Regardless, DO NOT allow * cookies.

[mcmanus: Its kind of weird to say do not allow normal http mechanisms
like cookies in a foo-over-http spec.. a lot of the libraries used for
implementation won't have great visibility into letting you do
that. Its tempting to just say do whatever you want for http auth
(which could be digest challenges, it could be bearer cookies, it
could be bearer auth headers.. it could be some wacky SAML thing..)

require secure templates ala https

for considerations: DNS API servers should understand CORS.. especially to the extent that
they are deployed behind firewalls

does a redirect for a recursive resolver make any sense?]
