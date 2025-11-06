---
layout: post
title:  "Dissecting a Crypto Theft Operation: When JavaScript Obfuscation Hides Wallet Address Manipulation"
author: Michael
tags: [ Forensics, Malware, JavaScript, Cryptocurrency ]
featured: false
permalink: /manler-cc-crypto-theft/
hidden: false
cover-img: /assets/img/manler-analysis.png
thumbnail-img: /assets/img/manler-analysis.png
share-img: /assets/img/manler-analysis.png
---

This post is part technical deep-dive, part forensic investigation story. I'll walk through how I discovered and analyzed a sophisticated cryptocurrency theft operation that uses advanced JavaScript obfuscation to steal funds by manipulating withdrawal forms. There are tools out there for this kind of analysis, but I built my own set of Python scripts to really understand what was happening under the hood. This isn't a step-by-step tutorial—it's more about the process of investigation, the patterns I found, and what they reveal about how these attacks work.

#### THE INITIAL TIP

Someone reached out asking me to take a look at `https://manler.cc/payouts/account/`. They said something felt off about the site—it was supposed to be a Bitcoin mining payout page, but the behavior seemed suspicious. I'm always curious about these kinds of reports, so I fired up a browser and took a look.

At first glance, it looked legitimate. Clean interface, professional design, a withdrawal form asking for wallet addresses. Nothing immediately screamed "scam" to me. But when you've been doing this long enough, you develop a sense for when something doesn't add up. The site was asking users to enter their wallet addresses to withdraw funds from their "mining earnings." That's a common enough pattern, but something about it felt engineered.

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
- Make blockchain analysis harder

But I still needed to understand what the code was actually *doing*.

#### UNPACKING THE OBFUSCATION

That 3.3MB script was heavily obfuscated. I built a forensic analysis tool to systematically detect obfuscation patterns. Here's what I found:

**907 instances** of `Function()` constructor calls. This is a classic obfuscation technique—instead of writing functions normally, the code generates them dynamically at runtime. It makes static analysis nearly impossible because you can't see what the function does until it executes.

**58 instances** of Unicode encoding (`\u002F` instead of `/`).  
**36 instances** of hex encoding (`\x2F` instead of `/`).  
**5 instances** of `String.fromCharCode()` for character conversion.  
**1 instance** of `eval()` for direct code execution.

The entire script was minified into a single line with variable names shortened to 1-2 characters. This is multi-layered obfuscation designed to make analysis as difficult as possible.

But obfuscation doesn't make code safe—it just makes it harder to analyze. I kept digging.

#### FINDING THE MALICIOUS PATTERNS

I wrote another script to search for specific malicious patterns. This is where it got interesting:

**26 form interception patterns** detected. The code was using `addEventListener('submit')` to catch form submissions before they were sent to the server. Then it was calling `preventDefault()` to stop the normal submission process.

**147 value manipulation operations** found. This was the smoking gun. The code was reading input field values and modifying them. Specifically, it was targeting wallet address fields.

**119 data exfiltration patterns**. The code was packaging up form data and sending it somewhere via XMLHttpRequest or fetch API.

**713 DOM manipulation operations**. The code was actively hiding and showing elements, probably to make the site look legitimate while hiding its malicious behavior.

Let me reconstruct what's happening here. When a user enters their wallet address and clicks withdraw:

```javascript
// Pseudo-code of what the malicious code does
element.addEventListener('submit', function(e) {
    e.preventDefault(); // Stop normal form submission
    
    const walletInput = document.getElementById('wallet-address');
    const userAddress = walletInput.value; // Get user's address
    const attackerAddress = 'ATTACKER_WALLET_ADDRESS'; // Attacker's address
    walletInput.value = attackerAddress; // Replace it
    
    // Package and send the modified form
    const formData = new FormData();
    formData.append('wallet', attackerAddress);
    formData.append('amount', userAmount);
    
    fetch('/pay.php?p=53', {
        method: 'POST',
        body: formData
    });
});
```

The user thinks they're withdrawing to their own wallet, but the code silently replaces their address with the attacker's address before submission. The funds go to the attacker instead.

