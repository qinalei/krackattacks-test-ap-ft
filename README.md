This script tests if APs are affected by CVE-2017-13082 (KRACK attack). See [the KRACK attack website for details](https://www.krackattacks.com) and also read [the research paper](https://papers.mathyvanhoef.com/ccs2017.pdf).
这个脚本是测试是否AP受CVE-2017-13082的影响。

# CVE-2017-13082: Key Reinstall in FT Handshake (802.11r)

Access Points (APs) might contain a vulnerable implementation of the Fast BSS Transition (FT) handshake. More precisely, a retransmitted or replayed FT Reassociation Request may trick the AP into reinstalling the pairwise key. 

AP可能具有FT握手漏洞。它是一个重传或是一个重放了FT重新关联请求，可以使用AP重新安装KEY对。

If the AP does not process retransmitted FT reassociation requests, or if it does not reinstall the pairwise key, it is not vulnerable. 
如果AP没有处理重传FT的重关联请求，或是它不能重装密钥对，那它就没有漏洞。
If it does reinstall the pairwise key, the effect is similar to the attack against the 4-way handshake, except that the AP instead of the client is now reinstalling a key. 
如果它重装了密钥对，其效果就和4次握手的攻击相似。除非AP代替客户端现在就重装了KEY



More precisely, the AP will subsequently reuse packet numbers when sending frames protected using TKIP, CCMP, or GCMP. This causes nonce reuse, voiding any security these encryption schemes are supposed to provide. Since the packet number is also used as a replay counter for received frames, frames sent *towards* the AP can also be replayed.

In contrast to the 4-way handshake and group key handshake, this is not an attack against the specification. That is, if the state machine as shown in Figure 13-15 of the 802.11-2016 standard is faithfully implemented, the AP will not reinstall the pairwise keys when receiving a retransmitted FT Reassociation Request. However, we found that many APs do process this frame and reinstall the pairwise key.


# Script Usage Instructions

We created scripts to determine whether an implementation is vulnerable to any of our attacks. These scripts were tested on Kali Linux. To install the required dependencies on Kali, execute:

	apt-get update
	apt-get install libnl-3-dev libnl-genl-3-dev pkg-config libssl-dev net-tools git sysfsutils python-scapy python-pycryptodome

Remember to disable Wi-Fi in your network manager before using our scripts. After doing so, execute `sudo rfkill unblock wifi` so our scripts can still use Wi-Fi.

The included Linux script `krack-ft-test.py` can be used to determine if an AP is vulnerable to our attack. The script contains detailed documentation on how to use it:

	./krack-ft-test.py --help

Essentially, it wraps a normal `wpa_supplicant` client, and will keep replaying the FT Reassociation Request (making the AP reinstall the PTK). We tested the script on a Kali Linux distribution using a USB WiFi dongle (a TP-Link WN722N v1).

Remember that this is not an attack script! You require credentials to the network in order to test if an access point is affected by the attack.

**This tool may incorrectly say an AP is vulnerable to due benign retransmissions of data frames.** However, we are already releasing this code because the script got leaked. Please run this script in an environment with low background noise. Benign retransmissions can be detected in the output of the script: if two data frames have the same `seq` (sequence number), it's a benign retransmission. Example of such a benign retransmission:

	[15:48:47] AP transmitted data using IV=5 (seq=4)
	[15:48:47] AP transmitted data using IV=5 (seq=4)
	[15:48:47] IV reuse detected (IV=5, seq=4). AP is vulnerable!

Here there was a benign retransmission of a data frame, because both frames used the same sequence number (`seq`). This wrongly got detected as IV reuse. There is code to fix this ready, but merging those fixes may take some time.


# Suggested Solution

If the implementation is vulnerable, the suggested fix is similar to the one of the 4-way handshake. That is, a boolean can be added such that the first FT Reassociation Requests installs the pairwise keys, but any retransmissions will skip key installation. Note that ideally the AP should still send a new FT Reassociation Response, even though it did not reinstall any keys.


# Impact and Exploitation Details

Exploiting this vulnerability does not require a man-in-the-middle position! Instead, an adversary merely needs to capture a Fast BSS Transition handshake and save the FT Reassociation Request. Because this frame does not contain a replay counter, the adversary can replay it at any time (and arbitrarily many times). Each time the vulnerable AP receives the replayed frame, the pairwise key will be reinstalled. This attack is illustrated in Figure 9 of the paper.

An adversary can trigger FT handshakes at will as follows. First, if no other AP of the network is within range of the client, the adversary clones a real AP of this network next to the client using a wormhole attack (i.e. we forward all frames over the internet). The adversary then sends a BSS Transition Management Request to the client. This request commands to the client to roam to another AP. As a result, the client will perform an FT handshake to roam to the other AP.

The included network trace [example-ft.pcapng](example-ft.pcapng) is an example of the attack executed against Linux's hostapd. When using the wireshark filter `wlan.sa == 7e:62:5c:7a:cd:47`, notice that packets 779 to 1127 all use the CCMP IV value 1. This was caused by malicious retransmissions of the FT reassociation request.


# This project is under a 2-clause BSD license

Copyright 2017 Mathy Vanhoef

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

