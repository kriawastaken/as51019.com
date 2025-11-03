+++
title = "Quick intro to how the internet really works"
date = "2024-05-02T00:10:04Z"
author = "Kría Elínarbur"
authorTwitter = "kriawastaken" #do not include @
cover = "/posts/intro-internet/IRIS-link_NORDUnet.png"
tags = ["internet", "bgp", "routing"]
keywords = ["bgp", "internet", "routing", "submarine cables"]
description = ""
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
+++

Ever since I can remember I have always had some curiosity about the internet. It began with a desire to create websites. Then a desire to publish those websites. Then a desire to build apps out from those websites. And so on.

Eventually I was deep into application development, having started with PHP, and eventually settling in the JavaScript/TypeScript ecosystem. Publishing (hosting) those applications is what led me to learning about Linux, networking, and a bunch of other "sysadmin-y" stuff. This is where a new curiosity formed: **how does the internet work?**

# How does the internet work?

Whenever I asked this question - online or in real life - I never felt satisfied with the answer. Sometimes I'd receive more information than other times, but never did it go any deeper than merely "the internet is like a series of tubes". This isn't incorrect, those "tubes" would be subsea and landline fibre routes, but it isn't very helpful either. Someone who wants to dive deeper won't feel very satisfied, like how I wasn't, hearing the same information repeatedly. However, this is useful foundational knowledge.

When people repeat this phrase "the internet is like a series of tubes" they are referring to, as I mentioned, the subsea and landline fibre routes that carry internet traffic between places. But it's more complicated than that.

To develop an understanding of how this seemingly abstract network of cables forms the internet, it is best to scratch the idea of an "internet" at all. Rather, the internet is actually a bunch of smaller networks all connected together in one way or another. These are companies - big and small - ranging from large ISPs like France's Orange to tiny content networks like Panascais back to large content networks like Google.

These smaller networks are connected to each other via the fibre mentioned above. Sometimes they'll be directly connected together, usually because it saves money. Most of the time they'll be connected through another network called a "transit provider" or a "carrier". We call them that because traffic on their network isn't actually originating or terminating from or to them. Rather it's crossing ("transiting") their network.

But ok... back to the question I had as a kid... how does the internet *really* work? Like, how does my computer know the right cable to send packets through if I ping a Hetzner IP address???

# Routing

Turns out the answer to this question is actually hidden behind another question - what is and how does routing work?

To put it simply, routing is like driving a car. You may intuitively know the route from your house to your closest "multi-lane road"&trade;, but you might need some guidance getting to a beach you haven't been to before.

In this analogy, the way to the big wide multi-lane road might be like the way from your device to your home router, and from your router to your ISPs router. The secret here really are these boxes we call "routers". They are really just computers, but they - somehow - know how to receive traffic on one network port (like an ethernet port, that is) and send it out another.

To do this routers keep a special table in their system called a "routing table". It is a table of "routes". Routes are also called "prefixes" in networking spaces as they used to work on a prefix based system. If an IP address started with `38` you would know to send it to Cogent (not the waste management company). Today this works slightly differently (Google "CIDR" if interested) but the principle is the same and the old model is easier to explain so I'll stick to that.

Now, your home router has a very simple routing table. Likely it has a route for your home network, commonly prefixed with `192.168.x`, and a "default" route that sends all traffic out the port connected to your ISP. This is your "internet connection".

A core element of this system is specificity. Your router will prefer routes that are *more specific*. A route for the prefix `192.168.1 via port0` will take priority over a route for `192.168 via port1`, if the destination address begins with `192.168.1`. This way you can target traffic bound to addresses with a certain prefix but leave other traffic within the same "parent prefix" alone. This is how your home router can communicate with your device despite having a route forwarding all traffic to your ISP. 

Lastly, your ISP also has some routers. In fact everyone on the internet has some routers. Really the internet is closer to "just a bunch of routers" than "just a bunch of tubes". These routers similarly hold routing tables that tell traffic where to go.

