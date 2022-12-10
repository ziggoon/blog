---
title: Phishing Infrastructure
layout: default
nav_order: 2
---

# Building Infrastructure for Voice / SMS Phishing Engagements
{: .fs-9 }

---

Recently, I performed a phishing engagement using phone calls and text messages which required building VoIP infrastructure.
So what does it take to make a phone call using a VoIP number at a high-level?

PBX
{: .label .label-blue}
To start you need a PBX, or private branch exchange, which is software that manages inbound and outbound VoIP calls. 

SIP Trunk
{: .label .label-yellow}
Secondly, a SIP trunk is required to route calls from the PBX using your DID numbers to the global telephone network.

DID Numbers
{: .label .label-green}
Lastly, you need a DID number, which essentially are locally registered phone numbers that are used to provide access to the regular telephone network using VoIP.