#### THE COMPLETE ATTACK FLOW

Here's the step-by-step of how this attack works:

1. User visits the site, sees a professional-looking Bitcoin mining payout interface
2. User enters their wallet address in the withdrawal form
3. User clicks "Withdraw"
4. JavaScript intercepts the form submission (`addEventListener('submit')`)
5. Code stops normal submission (`preventDefault()`)
6. Code reads the user's wallet address from the input field
7. Code replaces it with the attacker's wallet address
8. Modified form data is packaged and sent to `/pay.php`
9. Server processes the payment to the attacker's wallet
10. User never receives their funds

The entire attack happens invisibly in the background. The user sees a normal form submission, but their wallet address has been silently swapped out.

#### THE PAYMENT ENDPOINTS

Those thirteen payment endpoints I found earlier? Each one likely maps to a different attacker wallet address stored server-side. The parameter `p` (like `p=53` or `p=y264`) probably tells the server which wallet to use.

I tried to find the actual wallet addresses in the client-side code, but they weren't there. The attackers are smart—they store the wallet addresses server-side in the `/pay.php` script or a database. This means:

- They can change addresses without updating client code
- Addresses aren't visible in browser dev tools
- Different payment methods can use different wallets
- Makes detection and blocking harder

To find the actual wallet addresses, you'd need to either:
- Intercept network traffic when the form is submitted
- Analyze blockchain transactions from known victims
- Get access to the server-side code (unlikely)

The most practical approach is blockchain analysis—track where victim funds actually go, and that reveals the attacker wallets.

#### WHY THIS IS DANGEROUS

This attack is particularly effective because:

1. **It's invisible to users** - The address replacement happens in JavaScript before the form is submitted. Users have no way to know their address was changed.

2. **Sophisticated obfuscation** - With 1,007+ obfuscation techniques across 3.3MB of code, even security tools struggle to detect it. Manual analysis is extremely time-consuming.

3. **Multiple attack vectors** - Thirteen different payment endpoints mean the attackers can track which methods work best, distribute funds across wallets, and make analysis harder.

4. **No recovery** - Cryptocurrency transactions are irreversible. Once funds are sent, they're gone.

5. **Legitimate appearance** - The site looks professional. Without technical analysis, users have no reason to suspect it's malicious.

#### WHAT I LEARNED

This investigation reinforced a few things for me:

**Obfuscation is a red flag, not protection.** When you see heavily obfuscated JavaScript, especially on financial sites, that's a warning sign. Legitimate sites don't need to hide their code this aggressively.

**Form interception is a common attack vector.** The pattern of intercepting form submissions to modify data before sending is something I've seen before, but the sophistication here was notable.

**Server-side storage makes detection harder.** By storing wallet addresses on the server instead of in client code, the attackers made their operation more flexible and harder to analyze.

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
- Form submission interception
- Wallet address field manipulation  
- Multiple payment endpoint configurations
- Heavy JavaScript obfuscation (1,007+ techniques)
- DOM manipulation to hide UI elements

**Code Patterns:**
- 147 value manipulation operations
- 26 form interception patterns
- 907 Function() constructor calls
- Multiple obfuscation techniques

#### FINAL THOUGHTS

This was a well-engineered attack. The combination of obfuscation, form interception, and server-side address storage creates a theft mechanism that's difficult to detect and analyze. But it's not perfect—the patterns are still there if you know what to look for.

For users, the takeaway is simple: be extremely cautious when entering wallet addresses on any site. Verify the site's legitimacy, double-check addresses before confirming transactions, and use browser extensions that can detect address manipulation.

For security researchers, this case highlights the importance of pattern detection even when code is heavily obfuscated. The malicious behavior leaves traces that can be identified through systematic analysis.

The site should be blocked and reported to appropriate authorities. If you've been a victim of this or similar scams, report it to your local cybercrime unit and provide transaction hashes for blockchain analysis.

All the analysis tools and scripts I built for this investigation are available in my investigation repository for anyone who wants to reproduce the analysis or adapt the techniques for their own research.