So when you send traffic to that Hetzner IP address, your home router will send it to your ISPs router which will then send it to another router which will the... I think you get the idea. We can even see this semi-visually with tools like `traceroute` or `mtr`:

```
kjartan@fenrir:~$ mtr -4 -w hetzner.com
Start: 2024-05-02T00:52:02+0000
HOST: fenrir                             Loss%   Snt   Last   Avg StDev
  1.|-- _gateway                          0.0%    10    0.5   0.4   0.1
  2.|-- 85-220-0-1.dsl.dynamic.simnet.is  0.0%    10    1.5   2.5   0.8
  3.|-- siminn-linx-gw-1.isholf.is        0.0%    10   41.0  40.9   0.6
  4.|-- ???                              100.0    10    0.0   0.0   0.0
  5.|-- core6.par.hetzner.com             0.0%    10   55.4  55.4   0.8
  6.|-- core11.nbg1.hetzner.com           0.0%    10   57.4  57.1   1.3
  7.|-- ???                              100.0    10    0.0   0.0   0.0
```

Here I am "tracing the route" to `hetzner.com`. As you can see the first hop, labeled `1`, is my home router. Hop `2` is my ISPs router, so is `3` only in London this time. Hop `4` we don't know what is but hops `5` and `6` are Hetzner's. But... how does this then work??? There's no way Hetzner and my ISPs are good enough friends to be manually configuring routes to each other...

# Border Gateway Protocol

Border Gateway Protocol, BGP, is the glue of the internet backbone. It is how the networks we spent so long talking about actually do this "routing" between each other.

It's quite simple for what it does. Two routers, directly connected with each other, are configured to establish a "BGP session" between themselves. This just a fancy name for a TCP socket with some *very useful (critical)* safety features that make sure the other router ("neighbor") hasn't died or disconnected.

