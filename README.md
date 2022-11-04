# SSLproxy - transparent SSL/TLS proxy for decrypting and diverting network traffic to other programs for deep SSL inspection [![Build Status](https://travis-ci.com/sonertari/SSLproxy.svg?branch=master)](https://app.travis-ci.com/github/sonertari/SSLproxy)

Copyright (C) 2017-2022, [Soner Tari](mailto:sonertari@gmail.com).
https://github.com/sonertari/SSLproxy

Copyright (C) 2009-2019, [Daniel Roethlisberger](//daniel.roe.ch/).
https://www.roe.ch/SSLsplit

## Overview

SSLproxy is a proxy for SSL/TLS encrypted network connections. It is intended 
to be used for decrypting and diverting network traffic to other programs, such 
as UTM services, for deep SSL inspection. But it can handle unencrypted 
network traffic as well.

[The UTMFW project](https://github.com/sonertari/UTMFW) uses SSLproxy to 
decyrpt and feed network traffic into its UTM services: Web Filter, POP3 
Proxy, SMTP Proxy, and Inline IPS; and also indirectly into Virus Scanner and 
Spam Filter through those UTM software. Given that most of the Internet 
traffic is encrypted now, without SSLproxy it wouldn't be possible to deeply 
inspect most of the network traffic passing through UTMFW.

See [this presentation](https://drive.google.com/open?id=12YaGIGs0-xfpqMNAY3rzUbIyed-Tso8W) 
for a summary of SSL interception and potential issues with middleboxes that 
support it.

### Mode of operation

SSLproxy is designed to transparently terminate connections that are redirected
to it using a network address translation engine. SSLproxy then terminates
SSL/TLS and initiates a new SSL/TLS connection to the original destination
address. Packets received on the client side are decrypted and sent to the
program listening on a port given in the proxy specification. SSLproxy inserts
in the first packet the address and port it is expecting to receive the packets
back from the program. Upon receiving the packets back, SSLproxy re-encrypts
and sends them to their original destination. The return traffic follows the
same path back to the client in reverse order.

![Mode of Operation Diagram](https://drive.google.com/uc?id=1N_Yy5nMPDSvY8YaNFd4sHvipyLWq5zDy)

This is similar in principle to [divert 
sockets](https://man.openbsd.org/divert.4), where the packet filter diverts the 
packets to a program listening on a divert socket, and after processing the 
packets the program reinjects them into the kernel. If there is no program 
listening on that divert socket or the program does not reinject the packets 
into the kernel, the connection is effectively blocked. In the case of 
SSLproxy, SSLproxy acts as both the packet filter and the kernel, and the 
communication occurs over networking sockets.

SSLproxy supports split mode of operation similar to SSLsplit as well. In 
split mode, packets are not diverted to listening programs, effectively making 
SSLproxy behave similar to SSLsplit, but not exactly like it, because SSLproxy 
has certain features non-existent in SSLsplit, such as user authentication, 
protocol validation, and filtering rules. Also, note that the implementation 
of the proxy core in SSLproxy is different from the one in SSLsplit; for 
example, the proxy core in SSLproxy runs lockless, whereas SSLsplit 
implementation uses a thread manager level lock (which does not necessarily 
make sslproxy run faster than sslsplit). In SSLproxy, split mode can be 
defined globally, per-proxyspec, or per-connection using filtering rules.

SSLproxy does not automagically redirect any network traffic.  To actually
implement a proxy, you also need to redirect the traffic to the system running 
sslproxy.  Your options include running sslproxy on a legitimate router, ARP 
spoofing, ND spoofing, DNS poisoning, deploying a rogue access point (e.g. 
using hostap mode), physical recabling, malicious VLAN reconfiguration or 
route injection, /etc/hosts modification and so on.

#### Proxy specifications

SSLproxy supports three different types of proxy specifications, or proxyspecs 
for short, which can be in divert or split style.

- Command line proxyspecs passed on the command line
- One line proxyspecs in configuration files
- Structured proxyspecs in configuration files

The syntax of command line proxyspecs is as follows:

	(tcp|ssl|http|https|pop3|pop3s|smtp|smtps|autossl)
	  listenaddr listenport
	  [up:divertport [ua:divertaddr ra:returnaddr]]
	  [(targetaddr targetport|sni sniport|natengine)]

The syntax of one line proxyspecs is the same as the syntax of command line 
proxyspecs, except for the leading `ProxySpec` keyword:

	ProxySpec (tcp|ssl|http|https|pop3|pop3s|smtp|smtps|autossl)
	  listenaddr listenport
	  [up:divertport [ua:divertaddr ra:returnaddr]]
	  [(targetaddr targetport|sni sniport|natengine)]

The syntax of structured proxyspecs is as follows, and they can configure 
connection options too:

	ProxySpec {
	    Proto (tcp|ssl|http|https|pop3|pop3s|smtp|smtps|autossl)
	    Addr listenaddr       # inline
	    Port listenport       # comments
	    DivertPort divertport # allowed
	    DivertAddr divertaddr
	    ReturnAddr returnaddr
	    TargetAddr targetaddr
	    TargetPort targetport
	    SNIPort sniport
	    NatEngine natengine

	    # Divert or split
	    Divert (yes|no)

	    # Connection options
	    Passthrough (yes|no)

	    DenyOCSP (yes|no)
	    CACert ca.crt
	    CAKey ca.key
	    ClientCert client.crt
	    ClientKey client.key
	    CAChain chain.crt
	    LeafCRLURL http://example.com/example.crl
	    DHGroupParams dh.pem
	    ECDHCurve prime256v1
	    SSLCompression (yes|no)
	    ForceSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    DisableSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    EnableSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    MinSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    MaxSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    Ciphers MEDIUM:HIGH
	    CipherSuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
	    VerifyPeer (yes|no)
	    AllowWrongHost (yes|no)

	    RemoveHTTPAcceptEncoding (yes|no)
	    RemoveHTTPReferer (yes|no)
	    MaxHTTPHeaderSize 8192
	    ValidateProto (yes|no)

	    UserAuth (yes|no)
	    UserTimeout 300
	    UserAuthURL https://192.168.0.1/userdblogin.php

	    # The DivertUsers, PassUsers, and PassSite options will be deprecated
	    DivertUsers userlist
	    PassUsers userlist
	    PassSite rules

	    Define $macro valuelist

	    (Divert|Split|Pass|Block|Match) one line filtering rules
	    FilterRule {...} structured filtering rules
	}

For example, given the following command line proxyspec:

	https 127.0.0.1 8443 up:8080

- SSLproxy listens for HTTPS connections on 127.0.0.1:8443.
- Upon receiving a connection from the Client, it decrypts and diverts the 
packets to a Program listening on 127.0.0.1:8080. The default divert address 
is 127.0.0.1, which can be configured by the `ua` option.
- After processing the packets, the Program gives them back to SSLproxy 
listening on a dynamically assigned address, which the Program obtains from 
the SSLproxy line in the first packet in the connection.
- Then SSLproxy re-encrypts and sends the packets to the Server.

The response from the Server follows the same path back to the Client in 
reverse order.

Split style proxyspecs configure for split mode of operation similar to 
[SSLsplit](https://github.com/droe/sslsplit). See the SSLsplit documentation 
for the details of split style proxyspecs.

#### SSLproxy line

Given the proxyspec example above, a sample line SSLproxy inserts into the 
first packet in the connection may be the following:

	SSLproxy: [127.0.0.1]:34649,[192.168.3.24]:47286,[192.168.111.130]:443,s

- The first IP:port pair is a dynamically assigned address that SSLproxy 
expects the program send the packets back to it.
- The second and third IP:port pairs are the actual source and destination 
addresses of the connection, respectively. Since the program receives the 
packets from SSLproxy, it cannot determine the source and destination 
addresses of the packets by itself, for example by asking the NAT engine, 
hence must rely on the information in the SSLproxy line.
- The last letter is either s or p, for SSL/TLS encrypted or plain traffic, 
respectively. This information is also important for the program, because it 
cannot reliably determine if the actual network traffic it is processing was 
encrypted or not before being diverted to it.

#### Listening programs

The program that packets are diverted to should support this mode of operation.
Specifically, it should be able to recognize the SSLproxy address in the first
packet, and give the first and subsequent packets back to SSLproxy listening 
on that address, instead of sending them to their original destination as it 
normally would.

You can use any software as a listening program as long as it supports this 
mode of operation. So existing or new software developed in any programming 
language can be modified to be used with SSLproxy to inspect and/or modify any 
or all parts of the packets diverted to it.

Given the proxyspec example above, a program should be listening on port 8080.

You can offload the system SSLproxy is running on by diverting packets to 
remote listening programs too. For example, given the following proxy 
specification:

	https 127.0.0.1 8443 up:8080 ua:192.168.0.1 ra:192.168.1.1

- The `ua` option instructs SSLproxy to divert packets to 192.168.0.1:8080, 
instead of 127.0.0.1:8080 as in the previous proxyspec example.
- The `ra` option instructs SSLproxy to listen for returned packets from the 
program on 192.168.1.1, instead of 127.0.0.1 as in the previous SSLproxy line.

Accordingly, the SSLproxy line now becomes (notice the first IP address):

	SSLproxy: [192.168.1.1]:34649,[192.168.3.24]:47286,[192.168.111.130]:443,s

And a listening program should be running at address 192.168.0.1 on port 8080.

So, the listening program can be running on a machine anywhere in the world. 
Since the packets between SSLproxy and the listening program are always 
unencrypted, you should be careful while using such a setup.

### Protocols

#### Supported protocols

SSLproxy supports plain TCP, plain SSL, HTTP, HTTPS, POP3, POP3S, SMTP, and 
SMTPS connections over both IPv4 and IPv6. It also has the ability to 
dynamically upgrade plain TCP to SSL in order to generically support SMTP 
STARTTLS and similar upgrade mechanisms. Depending on the version of OpenSSL, 
SSLproxy supports SSL 3.0, TLS 1.0, TLS 1.1, TLS 1.2, and TLS 1.3, and 
optionally SSL 2.0 as well. SSLproxy supports Server Name Indication (SNI), 
but not Encrypted SNI in TLS 1.3. It is able to work with RSA, DSA and ECDSA 
keys and DHE and ECDHE cipher suites.

The following features of SSLproxy are IPv4 only:

- Divert addresses for listening programs in proxyspecs
- SSLproxy return addresses dynamically assigned to connections
- IP addresses in the ua and ra options
- IP and ethernet addresses of clients in user authentication
- Target IP and ethernet addresses in mirror logging

#### OCSP, HPKP, HSTS, Upgrade et al.

SSLproxy implements a number of defences against mechanisms which would
normally prevent MitM attacks or make them more difficult. SSLproxy can deny
OCSP requests in a generic way. For HTTP and HTTPS connections, SSLproxy
mangles headers to prevent server-instructed public key pinning (HPKP), avoid
strict transport security restrictions (HSTS), avoid Certificate Transparency
enforcement (Expect-CT) and prevent switching to QUIC/SPDY, HTTP/2 or
WebSockets (Upgrade, Alternate Protocols). HTTP compression, encodings and
keep-alive are disabled to make the logs more readable.

Another reason to disable persistent connections is to reduce file descriptor 
usage. Accordingly, connections are closed if they remain idle for a certain 
period of time. The default timeout is 120 seconds, which can be configured by 
the ConnIdleTimeout option.

#### Protocol validation

Protocol validation makes sure the traffic handled by a proxyspec is using the 
protocol specified in that proxyspec. If a connection cannot pass protocol 
validation, it is terminated. To enable protocol validation, the ValidateProto 
option can be defined globally, per-proxyspec, or per-connection using 
filtering rules. This feature currently supports HTTP, POP3, and SMTP 
protocols.

SSLproxy uses only client requests for protocol validation. However, it also 
validates SMTP responses until it starts processing the packets from the 
client. If there is no excessive fragmentation, the first couple of packets in 
the connection should be enough for validating protocols.

### Certificates

#### Certificate forging

For SSL and HTTPS connections, SSLproxy generates and signs forged X509v3
certificates on-the-fly, mimicking the original server certificate's subject
DN, subjectAltName extension and other characteristics. SSLproxy has the
ability to use existing certificates of which the private key is available,
instead of generating forged ones. SSLproxy supports NULL-prefix CN
certificates but otherwise does not implement exploits against specific
certificate verification vulnerabilities in SSL/TLS stacks.

#### Certificate verification

SSLproxy verifies upstream certificates by default. If the verification fails,
the connection is terminated immediately. This is in contrast to SSLsplit,
because in order to maximize the chances that a connection can be successfully
split, SSLsplit accepts all certificates by default, including self-signed
ones. See [The Risks of SSL Inspection](https://insights.sei.cmu.edu/cert/2015/03/the-risks-of-ssl-inspection.html)
for the reasons for this difference. You can enable or disable this feature by 
the VerifyPeer option, which can be defined globally, per-proxyspec, or 
per-connection using filtering rules.

#### Client certificates

SSLproxy uses the certificate and key from the pemfiles configured by the 
ClientCert and ClientKey options when the destination requests client 
certificates. These options can be defined globally, per-proxyspec, or 
per-connection using filtering rules.

Alternatively, you can use Pass filtering rules to pass through certain 
destinations requesting client certificates.

### User authentication

If the UserAuth option is enabled, SSLproxy requires network users to log in 
to the system to establish connections to the external network.

SSLproxy determines the user owner of a connection using a `users` table in an 
SQLite3 database configured by the UserDBPath option. The users table should 
be created using the following SQL statement:

	CREATE TABLE USERS(
	   IP             CHAR(45)     PRIMARY KEY     NOT NULL,
	   USER           CHAR(31)     NOT NULL,
	   ETHER          CHAR(17)     NOT NULL,
	   ATIME          INT          NOT NULL,
	   DESC           CHAR(50)
	);

SSLproxy does not create this users table or the database file by itself, nor 
does it log users in or out. So the database file with the users table should 
already exist at the location pointed to by the UserDBPath option. An external 
program should log users in and out on the users table. The external program 
should fill out all the fields in user records, except perhaps for the DESC 
field, which can be left blank.

When SSLproxy accepts a connection,

- It searches the client IP address of the connection in the users table. If 
the client IP address is not in the users table, the connection is redirected 
to a login page configured by the UserAuthURL option.
- If SSLproxy finds a user record for the client IP address in the users 
table, it obtains the ethernet address of the client IP address from the arp 
cache of the system, and compares it with the value in the user record for 
that IP address. If the client IP address is not in the arp cache, or the 
ethernet addresses do not match, the connection is redirected to the login 
page.
- If the ethernet addresses match, SSLproxy compares the atime value in the 
user record with the current system time. If the difference is greater than 
the value configured by the UserTimeout option, the connection is redirected 
to the login page.

If the connection passes all these checks, SSLproxy proceeds with establishing 
the connection.

The atime of the IP address in the users table is updated with the system time 
while the connection is being terminated. Since this atime update is executed 
using a privsep command, it is expensive. So, to reduce the frequency of such 
updates, it is deferred until after the user idle time is more than half of 
the timeout period.

If a description text is provided in the DESC field, it can be used with 
filtering rules to treat the user logged in from different locations, i.e. 
from different client IP addresses, differently.

If the UserAuth option is enabled, the user owner of the connection is 
appended at the end of the SSLproxy line, so that the listening program can 
parse and use this information in its logic and/or logging:

	SSLproxy: [127.0.0.1]:34649,[192.168.3.24]:47286,[192.168.111.130]:443,s,soner

The user authentication feature is currently available on OpenBSD and Linux 
only.

### Filtering rules

SSLproxy can divert, split, pass, block, or match connections based on 
filtering rules. Filtering rules can be defined globally and/or per-proxyspec.

- `Divert` action diverts packets to the listening program, allowing SSL 
inspection by the listening program and content logging of packets
- `Split` action splits the connection but does not divert packets to the 
listening program, effectively disabling SSL inspection by the listening 
program, but allowing content logging of packets
- `Pass` action passes the connection through by engaging passthrough mode, 
effectively disabling SSL inspection and content logging of packets
- `Block` action terminates the connection
- `Match` action specifies log actions and/or connection options for the 
connection without changing its filter action

SSLproxy supports one line and structured filtering rules.

The syntax of one line filtering rules is as follows:

	(Divert|Split|Pass|Block|Match)
	 ([from (
	     user (username[*]|$macro|*) [desc (desc[*]|$macro|*)]|
	     desc (desc[*]|$macro|*)|
	     ip (clientip[*]|$macro|*)|
	     *)]
	  [to (
	     (sni (servername[*]|$macro|*)|
	      cn (commonname[*]|$macro|*)|
	      host (host[*]|$macro|*)|
	      uri (uri[*]|$macro|*)|
	      ip (serverip[*]|$macro|*)) [port (serverport[*]|$macro|*)]|
	     port (serverport[*]|$macro|*)|
	     *)]
	  [log ([[!]connect] [[!]master] [[!]cert]
	        [[!]content] [[!]pcap] [[!]mirror] [$macro]|[!]*)]
	  |*) [# comment]

The syntax of structured filtering rules is as follows, and they can configure 
connection options too:

	FilterRule {
	    Action (Divert|Split|Pass|Block|Match)

	    # From
	    User (username[*]|$macro|*)  # inline
	    Desc (desc[*]|$macro|*)      # comments
	    SrcIp (clientip[*]|$macro|*) # allowed

	    # To
	    SNI (servername[*]|$macro|*)
	    CN (commonname[*]|$macro|*)
	    Host (host[*]|$macro|*)
	    URI (uri[*]|$macro|*)
	    DstIp (serverip[*]|$macro|*)
	    DstPort (serverport[*]|$macro|*)

	    # Multiple Log lines allowed
	    Log ([[!]connect] [[!]master] [[!]cert]
	         [[!]content] [[!]pcap] [[!]mirror] [$macro]|[!]*)

	    ReconnectSSL (yes|no)

	    # Connection options
	    Passthrough (yes|no)

	    DenyOCSP (yes|no)
	    CACert ca.crt
	    CAKey ca.key
	    ClientCert client.crt
	    ClientKey client.key
	    CAChain chain.crt
	    LeafCRLURL http://example.com/example.crl
	    DHGroupParams dh.pem
	    ECDHCurve prime256v1
	    SSLCompression (yes|no)
	    ForceSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    DisableSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    EnableSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    MinSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    MaxSSLProto (ssl2|ssl3|tls10|tls11|tls12|tls13)
	    Ciphers MEDIUM:HIGH
	    CipherSuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
	    VerifyPeer (yes|no)
	    AllowWrongHost (yes|no)

	    RemoveHTTPAcceptEncoding (yes|no)
	    RemoveHTTPReferer (yes|no)
	    MaxHTTPHeaderSize 8192
	    ValidateProto (yes|no)

	    UserAuth (yes|no)
	    UserTimeout 300
	    UserAuthURL https://192.168.0.1/userdblogin.php
	}

The specification of which connections a filtering rule will be applied to is 
achieved by the `from` and `to` parts of the filtering rule and by the 
proxyspec that the rule is defined for.

- The `from` part of a rule defines source filter based on client IP address, 
user and/or description, or `*` for all.
- The `to` part defines destination filter based on server IP and/or port, SNI 
or Common Names of SSL connections, Host or URI fields in HTTP Request 
headers, or `*` for all.
	+ Dst Host type of rules use the `ip` site field
	+ SSL type of rules use the `sni` or `cn` site field
	+ HTTP type of rules use the `host` or `uri` site field
	+ All rule types can use the `port` field
- The proxyspec handling the connection defines the protocol filter for the 
connection.

If and how a connection should be logged is specified using the `log` or 
`Log` part of one line or structured filtering rules, respectively:

- `connect` enables logging connection information to connect log file
- `master` enables logging of master keys
- `cert` enables logging of generated certificates
- `content` enables logging packet contents to content log file
- `pcap` enables writing packets to pcap file
- `mirror` enables mirroring packets to mirror interface or target

You can add a negation prefix `!` to a log action to disable that logging.

Structured filtering rules can also specify connection options to be 
selectively applied to matching connections, not just globally or 
per-proxyspec. One line filtering rules cannot specify connection options.

For example, if the following rules are defined in a structured HTTPS proxyspec,

	Split from user soner desc notebook to sni example.com log content
	Pass from user soner desc android to cn .fbcdn.net*

The first filtering rule above splits but does not divert HTTPS connections 
from the user `soner` who has logged in with the description `notebook` to SSL 
sites with the SNI of `example.com`. Also, the rule specifies that the packet 
contents of the matching connection be written to content log file configured 
globally.

The second rule passes through HTTPS connections from the user `soner` who has 
logged in with the description `android` to SSL sites with the Common Names 
containing the substring `.fbcdn.net` anywhere in it (notice the asterisk at 
the end). Since connection contents cannot be written to log files in 
passthrough mode, the rule does not specify any content log action.

The default filter action is Divert. So, if those are the only filtering rules 
in that proxyspec, the other connections are diverted to the listening program 
specified in that proxyspec, without writing any logs.

If you want to enable, say, connect logging for the other connections handled 
by that proxyspec, without changing their default Divert filter action, you 
can add a third filtering rule to that proxyspec:

	Match * log connect

Note that the second example above is a filtering rule you can use to resolve 
one of the certificate issues preventing the Facebook application on Android 
smartphones to connect to the Internet from behind sslproxy.

Filtering rules are applied based on certain precedence orders:

- More specific rules have higher precedence. Log actions increase rule 
precedence too.
- The precedence of filter types is as HTTP > SSL > Dst Host. Because, the 
application order of filter types is as Dst Host > SSL > HTTP, and a filter 
type can override the actions of a preceding filter type.
- The precedence of filter actions is as Divert > Split > Pass > Block. This is 
only for the same type of filtering rules.
- The precedence of site fields is as sni > cn for SSL filter and host > uri 
for HTTP filter.

For example, the pass action of a Dst Host filter rule is taken before the 
split action of an SSL filter rule with the same from definition, due to the 
precedence order of filter types. Or, the pass action of a rule with sni site 
field is taken before the split action of the same rule with cn site field, due 
to the precedence order of site fields.

Pass and Block filter actions are deferred until the last moment they can be 
applied to a connection, so that Divert and Split filter actions can override 
them.

In terms of possible filter actions,

- Dst Host filtering rules can take all of the filter and log actions.
- SSL filtering rules can take all of the filter and log actions.
- HTTP filtering rules can take match and block filter actions, can keep 
enabled divert and split modes, but cannot take pass action. Also, HTTP 
filtering rules can only disable logging.

Log actions do not configure any loggers. Global loggers for respective log 
actions should have been configured for those log actions to have any effect.

If no filtering rules are defined for a proxyspec, all log actions for that 
proxyspec are enabled. Otherwise, all log actions are disabled, and filtering 
rules should enable them specifically.

To increase rule reuse, one or more of SNI, CN, Host, URI, and DstIp site 
fields can be specified in the same structured filtering rule.

Connection options specified in a structured filtering rule can have any 
effect only if the rule matches the connection before global or proxyspec 
connection options are applied. Otherwise, the global or proxyspec connection 
options already applied to a connection cannot be overriden by the connection 
options specified in the matching structured filtering rule. For example, SSL 
options of a connection cannot be changed after the SSL connection is 
established. So, normally SSL type of rules cannot modify SSL options of a 
connection, but you can use the ReconnectSSL option to reconnect the server 
side of an SSL connection to enforce the SSL options in the SSL type of 
filtering rules. In other words, the ReconnectSSL option allows for using the 
SNI and CN fields in stuctured filtering rules to match SSL connections and 
change their SSL configuration.

Macro expansion is supported. The `Define` option can be used for defining 
macros to be used in filtering rules. Macro names must start with a `$` sign. 
The macro name must be followed by words separated by spaces.

You can append an asterisk `*` to the fields in filtering rules for substring 
matching. Otherwise, the filter searches for an exact match with the field in 
the rule. The filter uses B-trees for exact string matching and Aho-Corasick 
machines for substring matching.

The ordering of filtering rules is important. The ordering of from, to, and 
log parts of one line filtering rules is not important. The ordering of log 
actions is not important.

If the UserAuth option is disabled, only client IP addresses can be used in 
the from part of filtering rules.

#### Excluding sites from SSL inspection

PassSite option is a special form of Pass filtering rule. PassSite rules can 
be written as Pass filtering rules. The PassSite option will be deprecated in 
favor of filtering rules in the future.

PassSite option allows certain SSL sites to be excluded from SSL inspection. 
If a PassSite rule matches the SNI or Common Names in the SSL certificate of a 
connection, the connection is passed through the proxy without being diverted 
to the listening program. SSLproxy engages the Passthrough mode for that 
purpose. For example, sites requiring client authentication can be added as 
PassSite rules.

Per-site filters can be defined using client IP addresses, users, and 
description. If the UserAuth option is disabled, only client IP addresses can 
be used in PassSite filters. Multiple sites can be defined, one on each line. 
PassSite rules can search for exact or substring matches, but do not support 
macro expansion.

#### User control lists

User control lists can be implemented using filtering rules. The DivertUsers 
and PassUsers options will be deprecated in favor of filtering rules in the 
future.

DivertUsers and PassUsers options can be used to divert, pass through, or 
block users.

- If neither DivertUsers nor PassUsers is defined, all users are diverted to 
listening programs.
- Connections from users in DivertUsers, if defined, are diverted to listening 
programs.
- Connections from users in PassUsers, if defined, are simply passed through 
to their original destinations. SSLproxy engages the Passthrough mode for that 
purpose.
- If both DivertUsers and PassUsers are defined, users not listed in either of 
the lists are blocked. SSLproxy simply terminates their connections.
- If *no* DivertUsers list is defined, only users *not* listed in PassUsers 
are diverted to listening programs.

These user control lists can be defined globally or per-proxyspec. User 
control lists do not support macro expansion.

### Logging

Logging options include connect and content log files as well as PCAP files 
and mirroring decrypted traffic to a network interface. Additionally, 
certificates, master secrets and local process information can be logged. 
Filtering rules can selectively modify connection logging.

See the manual pages `sslproxy(1)` and `sslproxy.conf(5)` for details on using 
SSLproxy, setting up the various NAT engines, and for examples.


## Requirements

SSLproxy depends on the OpenSSL, libevent 2.x, libpcap, libnet 1.1.x, and 
sqlite3 libraries by default. Libpcap and libnet are not needed if the 
mirroring feature is omitted. Sqlite3 is not needed if the user authentication 
feature is omitted. The build depends on GNU make and a POSIX.2 environment 
in `PATH`. If available, pkg-config is used to locate and configure the 
dependencies. The optional unit tests depend on the check library. The 
optional end-to-end tests depend on the [TestProxy](https://github.com/sonertari/TestProxy) 
tool, and are supported on Linux only.

SSLproxy currently supports the following operating systems and NAT mechanisms:

- FreeBSD: pf rdr and divert-to, ipfw fwd, ipfilter rdr
- OpenBSD: pf rdr-to and divert-to
- Linux: netfilter REDIRECT and TPROXY
- Mac OS X: pf rdr and ipfw fwd

Support for local process information (`-i`) is currently available on Mac OS X
and FreeBSD.

SSL/TLS features and compatibility greatly depend on the version of OpenSSL
linked against. For optimal results, use a recent release of OpenSSL or
LibreSSL.


## Installation

With the requirements above available, run:

    make
    make test       # optional unit and e2e tests
    make sudotest   # optional unit tests requiring privileges
    make install    # optional install

Dependencies are autoconfigured using pkg-config. If dependencies are not
picked up and fixing `PKG_CONFIG_PATH` does not help, you can specify their
respective locations manually by setting `OPENSSL_BASE`, `LIBEVENT_BASE`,
`LIBPCAP_BASE`, `LIBNET_BASE`, `SQLITE_BASE` and/or `CHECK_BASE` to the 
respective prefixes.

You can override the default install prefix (`/usr/local`) by setting `PREFIX`.
For more build options and build-time defaults see [`main.mk`](Mk/main.mk)
and [`defaults.h`](src/defaults.h).


## Documentation

See the manual pages `sslproxy(1)` and `sslproxy.conf(5)` for user
documentation. See [`NEWS.md`](NEWS.md) for release notes listing significant
changes between releases and [`SECURITY.md`](SECURITY.md) for information on
security vulnerability disclosure.


## License

SSLproxy is provided under a 2-clause BSD license.
SSLproxy contains components licensed under the MIT, APSL, and LGPL licenses.
See [`LICENSE`](LICENSE), [`LICENSE.contrib`](LICENSE.contrib) and
[`LICENSE.third`](LICENSE.third) as well as the respective source file headers
for details.


## Credits

See [`AUTHORS.md`](AUTHORS.md) for the list of contributors.

SSLproxy was inspired by and has been developed based on [SSLsplit](https://www.roe.ch/SSLsplit) 
by Daniel Roethlisberger.


## Listening Program Example
```
import socket, traceback

HOST = ''
PORT = 8080
CLRF = '\r\n'

class InvalidRequest(Exception):
	pass

class Request(object):
	"A simple http request object"
	
	def __init__(self, raw_request):
		self._raw_request = raw_request
		
		self._respomse = self.parse_request()
	
	def parse_request(self):
		"Turn basic request headers in something we can use"
		temp = [i.strip() for i in self._raw_request.splitlines()]
		
		if -1 == str(temp[0]).find('HTTP'):
			raise InvalidRequest('Incorrect Protocol')
		startOfPort = temp[1].find(":",9)+1
		#portC= temp[1][startOfPort]
		endOfPort = temp[1].find(",")
		portC = temp[1][startOfPort:endOfPort]
		clientSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM);
		clientSocket.connect(("127.0.0.1",int(portC)));
		clientSocket.sendall(str.encode(self._raw_request));
		dataR = clientSocket.recv(40960);
		dataRDecoded = dataR.decode()
		if dataRDecoded.find("HTTP/1.0 200") ==0:
			clientSocket.close()
			return dataR
		clientSocket.close()

		
		return 1
	
	def __repr__(self):
		return repr({'method': self._method, 'path': self._path, 'protocol': self._protocol, 'headers': self._headers})
		
		

# the actual server starts here
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind((HOST, PORT))
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.listen(5)

while True:
	try:
		clientsock, clientaddress = s.accept()
	except KeyboardInterrupt:
		raise
	except:
		traceback.print_exc()
	
	try:
		request = clientsock.recv(1024)
		request = Request(request.decode('utf-8'))
		clientsock.send(request._respomse)
	except(KeyboardInterrupt, SystemExit):
		raise
	except InvalidRequest:
		clientsock.send('HTTP/1.1 400 Bad Request' + CLRF)
		clientsock.send('Content-Type: text/html' + CLRF*2)
		clientsock.send('<h1>Invalid Request: %s</h1>' )
	except:
		traceback.print_exc()
	
	try:
		clientsock.close()
	except KeyboardInterrupt:
		raise
	except:
		traceback.print_exc()
		
```
