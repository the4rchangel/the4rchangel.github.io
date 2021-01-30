---
layout: post
title:  "Email Spoofing With Netcat or Telnet"
author: Michael
categories: [ netcat, telnet, spoofing, email ]
image: assets/images/netcat3.png
featured: false
hidden: false
---
<blockquote> I have also written a follow up post about <a href="https://exploits.run/email-spoofing-powershell/">spoofing with powershell here</a>.</blockquote>
Recently, while having a discussion with a security research team I’m on, we stumbled into discussion about email spoofing. This ultimately led to all sorts of shenanigans including an email from the President himself! The more we talked the more we discussed the ease of pulling off something like this against the average user. In this post, I plan to examine basic concepts of email and email spoofing though, of course, it will not be exhaustive and you should do some additional research on how to identify emails like this or advanced spoofing concepts.
<p><img src="/assets/images/netcat1.png"></p>

#### THE SYNTAX

You can initiate a connection to a mail relay using Netcat or Telnet, traditionally. Identifying a mail relay server isn’t very difficult as it’s easily found in a domains MX records. For example, if I wanted to send an email to a gmail address, I can simply look up the DNS records and I’m provided with a list of their MX servers.
<p><img src="/assets/images/netcat2.png"></p>
<blockquote>Image from a search conducted using DNSdumpster.com</blockquote>
Using this as an example, let’s walk through the spoof and see how it plays out.

<strong>Windows:</strong>
`telnet qr-in-f26.1e100.net 25`

<strong>Linux:</strong>
`nc -C qr-in-f26.1e100.net 25`

These commands will initiate a connection to one of the gmail relay servers where we can imitate a mail client and send the email. Let’s look at a finished mail below and see what happened:
<p><img src="/assets/images/netcat3.png"></p>

#### COMPOSING THE EMAIL

After initiating our netcat connection we receive a 220 from the email server letting us know they’re ready to accept information from us. We send the server a “HELO” telling them we are ‘gmail.com’ and they acknowledge it with a 250. Then we tell them who is sending the mail, this can be anyone. In this case we chose ‘chuck@assurancedata.com’. We set the victims address with “RCPT TO”. Then we send a “DATA” request to begin forming the email that the victim will see. The data fields it will first be looking for are laid out above in order: From, To, Subject, and Date.

If you examine the layout above, you’ll see you can select the display name and email address they see (you’ll see this in the received email I’ll show you below). The funky characters in the Date field will just add the current time/date information in the format the email server is looking for. You can elect to put different time/date information should you choose to do so but it will make mail servers suspicious. Lastly, when you’re done with the email, you place a single period (.) on a line of it’s own and hit “Enter”. This let’s the mail server know you’re done with your message.
<p><img src="/assets/images/netcat4.png"></p>


The email did indeed make it to my Gmail inbox. You’ll see that by all intents and purposes it appears to be directly from “Chuck” at Assurance Data. What’s that red question mark? That is a notification by Google stating that they can’t verify this email came from Chuck. The reason for this is the email was sent without authenticating. Because no authentication took place on the mail server, the web app automatically wants you to know. But if you’re the average user and you use something like Outlook or a 3rd party mail app, you’ll likely not see anything like that.
<p><img src="/assets/images/netcat5.png"></p>
<blockquote>Thanks Chuck!</blockquote>

#### TIP OF THE ICEBERG

As I started off by saying, there’s so much more to be said about this. This only offers a brief overview of how to send a spoofed email through netcat/telnet. There are many precautions that email providers like Google take as well as businesses. The email headers are littered with information that these emails aren’t legitimate. For this to be anything more than a prank, an attacker or Red Teamer is going to have to be a bit more… creative in their approach. There are many ways to avoid that big red question mark, just ask Mr. President.
<p><img src="/assets/images/netcat6.png"></p>

#### STUFF YOU CAN COPY AND USE

Here’s the summary exchange for you to look at for your own use:

```
nc -C MAIL.SERVER.COM 25
(mail server says hello)
HELO spoofingdomain.com
(mail server acknowledges)
MAIL FROM: <spoofed@spoofingdomain.com>
(server says OK)
RCPT TO: <victim@address.com>
(server says OK)
DATA
(server says go)
From: Display Name <spoofed@spoofingdomain.com>
To: Victim Name <victim@address.com>
Subject: Hacking you
Date: Wed, 26 Sep 2018 14:21:26 -0400
Hello,
This is the email body.
Goodbye
.
(Server sends email)
```
