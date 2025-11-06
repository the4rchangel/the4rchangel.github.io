---
layout: post
title:  "Dissecting a Crypto Theft Operation: When JavaScript Obfuscation Hides a Fee-Based Scam"
author: Michael
tags: [ Forensics, Malware, JavaScript, Cryptocurrency ]
featured: false
permalink: /manler-cc-crypto-theft/
hidden: false
cover-img: /assets/img/manler-analysis.png
thumbnail-img: /assets/img/manler-analysis.png
share-img: /assets/img/manler-analysis.png
---

This post is part technical deep-dive, part forensic investigation story. I'll walk through how I discovered and analyzed a sophisticated cryptocurrency theft operation that uses advanced JavaScript obfuscation to hide a fee-based scam. The site claimed users needed to pay a withdrawal fee to access their "mining earnings," provided instructions on how to buy Bitcoin through legitimate exchanges, and generated unique one-time wallet addresses for each victim. There are tools out there for this kind of analysis, but I built my own set of Python scripts to really understand what was happening under the hood. This isn't a step-by-step tutorial, it's more about the process of investigation, the patterns I found, and what they reveal about how these attacks work.

#### THE SPAM EMAIL

I received a spam email claiming I had an inactive Bitcoin mining account with a significant balance that needed to be withdrawn within 24 hours or it would be forfeited. Classic phishing tactics: urgency, a large sum of money, and a threat of loss. The email was flagged as suspicious by my email client, but even without that warning, the red flags were obvious, the domain, the tone, the entire premise.

Most people would delete it and move on. But I'm curious about how these scams actually work under the hood. So instead of deleting it, I clicked through to see what the site was doing. The email led to `https://manler.cc/payouts/account/`, which presented itself as a Bitcoin mining payout page.

At first glance, it looked legitimate. Clean interface, professional design. But I already knew it was malicious, so I wasn't looking for whether it *seemed* legitimate, I was looking for *how* it was stealing funds. The site claimed users had accumulated Bitcoin mining earnings and needed to pay a withdrawal fee to access them. It displayed a payment page showing the required fee amount, provided helpful instructions on how to buy Bitcoin through legitimate exchanges like MoonPay, Coingate, Bitcoin.com, and TrustWallet, and generated a unique one-time Bitcoin wallet address for each user to send the fee to. That's a sophisticated approach, using legitimate services to build trust while funneling payments to attacker-controlled wallets.

I decided to dig deeper.

#### STARTING THE INVESTIGATION

My first step was to fetch the site and see what we were actually dealing with. I wrote a quick Python script using `requests` and `BeautifulSoup` to pull down the HTML and extract all the JavaScript. This is usually where the interesting stuff lives.

```python
import requests
from bs4 import BeautifulSoup

url = "https://manler.cc/payouts/account/"
response = requests.get(url)
soup = BeautifulSoup(response.text, 'html.parser')
```

What I found was interesting. There were two main scripts:

1. An inline script in the HTML that set up some configuration
2. An external script at `/_nuxt/entry.4e713294.js` that was **3.3 megabytes**

That external script size immediately raised a red flag. Most legitimate sites don't ship 3.3MB of JavaScript for a simple payout page. I downloaded it and opened it in a text editor. It was a single line. No breaks, no whitespace, just one massive line of minified code. This was clearly obfuscated.

#### THE CONFIGURATION CLUE

Before diving into that massive obfuscated script, I looked at the inline configuration. This is what caught my attention:

```javascript
window.__NUXT__ = (function(a) {
    return {
        serverRendered: false,
        config: {
            public: {
                payExchange: "\u002Fpay.php?p=53",
                payExchangeFee: 64,
                payCommissionfp: "\u002Fpay.php?p=63",
                payCommissionfpFee: 56,
                payCommissionsp: "\u002Fpay.php?p=y264",
                payCommissionspFee: 48,
                // ... and 10 more similar entries
            }
        }
    }
}(""))
```

Thirteen different payment endpoints, all pointing to `/pay.php` with different parameters. Each one had an associated "fee" ranging from 48 to 268. That's not normal. A legitimate site would have maybe one or two payment processors, not thirteen different endpoints with varying fees.

This told me the attackers were running multiple payment methods, probably to:
- Track which methods users prefer
- Distribute stolen funds across multiple wallets
- Generate unique wallet addresses per transaction to make tracking harder
- Make blockchain analysis more difficult

But I still needed to understand what the code was actually *doing*.

#### UNPACKING THE OBFUSCATION

That 3.3MB script was heavily obfuscated. I built a forensic analysis tool to systematically detect obfuscation patterns. Here's what I found:

