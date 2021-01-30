---
layout: post
title:  "Facebook | Uncovering Seller Info Without Login"
author: Michael
categories: [ OSINT, Facebook ]
image: assets/img/marketplace1.png
featured: false
hidden: false
#subtitle: Excerpt from Soulshaping by Jeff Brown
#cover-img: /assets/img/path.jpg
#thumbnail-img: /assets/img/thumb.png
#share-img: /assets/img/path.jpg
#tags: [books, test]hidden: false
---
<center><p><img src="/assets/img/marketplace1.png" height="200" width="400"></p></center>

<p>I was doing some research this week on <a href="https://www.facebook.com/marketplace/" target="_blank">Facebook Marketplace</a> looking for some... unethically acquired and resold items and the sellers behind them. However, my research was limited to only acquiring data without being logged into a Facebook account. Finding the sellers behind the items becomes more troublesome in this scenario as Facebook doesn't display the seller information on Marketplace posts if you are not logged in.</p>

<p>Now, checking the source of a page for hidden information can be daunting for many. Especially on a site like Facebook where there is a sizable amount of code making things work on the page you're viewing. But nonetheless, this is where our journey takes us. For those of you who are unaware, you can view the source code behind any web page by just right clicking it and clicking "View Source" (or some varient of that) from the right click menu. This will spawn another tab with most of the code that's rendering the page you're looking at.</p>

### WORKING BACKWARDS

<p>So admittedly I cheated on this task. I had no idea what I was looking for and I thought if the info was there, it'd be better to work backwards. I logged into Facebook and viewed a post of interest to find the user's profile name. I then logged out, viewed the same listing and searched the source. This led me to a javascript block where Facebook was tracking the poster info still even though they weren't displaying it. I determined from what I was viewing that you can search for the string <code>typename:"User",id:</code> to find the seller's profile ID, name, and sometimes the URL to their profile picture in the source code. So if you open any marketplace listing, while not logged in, you can open the source, search for that string and get the results below:</p>
<center><p><img src="/assets/img/marketplace2.png"></p></center>
<center><p><img src="/assets/img/marketplace3.png"></p></center>

<p>The user ID also doubles at the url to the seller's profile. Even if they have a custom link like facebook.com/coolseller1, the user ID will forward to that. So now with a little source code magic you can identify the seller behind any Facebook Marketplace post without needing to have a profile on the site or logging in to it. Happy 2020!</p>
