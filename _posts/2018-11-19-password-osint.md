---
layout: post
title:  "Using Password Resets for OSINT"
author: Michael
categories: [ OSINT, Passwords ]
image: assets/images/passosint1.png
featured: false
hidden: false
---
This post is part practical, but mostly story. I’ll go through how I use password resets on various services to gather fragments of information on someone, alongside a story of how I piece those together to get more definitive information. There may be tools out there that do something similar to this but my process is predominately manual. There is no step by step guide here as it’s more about attention to detail and your ability to piece things together. This is just an example of how I put that to use personally.

#### WHAT YOU CAN GATHER

This form of enumeration can be used to enumerate email addresses, usernames, or phone numbers, with another username, alternate email address, or phone number. Keep in mind, most services have begun taking measures to prevent or at least minimize this kind of information gathering. However, for a persistent researcher, you can still achieve success many times. Typically, what you’ll find it is a redacted glimpse of the information they have on file for a person which you can piece together with other information you find to build a more complete picture.

#### PUTTING IT TO USE

Recently, someone had gotten a hold of my résumé and reached out to me about a potential opportunity. They asked about speaking the following day at a specific time. I responded promptly letting them know that I was certainly interested in speaking at that time. The next day came around and I hadn’t heard back from them on the specifics of the call and there was no phone contact in their email signature. I sent a follow up email reaffirming my interest in speaking and inquiring about the format of the call. Should I be expecting a call? Is there a conference line I should dial into? Silence. The call time came and went and I was disappointed.

At this point, I had a few thoughts:
* Something came up on her side.
* She changed her mind and went with someone else.
* She’s not receiving my emails.

All of those seem like potential options in this scenario but I was leaning on the idea that my emails were getting filtered by their mail system. So, if she isn’t getting my emails then it doesn’t make sense to continue sending them. On the other hand, if she is receiving them then it probably still doesn’t make sense to keep sending them. The day of the call ended and I still hadn’t heard back.

That evening I made a decision that I need to find an alternative way to reach out because if she’s not receiving my emails she may assume I’m not interested and I may lose out on this opportunity. Given that there was no alternative contact information in the email signature, I took to the internet to accomplish my task.

#### GATHERING THE SMALL STUFF

Initially I wanted to gather as much accurate information as possible to lay out my puzzle pieces. I started with social media. After some short searching, I was able to accurately find her Facebook page which listed her full name including a middle (or maiden) name. This was very helpful in eliminating other false positives during my search. Through this profile, I was able to locate an unused Pinterest page of hers. This page yielded my next clue which was a username. The username followed the format of: first initial, last name, 5). For this example we’ll use jsmith5. So now that I had that, I began running searches for that username. That username only yielded one Google page of results and amongst them was a Yahoo email address. Now I’m making some solid progress.

#### ENUMERATING WITH RESETS

Now I knew a potential email address linked to her, as well as a username scheme she commonly uses and some social media profiles. This is where the password resets can become quite useful. Many major sites will only disclose redacted information on a password reset screen but they’ll do so WITHOUT actually initiating the reset and notifying the user. This includes Facebook. So, when I took to Facebook and told them I forgot my password and entered jsmith5@yahoo.com (still an example mind you), I was able to view this:
<p><img src="/assets/images/passosint1.png"></p>

Jackpot. Now I have the length of an alternate Gmail address that appears to match the same username convention I already know she uses and I know her phone number ends in 17. So, at this point, I can take to Gmail and begin starting a password reset process there to try and guess the email address. I begin guessing different naming patterns and repetitively meet an “account doesn't exist” error. I counted the characters in the redacted address I found on Facebook and realized it was one character longer than the other username I had found. At this point, I inserted the initial from the middle/maiden name on Facebook at various points and after a few more guesses, a password reset screen for a legitimate Gmail address came up. I have now identified and validated her Gmail address.

Now that I have 2 email addresses, I want to find that phone number and I will be easily able to identify it because I know it ends in 17. I did some Google dorking for that Gmail address and on the first page of the results I was shocked to come across this:
<p><img src="/assets/images/passosint2.png"></p>

#### MAKING CONTACT

At this point it’s about 6pm my time on the west coast and she’s out east so it’s reaching 9pm there. I want to make contact via phone but I don’t want to ring her personal cell phone late at night. Texting seemed too unprofessional, especially as I’m hoping to be a candidate for a position with her company. However, I didn’t want to let time go on and have her move on to someone else assuming I was ignoring her. I used a service called “SlyDial” which lets you dial into their line and then enter a phone number and be connected directly to their voicemail without ringing their phone. *Side note: SlyDial can be used to listen to a persons voicemail greeting and potentially gather more information from that as well.* I called and was met with her voicemail message. I left a message stating who I was and that I was still interested in the position and I believed my emails were not reaching her. Then I waited…

It’s fair to say at this point there was obviously some risk in dialing her personal cell phone, unsolicited. While I agree, I didn’t have much to lose at this point because at least this gave me a chance as opposed to her just never getting my emails and having this opportunity disappear. It also helped that the position desired OSINT experience and this was a great opportunity to showcase competence in that.

Not even 15 minutes later, I received an email from her. It read:
<blockquote>
Michael,
I just got your voicemail and as a result, checked my junk folder. Your replies and this email went to my junk folder so I did not know you replied.

In any event, thanks for calling because I would not have known otherwise. We can definitely set-up another time to speak.
Let me know what your schedule looks like tomorrow.
</blockquote>
A risk had been taken but in the end it turned out to have been well worth it. We spoke on the phone the very next day and then a day later I received a very surprising email notifying me I had been selected for my very first formal infosec position.

#### PRACTICAL PASSWORD RESET INFORMATION

Below I’ve included a running copy of various major online services and how they’ll respond to password reset requests. Some of them will notify the user immediately and should obviously be avoided as they’ll not only yield you no information but they’ll tip off the user.

```
Facebook: You will be met with a screen displaying alternative contact methods that can be used to reset the password as seen in the post above. It also accurately uses the number of asterisks that match the length of the email addresses.
Google: You will be asked to enter the last password remembered which can be anything you want and the next screen will display a redacted recovery phone number with the last 2 digits if one is on file.
Twitter: Entering a Twitter username will yield a redacted email address on file with the first 2 characters of the email username and the first letter of the email domain. It also accurately uses the number of asterisks that match the length of the email address.
Yahoo: Will display a redacted alternate email address if on file. Displays accurate character count as well as first character and last 2 characters of email username along with full domain.
Microsoft: Displays redacted phone number with last 2 digits.
Pinterest: Displays a users profile as well as a redacted email address without an accurate character count.
Instagram: Automatically initiates a reset and emails the user. Do not use.
LinkedIn: Automatically initiates a reset and emails the user. Do not use.
FourSquare: Automatically initiates a reset and emails the user. Do not use.
```