**907 instances** of `Function()` constructor calls. This is a classic obfuscation technique, instead of writing functions normally, the code generates them dynamically at runtime. It makes static analysis nearly impossible because you can't see what the function does until it executes.

**58 instances** of Unicode encoding (`\u002F` instead of `/`).  
**36 instances** of hex encoding (`\x2F` instead of `/`).  
**5 instances** of `String.fromCharCode()` for character conversion.  
**1 instance** of `eval()` for direct code execution.

The entire script was minified into a single line with variable names shortened to 1-2 characters. This is multi-layered obfuscation designed to make analysis as difficult as possible.

But obfuscation doesn't make code safe, it just makes it harder to analyze. I kept digging.

#### FINDING THE MALICIOUS PATTERNS

I wrote another script to search for specific malicious patterns. This is where it got interesting:

**26 form interception patterns** detected. The code was using `addEventListener('submit')` to catch form submissions and handle payment processing.

**147 value manipulation operations** found. The code was reading and modifying form values, likely handling the payment flow and wallet address generation.

**119 data exfiltration patterns**. The code was packaging up form data and sending it to the payment endpoints via XMLHttpRequest or fetch API.

**713 DOM manipulation operations**. The code was actively hiding and showing elements, probably to display the payment interface, show the unique wallet address, and manage the countdown timer for payment expiration.

The attack works like this: when a user visits the site, the JavaScript communicates with the server to generate a unique one-time Bitcoin wallet address. The site displays this address along with the required fee amount and a countdown timer. The code also includes instructions on how to buy Bitcoin through legitimate exchanges, which builds trust and makes the scam seem more legitimate. When a user sends Bitcoin to the generated address, the funds go directly to the attacker's wallet. The user never receives their "mining earnings" because those earnings never existed in the first place.

#### THE COMPLETE ATTACK FLOW

Here's the step-by-step of how this attack works:

1. User receives spam email claiming they have inactive Bitcoin mining earnings
2. User clicks through to `https://manler.cc/payouts/account/`
3. Site displays a professional-looking payment interface showing a required withdrawal fee
4. JavaScript generates a unique one-time Bitcoin wallet address for this specific user/session
5. Site displays the wallet address, fee amount, and a countdown timer creating urgency
6. Site provides helpful instructions on how to buy Bitcoin through legitimate exchanges (MoonPay, Coingate, Bitcoin.com, TrustWallet) to build trust
7. User follows instructions, buys Bitcoin from a legitimate exchange
8. User sends Bitcoin to the unique wallet address displayed on the site
9. Funds go directly to the attacker's wallet
10. User never receives their "mining earnings" because they never existed

The entire scam relies on social engineering: the promise of free money, the urgency of a countdown timer, and the legitimacy of linking to real Bitcoin exchanges. The unique wallet address per user makes it harder to track and block the operation.

#### THE PAYMENT ENDPOINTS

Those thirteen payment endpoints I found earlier? Each one likely maps to a different payment method or fee structure. The parameter `p` (like `p=53` or `p=y264`) probably tells the server which payment configuration to use, and the associated fee values (ranging from 48 to 268) determine how much Bitcoin the user is asked to send.

I tried to find the actual wallet addresses in the client-side code, but they weren't there. The attackers are smart, they generate wallet addresses dynamically server-side in the `/pay.php` script. This means:

- Each user gets a unique wallet address, making tracking harder
- Addresses aren't visible in browser dev tools until generated
- Different payment methods can use different wallet pools
- They can rotate addresses without updating client code
- Makes detection and blocking significantly harder

To find the actual wallet addresses, you'd need to either:
- Intercept network traffic when the payment page loads and the wallet is generated
- Analyze blockchain transactions from known victims
- Get access to the server-side code (unlikely)

The most practical approach is blockchain analysis, track where victim funds actually go, and that reveals the attacker wallet addresses. However, with unique addresses per transaction, this becomes more challenging.

#### WHY THIS IS DANGEROUS

This attack is particularly effective because:

1. **Social engineering** - The combination of promised earnings, urgency (countdown timer), and helpful instructions on buying Bitcoin creates a convincing narrative. Users think they're paying a legitimate fee to access their money.

2. **Legitimate exchange links** - By linking to real Bitcoin exchanges (MoonPay, Coingate, Bitcoin.com, TrustWallet), the site builds trust. Users see legitimate services and assume the entire operation is legitimate.

3. **Unique wallet addresses** - Each user gets a unique one-time wallet address, making it harder to track, block, or identify the operation through blockchain analysis.

4. **Sophisticated obfuscation** - With 1,007+ obfuscation techniques across 3.3MB of code, even security tools struggle to detect it. Manual analysis is extremely time-consuming.

5. **Multiple payment methods** - Thirteen different payment endpoints with varying fees mean the attackers can track which methods work best, distribute funds across wallets, and make analysis harder.

