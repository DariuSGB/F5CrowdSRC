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

