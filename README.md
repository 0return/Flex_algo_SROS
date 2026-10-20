Lab_Flex_algo — IGP Flexible Algorithm (SR-MPLS + IS-IS) with VPRN steering on Nokia SR OS

Four VPRNs between PE1 and PE2 share the same BGP next hop (2.2.2.2), yet each one crosses the network over a different path. Each VPRN resolves that next hop with a different IS-IS algorithm:

Algorithm	Optimizes / avoids	Service
0	IGP metric	INTERNET-A0
128	Delay	VOZ-A128
129	TE metric	BULK-A129
130	IGP metric, excluding RED links	BANCO-A130


The lab is built to show why each path is chosen, and what happens when links fail, FADs compete or nodes stop participating.
