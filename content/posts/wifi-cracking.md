+++ 
date = 2022-03-21
title = "WiFi cracking - WPA2 Personal"
description = "How to crack WPA2-PSK using hashcat"
slug = ""
authors = []
tags = ["writeup"]
categories = []
externalLink = ""
series = []
+++

## Table of Content

- [Background](#background)
  - [WPA2 PSK](#wpa2-psk-personal)
  - [PTK and the Four-way Handshake](#ptk-and-the-four-way-handshake)
  - [MIC](#mic) (the weakness to be exploited)
  - [PMKID](#pmkid) (other method to crack PSK)
- [Tools](#tools)
- [Steps](#steps)
- [Hash cracking](#hash-cracking)
  - [hxcpcapng](#hcxpcapng)
  - [Heuristics](#heuristics)
- [References](#references)
  - [WPA2 PSK](#wpa2-psk)
  - [PMKID](#pmkid-1)
  - [Hash cracking](#hash-cracking-1)

---

## Background

### WPA2 PSK (Personal)

---

Wireless Protected Access 2 (WPA2) is a standard established by the IEEE to secure wireless communication. PSK stands for Pre-Shared Key, meaning both the access point and the user knows a secret shared even before they establish a connection. The idea is, since both the access point and the client knows a secret, they can create a master key that can be used to encrypt/decrypt their traffic. If both parties can decrypt the message sent from each other, then they must have the secret components used to generate the master key. This secret, which is used by the access point to authenticate a user, is derived from the WiFi passphrase.

The generation of the Pairwise Master Key (PMK) is as follows:

```
PMK = PBKDF2(HMAC-SHA1, Passphrase, SSID, 4096, 256)

More details:
- HMAC-SHA1 : pseudorandom function used to derive the key
- passphrase: WiFi password, used as the master password for PBKDF2
- SSID      : WiFi name, used as cryptographic salt
- 4096      : number of PBKDF2 iteration
- 256       : length of the derived key
```

> Note that PMK is a general term used in all WPA2 standards. For WPA2 personal, PMK is the same with PSK. For WPA2 Enterprise, PMK is derived from the information (EAP parameter) gained from an authentication server.

### PTK and the Four-way Handshake

---

To decrease the risk of an attacker hacking into the network, the PMK should be exposed as little as possible. We do this by creating another key which will be used to encrypt one session only (session key). In WPA2, this session key is called the Pairwise Transient Key (PTK).

PTK is generated from a four-way handshake protocol:

> **IMPORTANT: SOME DETAILS ARE OMITTED FOR SIMPLICITY**

```
1. AP creates and sends AP Nonce
2. User creates Station Nonce, and derives the PTK. Then it sends the Station Nonce, together with MIC (Message Integrity Code).
3. AP derives the PTK from the STA Nonce, and sends the GTK with the MIC.
4. User sends ACK
```

> **NOTES:**
>
> 1. From the attacker point of view, the information contained in the first two handshake is sufficient to conduct an offline password attack
> 1. Should the passphrase provided by the station is incorrect, the MIC (part of the PTK) will be incorrect and the AP will refuse to continue the handshake.

More details regarding PTK and its derivation:

```
PTK = PRF-512(PMK, "Pairwise key expansion", Min(Mac AP, Mac STA) || Max(Mac AP, Mac STA) || Min(ANonce, SNonce) || Max(ANonce, SNonce))
---
* More details with regard to PRF-512 can be found on IEEE 802.11i-2004 section 8.5.1.1
```

PTK is divided into five keys:

1. **16 bytes of Key Confirmation Key (KCK) - used to compute MIC** (will be relevant for the next section)
2. 16 bytes of Key Encryption Key (KEK) - used by the AP during data encryption
3. 16 bytes of Temporal Key (TK) - used to encrypt unicast data packets
4. 8 bytes of Michael MIC Tx Key - used to compute MIC on unicast data packets transmitted by the AP
5. 8 bytes of Michael MIC Rx Key - used to compute MIC on unicast data packets transmitted by the station

The Michael MIC Authenticator Tx/Rx Keys are only used if the network is using TKIP to encrypt the data

### GTK

> Additional section, irrelevant for the attack

Having a new session key for each user has its limitation; the AP has to encrypt a broadcast/multicast message N times with different keys. To overcome this, the AP generate a GTK (Group Temporal Key) which will be used to encrypt broadcast/multicast messages. GTK is sent on the third handshake in the four-way handshake.

Some notes on GTK:

1.  GTK needs to be updated when one condition is met:

    - the timer expires
    - a device leaves the network (so that they cannot read the broadcast message after leaving the network)

1.  GTK is updated using two-way handshake, practically just the AP sending the new GTK to the clients and the clients send an ACK back to the AP.

### MIC

This is the Achilles's heel of WPA2; we shall use the value of MIC to get the passphrase candidates. First, we observe how MIC is derived:

```
MIC = HMAC-MD5(KCK, EAPoL data)    # WPA1
MIC = HMAC-SHA1(KCK, EAPoL data)   # WPA2
```

WPA2 uses the first 16 bytes of the PTK (KCK) as the key to create a keyed message authentication code. Now, notice that we can get the both MIC and EAPoL data by sniffing the four-way handshake. Given the EAPoL data, if we can find a PTK that can generate the same MIC as the one we sniffed, then we win.

We can then formulate our attack plan as follows:

1. Pick a passphrase, then compute the PMK/PSK = PBKDF2(HMAC-SHA1, Passphrase, SSID, 4096, 256)
1. From the PMK, compute the PTK = PRF-512(PMK, "Pairwise key expansion", Min(Mac AP, Mac STA) || Max(Mac AP, Mac STA) || Min(ANonce, SNonce) || Max(ANonce, SNonce))
1. Last, compute the MIC = HMAC-SHA1(PTK[:16], EAPoL data)
1. If MIC == Captured MIC, then we have successfully cracked WPA2. Else, repeat step 1.

### PMKID

---

> New attack on WPA2-PSK

Other than the PTK-EAPoL method, there is a new attack on WPA2-PSK found by Atom, the author of Hashcat. To ensure smooth roaming, PMKID is used by the AP to authenticate the user. This is especially important for time-sensitive application which uses WPA2-Enterprise network for which the authentication process could take almost one second.

```
PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)
```

Here, the attack is simpler:

1. Pick a passphrase, then compute the PMK/PSK = PBKDF2(HMAC-SHA1, Passphrase, SSID, 4096, 256)
1. Then compute the PMKID = HMAC-SHA1-128(PMK, "PMK Name" | MAC_AP | MAC_STA)
1. If the PMKID == Captured PMKID, then we win. Else, repeat step 1.

---

## Tools

To facilitate our attack, we will use the following tools:

- [fluxion](https://github.com/FluxionNetwork/fluxion)
- [hcxpcapngtool](https://github.com/ZerBea/hcxtools)
- [hashcat](https://hashcat.net/hashcat/)

---

## Steps

Here are the steps of performing the WiFi attack:

- Capture the traffic containing the four-way handshake using fluxion. By default, we need to wait for a user to connect to the network and perform the four-way handshake. Fluxion can force this by sending a deauth packet. The output is a `.cap` or `.pcap` file.

  ```
  sudo ./fluxion.sh
  ```

  ![fluxion-handshake](/images/wifi-cracking/fluxion-handshake.png)

  ![fluxion-target-search](/images/wifi-cracking/fluxion-target-search.png)

  ![fluxion-channel](/images/wifi-cracking/fluxion-channel.png)

  ![fluxion-target](/images/wifi-cracking/fluxion-target.png)

  ![fluxion-deauth](/images/wifi-cracking/fluxion-deauth.png)

  ![fluxion-tracking](/images/wifi-cracking/fluxion-tracking.png)

  ![fluxion-monitor-jamming](/images/wifi-cracking/fluxion-monitor-jamming.png)

  ![fluxion-verify](/images/wifi-cracking/fluxion-verify.png)

  ![fluxion-interval](/images/wifi-cracking/fluxion-interval.png)

  ![fluxion-sync](/images/wifi-cracking/fluxion-sync.png)

- Extract the information we need (i.e. MIC, EAPoL, etc) from the `.cap` file using hcxpcapngtool. The output contains the hash which hashcat understands.

  ```
  hxcpcapngtool -o hash-file.txt pcapfile.cap
  ```

  ![hcxpcapng](/images/wifi-cracking/hcxpcapng.png)

  ![hcxpcapng-hash](/images/wifi-cracking/hcxpcapng-hash.png)

  Technical detail of hxcpcapngtool can be found on the next section.

- Crack the password using hashcat

  ```
  # dictionary attack using wordlist
  hashcat -a 0 -m 22000 -w 4 hash-file.txt wordlist

  # brute-force mask attack, 8 lowercase letters
  hashcat -a 3 -m 22000 -w 4 hash-file.txt ?l?l?l?l?l?l?l?l

  Note:
  -a  : attack type       0 = dictionary, 3 = brute-force
  -m  : hash type         22000 = WPA-PBKDF2-PMKID+EAPOL
  -w  : workload profile  4 = nightmare (fastest)
  ```

  ![hashcat-poc](/images/wifi-cracking/hashcat-poc.png)

  We intentionally put the wifi in the dictionary as a proof of concept.

---

## Hash Cracking

### hcxpcapng

Given the pcap file, hcxpcapngtool will extract the relevant handshake packet and put it in a text file with this format:

```
WPA*01*PMKID*MAC_AP*MAC_CLIENT*ESSID***
WPA*02*MIC*MAC_AP*MAC_CLIENT*ESSID*NONCE_AP*EAPOL_CLIENT*MESSAGEPAIR
```

Example:

```
WPA*02*3847cec66523043ac915521085d9ecec*302303c5eeb3*94659c310204*6c696e6b737973322e3447487a*b1ad763b12e9529081592ec69b43538cd4c2650f8e5761591e65349279cfed68*0103007702010a000000000000000000014668fef7d68df98ac78c2bc1ea480cc029b16e4d1a64f5027b42819f3990a47e000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001830160100000fac040100000fac040100000fac023c000000*02
```

See the following image to see the comparison between the capture file and the output of hcxpcapng:

![wpa-hash](/images/wifi-cracking/WPA-HASH.jpg)

Refer to [hashcat documentation](https://hashcat.net/wiki/doku.php?id=cracking_wpawpa2) for the explanation of the MESSAGEPAIR field.

### Heuristics

All in all, this attack boils down to conventional offline password attack. Thus, given a strong password, it is practically impossible to retrieve the passphrase. Brute force is obviously not an option, and therefore we usually go with dictionary attacks. Dictionary attack really depends on the word list used. To increase the probability of getting the key, the following password cracking heuristics can be used to generate

- l33t substitution
- prince attack https://github.com/hashcat/princeprocessor
- word list drawn from victim's website https://github.com/digininja/CeWL
- toggle upper n lower case https://github.com/hashcat/hashcat/blob/master/rules/toggles5.rule
- combine words from dictionaries https://hashcat.net/wiki/doku.php?id=combinator_attack
- add character before/inside/after the word https://hashcat.net/wiki/doku.php?id=maskprocessor

## References

### WPA2-PSK:

- IEEE Std 802.11i-2004
- [IEEE 802.11.i-2004 - Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11i-2004)
- [Understanding WPA-PSK cracking](https://ins1gn1a.com/understanding-wpa-psk-cracking)
- [Cracking Wi-Fi Protected Access, Part 2 - Cisco](https://ciscopress.com/articles/article.asp?p=370636&seqNum=6)
- [aircrack-ng](https://www.aircrack-ng.org/doku.php?id=cracking_wpa)
- [hashcat: cracking wpa](https://hashcat.net/wiki/doku.php?id=cracking_wpawpa2)
- [4 way Handshake vs PMKID](https://hashcat.net/forum/thread-8285-page-2.html)

### PMKID:

- [New attack on WPA/WPA2 using PMKID](https://hashcat.net/forum/thread-7717.html) (Original writeup)
- [Cracking wifi at scale](https://www.cyberark.com/resources/threat-research-blog/cracking-wifi-at-scale-with-one-simple-trick)
- [What is PMKID?](https://superuser.com/questions/1547307/what-is-pmkid-why-would-even-a-router-give-away-the-pmkid-to-an-unauthorized-st)

### Hash Cracking

- https://hashcat.net
- https://github.com/hashcat/team-hashcat/blob/main/CMIYC2017/CMIYC2017WriteupHashcat.pdf
- https://github.com/hashcat/team-hashcat/blob/main/CMIYC2021/CMIYC2021TeamHashcatWriteup.pdf
