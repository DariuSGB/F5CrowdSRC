#	Introduction
The iRule based RADIUS Client Stack can be used to perform RADIUS based user authentication via SIDEBAND UDP connections. The RADIUS Client Stack covers the RADIUS protocol core-mechanics outlined in RFC 2865 (see https://tools.ietf.org/html/rfc2865) and RFC 5080 (see https://tools.ietf.org/html/rfc5080) and can be utilized for a Password Authentication Protocol (PAP) authentication without requiring an APM-Addon license.

***Note:** The provided RADIUS Client Stack could be also useful for APM based deployments if more control of the RADIUS request/response attributes or granular RADIUS request retransmits and RADIUS response timeouts are required. Especially in cloud based 2FA RADIUS scenario (e.g. Microsoft Azure MFA) you will notice improved reliability by sending a couple RADIUS request retransmits at the beginning of the RADIUS conversation and then simply wait for the RADIUS response up to 60 seconds to allow the user to accept any push notifications or to answer the incoming phone call.*

#	Design of the iRule based RADIUS Client Stack
The design goal of the RADIUS Client Stack was to simplify its execution as much as possible, so that integrators of this iRule don't need to be experts of the RADIUS protocol specification or full time TCL developers. All you need to know is how to set up the configuration parameters to craft a RADIUS authentication request, how to execute the RADIUS Client Stack and how to evaluate the results of the RADIUS responses.

The RADIUS Client Stack is implemented by loading and storing TCL script code into global accessible variables within the `$static::*` namespace. This step is performed by the provided iRule using a `RULE_INIT` event. The TCL script code stored in the global accessible `$static::*` variables can later be accessed and `[eval]` executed by any other iRule stored on the same device.

***Note:** If you have never worked with the `[eval]` command, then simply think of a classic macro recorder that stores and loads a TCL script to/from a `$static::*` variable, which is then inserted and executed at a given position of your custom iRule which needs to perform a RADIUS Client authentication. It’s kind of a TCL procedure, only more in-line to the TCL code which has called the TCL macro code.*

The iRule sample below outlines how the RADIUS Client Stack can be `[eval]` executed from one of your iRule solutions. The provided sample contains a RADIUS Server configuration section (to assign certain RADIUS Server specific configuration options), a RADIUS request section (to set the username, password and optional RADIUS request attributes), the `[eval]` execution of the RADIUS Client Stack module and finally a RADIUS response section (to evaluate the outcome of the RADIUS response).

```
#####################################################
# RADIUS Server configuration

# RADIUS Server IP:Port or Virtual Server name
set server_config(address)	"192.168.0.1:1812"

# RADIUS shared secret
set server_config(shared_key)	"MySecureSharedKeyString"

# RADIUS response timeout in milliseconds
set server_config(timeout)	5000

# List of RADIUS retransmit timers in milliseconds
set server_config(retransmits)	"100 200 400 800"

#####################################################
# RADIUS request configuration

# Username to authenticate
set client_request(username) 	"MyUser"	

# Password to authenticate
set client_request(password) 	"MyP@ss1+"		

#####################################################
# Handler for RADIUS Client execution


eval $static::RadCLI_Processor

#####################################################
# RADIUS response evaluation
#

if { $server_response(message) eq "ACCEPT" } then {

	log local0.debug "USER-NAME \"$client_request(username)\" was accepted!
					( Response Code = \"$server_response(code)\",
					   Response MSG = \"$server_response(message)\") "

} elseif { $server_response(message) eq "REJECT" } then {

	log local0.debug "USER-NAME \"$client_request(username)\" was rejected!
					( Response Code = \"$server_response(code)\",
					   Response MSG = \"$server_response(message)\")"

} elseif { $server_response(message) eq "CHALLENGE" } then {

	log local0.debug "USER-NAME \"$client_request(username)\" was challenged!
					( Response Code = \"$server_response(code)\",
					   Response MSG = \"$server_response(message)\",
					       State-ID = \"$server_response(avp_24_0)\",
					       Question = \"$server_response(avp_18_0)\" )"

} else {

	log local0.debug "An error has been occurred!
					( Response Code = \"$server_response(code)\",
					   Response MSG = \"$server_response(message)\" )"

}
```

***Note:** By storing the TCL code into global variables and executing them via TCLs `[eval]` command, it is possible to keep the calling iRule as well as the RADIUS Client Stack Processor in the same TCL execution level. This approach eliminates the need to map variables into/from a child execution level like regular `[call]` procedures would require. Compared to a `[call]` procedure, the `[eval]` based execution also adds less overhead, since it does not require any housekeeping’s to create and maintain a TCL child execution level. An interesting background information in regards to this approach is, that the second representation of the TCL scripts stored in the `$static::*` variables are getting shimmered (see http://wiki.tcl.tk/3033) on their first execution to bytecode. This behavior allows subsequent `[eval]` executions to reuse the already computed bytecode representation. Compared to in-line TCL code, the performance overhead of those consecutive `[eval]` executions are in the range of just a few CPU cycles and completely independent of the executed TCL script syntax and its size.*

# Functionality of the RADIUS Server Configuration section
The RADIUS Server configuration section is responsible to specify the RADIUS Server configuration and `SIDEBAND` connection specific timer values to control UDP retransmits and timeouts. The mandatory configuration settings include the destination of the RADIUS Request (via `$server_config(address)`), the RADIUS Shared-Key (via `$server_config(shared_key)`) and the RADIUS response timeout (via `$server_config(timeout)`) and a list-item containing timer values for UDP retransmit (via `$server_config(retransmits)`).

***Note:** The destination could be either a IPv4:Port or IPv6:Port value or a Virtual Server name listening for RADIUS request and forwarding them to a redundant pool of RADIUS servers.* 

***Note:** Unlike traditional RADIUS Client implementations where the initial RADIUS request and each subsequent RADIUS retransmit has a linear-scaling timeout value, you could configure a free-form retransmit / timeout timeline like "Retransmit at 500 1000 1500 5000 5500 10000 10500 20000 20500 25000 25500" (each number represents a timeline entry in milliseconds) in combination with a long living total response timeout of 30000 milliseconds. By doing so you can aggressively burst a couple RADIUS request at the beginning of the RADIUS conversation to make sure that a packet loss is recovered as fast as possible and then simply wait to allow the RADIUS Server to authenticate the user.*

***Note:** Sending rather aggressive RADIUS retransmits should not be a problem for the overall authentication process if the RADIUS Server has adopted the recommendations of RFC5080. In this case the aggressive retransmits would be simply treated as duplicated UDP-Datagrams and therefore silently discarded by the RADIUS server. To test if your RADIUS Server has adopted the recommendations of RFC5080, you may try a very aggressive retransmit setting and then check in the RADIUS log files if such a burst has caused duplicate authentication attempts.*

# Functionality of the RADIUS Client Request configuration section 
In the RADIUS request section, you must specify at least the RADIUS request USER-NAME attribute (Attribute-ID 1) via the `$client_request(username)` variable and the RADIUS request PASSWORD attribute (Attribute-ID 2) via the `$client_request(password)` variable.

In case you need to send some custom RADIUS request attributes to your RADIUS server, you can optionally pass a user-friendly attribute list to the RADIUS Client Stack so that it includes those attributes in its requests. The RADIUS request attribute list is built by creating a three-part value pair specifying the Attribute-ID, Attribute-Format and Attribute-Value as outlined below.

```
#####################################################
# RADIUS request configuration

# Username to authenticate
set client_request(username) 	"MyUserName"	

# Password to authenticate
set client_request(password) 	"MyPassword"	

# Store the value raw value "Hello World" into RADIUS attribute 11
lappend client_request(attributes)	11	string	"Hello World"

# b64decode the value "SGVsbG8gV29ybGQ=" and store it into RADIUS attribute 12
lappend client_request(attributes)	12	base64	"SGVsbG8gV29ybGQ="

# Store the value "123" as 16-bit integer into RADIUS attribute 13
lappend client_request(attributes)	13	int16		"123"

# Store the value "234" as 32-bit integer into RADIUS attribute 14
lappend client_request(attributes)	14	int32		"234"

# Store the value "345" as 64-bit integer into RADIUS attribute 15
lappend client_request(attributes)	15	int64		"345"

# Decode the hexadecimal value "AFFEAFFE" and store it into RADIUS attribute 16
lappend client_request(attributes)	16	hex		"AFFEAFFE"

# Decode the IPv4 notation and store it into RADIUS attribute 17
lappend client_request(attributes)	17	ipv4		"10.10.10.10"

# Decode the IPv4 subnet-notation and store it into RADIUS attribute 18
lappend client_request(attributes)	18	ipv4prefix	"10.10.10.0/24"
```

***Note:** The RADIUS Client Stack automatically includes the NAS-IPv4 attribute (Attribute-ID 4) referencing the IPv4 of the executing Virtual Server, the Service-Type attribute (Attribute-ID 6) with the value set to Authenticate Only (Code 8), the Calling-Station-ID attribute (Attribute-ID 31) referencing the IPv4 of the connecting client and the NAS-ID attribute (Attribute-ID 32) referencing the name of the executing Virtual Server.*

***Note:** The RADIUS request PASSWORD attribute value (Attribute-ID 2) will become automatically decrypted by the RADIUS Client Stack by using the `$server_config(shared_key)` variable.*

***Note:** The RADIUS Client Stack also computes and sends a HMAC-based Message-Authenticator attribute (Attribute-ID 80) for each RADIUS request and verifies this attribute on RADIUS responses (if send by the RADIUS server).*

***Note:** For a comprehensive list of the available RADIUS request attributes and their data-types refer to the IANA web site (https://www.iana.org/assignments/RADIUS-types/RADIUS-types.xhtml).* 

# Functionality of the RADIUS Client Stack Processor
The main purpose of the RADIUS Client Stack Processor is to construct a RFC compliant RADIUS request based on the given configuration settings. Further to open and maintain a UDP connection to the RADIUS server, send an initial RADIUS request and trigger UDP retransmits as needed. It will wait until the RADIUS response is received to verify the integrity of the RADIUS response and finally output result variables back to the executing iRule. The RADIUS Client Stack Processor can be seen like an additional iRule command which can be integrated into your custom iRule solutions. The only difference to a native iRule `[command]` or `[call xyz]` procedure is, that the configuration options (aka. arguments) and the results (aka. return values) are getting passed via the `$server_config()`, `$client_request()` and `$server_response()` array variables.

In detail the RADIUS Client Stack Processor has the following functionality:
- Verification of the `$server_config()` and `$client_request()` configuration options.
- Encryption of the `$client_request(password)` value by using the `$server_config(key)`.
- Calculation of the HMAC-based Message-Authenticator request attribute.
- Construction of the default as well as user defined RADIUS request attributes.
- Construction of the RADIUS request headers including a randomly generated RADIUS authenticator attribute.
- UDP connection establishment, RADIUS request/response handling incl. tracking of retransmit timers and connection timeouts as well as UDP response deduplication.
- Verification of HMAC-based Message-Authenticator response attributes.
- RADIUS response code and RADIUS response attribute decoding and verification.
- Optional RADIUS request and response logging.
- Comprehensive logging support with adjustable log-levels.
- Integrated `[ISTATS]` performance counters.

# Workflow of the RADIUS Client Stack Processor
The workflow below outlines the detailed logic of the RADIUS Client Stack Processor. 

![This is an image](RadCLI%20-%20Processor.svg)