Through this socket these routers will send each other the routes they wish to "announce". When either router receives a route from the other it decides if it wants to accept it and insert it into its routing table or not. Famously, this has caused [massive problems](https://www.ripe.net/publications/news/youtube-hijacking-a-ripe-ncc-ris-case-study/) before.

What truly makes everything work, though, is the transit networks we discussed above. *Obviously* not every AS ("autonomous system" aka. "network") can connect with every other AS - it'd be impractical, expensive, and messy. So what our dear ISPs (and content networks too!) do is... buy *their* internet from... another ISP?

It sounds confusing and silly at first but in the context of just "exchanging routes" it makes a lot of sense. These "transit networks" (or "ISPs for ISPs") will do all the 'connecting with other networks' business *for us* and then basically 'rent the routes' they receive out to us. In technical terms they take the routes they accept from a peer and announce them to all other peers and customers.

Of course, the actual value of this service comes from the fact that these networks maintain high capacity, redundant connections between places on behalf of the smaller guys. The cost is paid up by the purchasers of the service (ISPs) and a profit is made from marking the price up slightly (or not so slightly, ~~shoutout AS3320~~).

It's also worth mentioning that BGP identifies networks by an "autonomous system number" ("ASN"). They start with `AS` followed by up to ten digits. Though these days we only ever see one to six digits. This is why this "blog" is called `AS51019`, it's my ASN.

Indeed, if we re-run our `mtr` from above with the `-z` flag we can see the ASNs each router belongs to:

```
kjartan@fenrir:~$ mtr -4 -zw hetzner.com
Start: 2024-05-02T01:03:49+0000
HOST: fenrir                                    Loss%   Snt   Last   Avg StDev
  1. AS???    _gateway                           0.0%    10    0.5   0.4   0.1
  2. AS6677   85-220-0-1.dsl.dynamic.simnet.is   0.0%    10    2.3   2.1   0.7
  3. AS???    siminn-linx-gw-1.isholf.is         0.0%    10   41.5  40.9   0.8
  4. AS???    ???                               100.0    10    0.0   0.0   0.0
  5. AS24940  core6.par.hetzner.com              0.0%    10   55.7  55.7   0.7
  6. AS24940  core11.nbg1.hetzner.com            0.0%    10   57.5  56.8   0.6
  7. AS???    ???                               100.0    10    0.0   0.0   0.0
```

`AS6677` is my current ISP; `AS24940` is Hetzner. `AS???` is, of course, simply an AS we can't identify (usually because *some* assholes block ICMP).

Similarly, we can see a transit network in a traceroute like this by tracing the route to something e.g. on another continent:

```
kjartan@fenrir:~$ mtr -4 -zw 64.187.208.1
Start: 2024-05-02T01:26:51+0000
HOST: fenrir                                       Loss%   Snt   Last   Avg StDev
  1. AS???    _gateway                              0.0%    10    0.4   0.4   0.1
  2. AS6677   85-220-0-1.dsl.dynamic.simnet.is      0.0%    10    2.1   2.4   0.8
  3. AS6677   hu0-0-40.db2.dub.ie.ip.siminn.is      0.0%    10   24.9  25.1   0.6
  4. AS2128   10ge16-1.core1.dub1.he.net           10.0%    10   25.4  25.1   0.9
  5. AS6939   port-channel4.core2.nyc5.he.net       0.0%    10  102.4 102.0   1.4
  6. AS6939   port-channel8.core2.nyc4.he.net      90.0%    10  105.3 105.3   0.0
  7. AS6939   port-channel18.core3.chi1.he.net     80.0%    10  121.1 122.1   1.4
  8. AS6939   port-channel1.core2.chi1.he.net      80.0%    10  117.2 116.5   1.1
  9. AS6939   port-channel18.core2.sea1.he.net     20.0%    10  153.7 153.5   2.4
 10. AS1299   doof-ic-375811.ip.twelve99-cust.net   0.0%    10  150.5 150.8   0.7
 11. AS47689  r1-sea-a.as47689.net                  0.0%    10  151.5 151.4   0.7
```

`AS6939`, Hurricane Electric of the USA, is a widely known transit network. But oh look! There's another! `AS1299`, Arelion of Sweden, sits between `AS6939` and `AS47689` (shoutout Wes!). Arelion happens to be the 2nd-largest transit network in the world as measured by CAIDA AS Rank.

# In summary

* The internet is "just a bunch of tubes" between "just a bunch of routers".
* Routers are computers that tell traffic entering through one port to exit through another, based on routing tables.
* Routing tables are tables of routes. Routes are "prefixes" to IP addresses. More specific routes take preference over less specific ones.
* Your home router has a static routing table. It sends all traffic to your ISP.
* ISPs use Border Gateway Protocol to exchange routing information.
* When ISPs can't connect to a network on their own they rely on another ISP. An ISP for your ISP. ISPception. They're called "transit networks" or "carriers" because they "transit" or "carry" traffic across their network.

# Further reading

* [Peering](https://en.wikipedia.org/wiki/Peering)
* [Internet exchange points](https://en.wikipedia.org/wiki/Internet_exchange_point)
* [Autonomous systems](https://en.wikipedia.org/wiki/Autonomous_system_(Internet))
* [IP addresses](https://en.wikipedia.org/wiki/IP_address)
* [IP routing](https://en.wikipedia.org/wiki/IP_routing)
* [Tier 1 networks](https://en.wikipedia.org/wiki/Tier_1_network)

# Other resources

* [Explore the internet ecosystem](https://bgp.tools/)
* ["Networking: IPv6" Discord server](https://discord.gg/ipv6)
* ["the nettwork" Discord server](https://discord.gg/Q3tZVbJYXH)

# Questions/comments

If you'd like to ask me something - perhaps inquiries related to the post rather than tech support on BGP etc. ;) - or have a comment I'd be happy chat via [kjartann@kjartann.is](mailto:kjartann@kjartann.is).

<img src="/owiehappy.gif" alt=":owiehappy: emoji" width="48" />