6. **No recovery** - Cryptocurrency transactions are irreversible. Once funds are sent, they're gone.

7. **Professional appearance** - The site looks legitimate and professional. Without technical analysis, users have no reason to suspect it's malicious.

#### WHAT I LEARNED

This investigation reinforced a few things for me:

**Obfuscation is a red flag, not protection.** When you see heavily obfuscated JavaScript, especially on financial sites, that's a warning sign. Legitimate sites don't need to hide their code this aggressively.

**Social engineering amplifies technical attacks.** The combination of obfuscated code with legitimate exchange links and helpful instructions makes the scam much more convincing than pure technical trickery.

**Dynamic wallet generation makes tracking harder.** By generating unique wallet addresses server-side for each user, the attackers made their operation more flexible and significantly harder to track or block.

**Pattern detection works.** Even with heavy obfuscation, searching for specific patterns (form interception, value manipulation, data exfiltration) can reveal malicious behavior.

#### PRACTICAL DETECTION TECHNIQUES

Below I've included some of the key patterns and techniques I used to identify this malicious behavior. These can be applied to similar investigations:

**Form Interception Detection:**
```python
# Look for these patterns in JavaScript
patterns = [
    r'addEventListener\s*\(\s*["\']submit["\']',  # Submit event listener
    r'preventDefault\s*\(',  # Preventing default form submission
    r'\.submit\s*\(',  # Programmatic form submission
]
```

**Value Manipulation Detection:**
```python
# Look for input field manipulation
patterns = [
    r'\.value\s*=',  # Direct value assignment
    r'getElementById.*value',  # Getting element by ID and modifying value
    r'querySelector.*value',  # Query selector with value modification
]
```

**Obfuscation Indicators:**
```python
# Common obfuscation techniques
obfuscation_patterns = [
    r'Function\s*\(',  # Function constructor (dynamic code generation)
    r'eval\s*\(',  # Eval usage
    r'atob\s*\(|btoa\s*\(',  # Base64 encoding/decoding
    r'String\.fromCharCode',  # Character code conversion
    r'\\x[0-9a-f]{2}',  # Hex encoding
    r'\\u[0-9a-f]{4}',  # Unicode encoding
]
```

**Payment Endpoint Analysis:**
When you see multiple payment endpoints pointing to the same script with different parameters, that's suspicious. Each parameter likely maps to a different wallet or payment method.

**Network Request Monitoring:**
Intercepting form submissions and monitoring where data is sent can reveal the attack. Tools like Burp Suite or OWASP ZAP are useful here.

**Blockchain Analysis:**
If you can identify victim transactions, tracking where the funds go reveals the attacker wallets. This is often the most practical way to identify the actual addresses being used.

#### INDICATORS OF COMPROMISE

For anyone investigating similar sites, here are the key indicators I found:

**URLs:**
- `https://manler.cc/payouts/account/` - Main malicious page
- `https://manler.cc/pay.php?p=*` - Payment endpoints (13 variants)
- `https://manler.cc/_nuxt/entry.4e713294.js` - Obfuscated script

**Behavioral Indicators:**
- Fee-based withdrawal requirement
- Unique wallet address generation per user
- Links to legitimate Bitcoin exchanges (MoonPay, Coingate, Bitcoin.com, TrustWallet)
- Countdown timers creating urgency
- Multiple payment endpoint configurations
- Heavy JavaScript obfuscation (1,007+ techniques)
- DOM manipulation to manage payment interface

**Code Patterns:**
- 147 value manipulation operations
- 26 form interception patterns
- 907 Function() constructor calls
- Multiple obfuscation techniques

#### FINAL THOUGHTS

This was a well-engineered attack. The combination of obfuscation, social engineering, legitimate exchange links, and dynamic wallet generation creates a theft mechanism that's difficult to detect and analyze. But it's not perfect, the patterns are still there if you know what to look for.

For users, the takeaway is simple: if a site claims you have money you never earned and asks you to pay a fee to access it, that's a scam. Legitimate services don't require you to pay fees to withdraw your own money. Be extremely skeptical of any site that generates unique wallet addresses for "fees" or "withdrawals," especially when combined with urgency tactics like countdown timers.

For security researchers, this case highlights the importance of pattern detection even when code is heavily obfuscated. The malicious behavior leaves traces that can be identified through systematic analysis.

The site should be blocked and reported to appropriate authorities. If you've been a victim of this or similar scams, report it to your local cybercrime unit and provide transaction hashes for blockchain analysis.

All the analysis tools and scripts I built for this investigation are available in my investigation repository for anyone who wants to reproduce the analysis or adapt the techniques for their own research.
