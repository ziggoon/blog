---
title: Phishing Infrastructure
layout: default
nav_order: 2
---

# Building Infrastructure for Voice / SMS Phishing Engagements
_Recently, I performed a phishing engagement using phone calls and text messages which required building VoIP infrastructure. This blog post is designed to help others build out their own infrastructure stack by outlining the steps I took._ 
{: .fs-3 .fw-300 .text-grey-dk-000 }

So what does it take to make a phone call using a VoIP number at a high-level? Three essentials:

PBX
{: .label .label-purple}

SIP Trunk
{: .label .label-blue}

DID Numbers
{: .label .label-green}

### PBX
To start you need a PBX, or private branch exchange, which is software that manages inbound and outbound VoIP calls. There are many options such as [FreePBX](https://www.freepbx.org/) and [3CX](https://www.3cx.com/). I chose 3CX due to their well-documented AWS installation process.

### SIP Trunk
Secondly, a SIP trunk is required to route calls from the PBX using your DID numbers to the global telephone network. [Twilio](https://www.twilio.com/en-us/sip-trunking) is 100% the best bet when it comes to SIP trunking due to their streamlined configuration process and pay-as-you-go policy. For more information on SIP trunking I recommend reading the Twilio [docs](https://www.twilio.com/docs/sip-trunking).

### DID Number(s)
Lastly, you need a DID number, which essentially are locally registered phone numbers that are used to provide access to the regular telephone network using VoIP. Twilio is also the best bet here.

---
# Initial setup
### AWS / 3CX 
The first step in this process is to configure a cloud PBX, or on-prem if you really want to go that route, but this guide will focus on 3CX hosted on AWS. 3CX provided a detailed [guide](https://www.3cx.com/docs/cloud-pbx-amazon-aws/) on how to host their product on AWS and I highly recommend following it. Once complete your 3CX console should look something like this: ![3CX console](../../assets/img/3cxconsole.png)
Make sure to take note of the public IPv4 address of the EC2 instance as it is required for configuring the SIP trunk.

### Twilio SIP Trunk
Create a Twilio account and navigate to this [url](https://console.twilio.com/develop/explore). Scroll to the "Super Network" section and select "Elastic SIP Trunking". ![](../../assets/img/twilio_explore.png){:width="80%" :float="left"} ![](../../assets/img/twilio-sidebar.png){:width="15%" :float="right"}
First, whitelist the EC2 instance under "Manage > IP access control lists" using its public IPv4 address. Next, create a credential for authorization on the trunk. 
Third step is to create the trunk and configure it with our 3CX instance. Under "Manage > Trunks" click "Create a new trunk" and enter a name for the trunk, I chose 3CX for simplicity. 

Now we must configure termination by adding the authentication methods we previously created (IP control list / authentication creds) and add a unique termination url. 
![](../../assets/img/trunk-termination.png)
Finally, add the origination url which is your assigned 3CX domain name, i.e (sip:phishing.fl.3cx.us) with a priority and weight on 10. ![](../../assets/img/trunk-origination.png)

### Twilio DID Numbers
If you require your calls to be made from a certain geopgraphical location you can search via area code. Once you purchase a number head to the management page and change the Voice & Fax configuration to your SIP trunk. 
![](../../assets/img/trunk-addnumber.png)

---
# 3CX Configuration
Login to your 3CX management console, you should see a dashboard similar to this:
![](../../assets/img/3cx-dashboard.png)

### Add SIP Trunk
Navigate to "SIP Trunks" and click "Add SIP Trunk". For country, select "Worldwide", provider as "Twilio", and "Main Trunk No" should be your Twilio DID number that you added to the trunk in [E.164](https://www.twilio.com/docs/glossary/what-e164) format (+15551234567).
![](../../assets/img/3cx-addtrunk.png)
Next, under "Registrar/Server/Gateway Hostname or IP" enter your SIP trunk's termination uri **(\<\>.ptsn.twilio.com)**.
Under "Authentication" you will want to enter the credential you added to the SIP trunk on Twilio and select "Do not require - IP based".
Finally, add your main trunk number in the "Route to" section.

The rest of the trunk configuration settings can be left default.
![](../../assets/img/3cx-trunkconfig1.png)

### Configure Outbound Rule
Navigate to "Outbound Rules" and add a new rule. The rule can have any name. Apply the rule to numbers with a length of 10 and configure "Route 1" to be routed to your trunk, strip 0 digits, and prepend +1.

_This will allow you to type the 10 digit number instead of manualy adding a +1 to each call._
{: .fs-3 .text-grey-dk-000}
![](../../assets/img/3cx-outboundrule.png)

This concludes the basic configuration of the PBX. You shoud now be able to make calls from your 3CX instance to any US phone number.
{: .fs-3 .fw-300 .text-grey-dk-000 }
---

# Custom SMS Tool
![](../../assets/img/spoofy.png)

[Spoofy](https://github.com/ziggoon/red-team/tree/main/spoofy) -
SMS messages can be sent through the 3CX web client, but I had some free time and wanted to expand my Rust skills, so I thought why not just write the SMS tool in Rust. The Twilio API was pretty straightforward and I found this nice [crate](https://github.com/neil-lobracco/twilio-rs) that eased my developer pains. I deployed it on a t2.micro AWS instance for the duration of the engagement and was very happy with the results.

This tool is capable of sending and receiving messages, which are then stored in a SQLite database. Moving forward I want to add a few more features such as message templates and some security measures, but for a first iteration it worked very well and I was pleased. 