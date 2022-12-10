---
title: Phishing Infrastructure
layout: default
nav_order: 2
---

# Building Infrastructure for Voice / SMS Phishing Engagements
{: .fs-9 }

---
{: .note }
Recently, I performed a phishing engagement using phone calls and text messages which required building VoIP infrastructure. This blog post is designed to help others build out their own infrastructure stack by outlining the steps I took in detail.


So what does it take to make a phone call using a VoIP number at a high-level? Three essenitals:

PBX
{: .label .label-blue}

SIP Trunk
{: .label .label-yellow}

DID Numbers
{: .label .label-green}

### PBX
To start you need a PBX, or private branch exchange, which is software that manages inbound and outbound VoIP calls. 

### SIP Trunk
Secondly, a SIP trunk is required to route calls from the PBX using your DID numbers to the global telephone network.

### DID Number(s)
Lastly, you need a DID number, which essentially are locally registered phone numbers that are used to provide access to the regular telephone network using VoIP.