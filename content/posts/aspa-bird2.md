+++
title = "ASPA path validation in BIRD2 with Routinator and TypeScript"
date = "2024-05-02T23:27:27Z"
author = "Kría Elínarbur"
authorTwitter = "kriawastaken" #do not include @
cover = ""
tags = ["routing security", "rpki", "aspa"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

Recently (or not so recently?) some RFCs were published on the topic of a new standard in routing security. This is the ASPA (Autonomous System Provider Authorization) Objects system.

Even more recently [the first route leak prevented by ASPA](https://manrs.org/2023/02/unpacking-the-first-route-leak-prevented-by-aspa/) ocurred, in february 2023.

I thought it might be fun to explore if we could achieve the same thing, only with BIRD2, some open source tools, and a bit of TypeScript. So let's dive in!

# rpki-client

Initially my intuition for this was to use `rpki-client` as I had seen this tool used by Job Snijders and other big names in the "routing security space".

The maintainers of `rpki-client` also happened to publish [a website](https://console.rpki-client.org/) showing dumps given by `rpki-client` at certain intervals in a web-accessible form. [There even is an ASPA page!](https://console.rpki-client.org/aspa.html). Cool!

Unfortunately I only ran into issues with `rpki-client`. It'd do funky stuff like not being able to connect to most repos and run out of memory, for whatever reason.

So I changed course.

# Routinator

When I was doing this I happened to be in a call in the [Network: IPv6 Discord](https://discord.gg/ipv6) where Lee, a comrade from the networking space, pointed out that NLnet's Routinator could do ASPA. Indeed, [it can](https://routinator.docs.nlnetlabs.nl/en/stable/advanced-features.html#aspa) though it requires some modification. 

To do this we have to follow [Routinator's guide for compiling from source](https://routinator.docs.nlnetlabs.nl/en/stable/building.html). This is so we can build the software with the feature flag that enables ASPA.

Luckily this is pretty simple and boils down to running the correct command after installing rust on your system:

```bash
# some build tools
apt install build-essential

# install rust (please dont pipe to bash in production)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# add cargo to $PATH (note the period, it's important)
. "$HOME/.cargo/env"

# build routinator with aspa
cargo install --locked --features aspa routinator
```

Now we can run `routinator` with the `--enable-aspa` flag. Cool!

# Getting current ASPA data

Now, to do anything useful here we have to retrieve all current ASPA ASAs (I think these are like certificates?) from the RPKI publication servers around the world.

Unfortunately, since this proposed standard is so new (and not even accepted yet - for the record) there exists no TCP transport mechanism (and certainly no implementation for it in BIRD2) to import ASPA data into a routing daemon. Instead we'll have to move the data ourselves and make it work with BIRD2 ourselves.

Routinator makes this *really easy* and lets us output to JSON, just what we need to do some scripting with Deno later:

```bash
routinator --enable-aspa vrps -f json -o dump.json --no-route-origins --no-router-keys
```

What's happening here is quite simple. We run `routinator` with the `--enable-aspa` flag to use ASPA features. Then we run the `vrps` subcommand and pass the following options to it:

* `-f json` - output to JSON format.
* `-o dump.json` - write the output to a new file called `dump.json`.
* `--no-route-origins` exclude ROAs..
* `--no-router-keys` exclude router keys.

This command does take a while to run, so it probably fits in nicely as a cronjob. But once it finishes we get a file that looks [something like this](/posts/aspa-bird2/dump.json).

Now we just have to use it.

# Writing a filter function for BIRD2

I'll be the first to admit to being no BIRD expert but I do have a few tricks up my sleeve. AS path filtering is one of them.

The filter (or I guess conditional) that makes what I want to achieve possible is the following:

```
bgp_path ~ [= * ASN * =]
```

In summary it will return true if the specified ASN exists within the AS path. You can also pass multiple ASNs to it and check for order:

```
bgp_path ~ [= * ASN1 ASN2 * =] )
```

In this example, `ASN1` would be `ASN2`'s permitted carrier ("provider" in ASPA speak).

Using these two conditions we can quite easily create a functional, but rather sloppy, filter function for published ASAs.

The pseudocode looks a little like this:

```java
boolean is aspa valid() {
    if (asa publisher asn in path) {
        if (provider1 asn before asa publisher asn in path) return true;
        if (provider2 asn before asa publisher asn in path) return true;
        if (providerN asn before asa publisher asn in path) return true;

        return false; 
    }

    return true;
}
```

I'm sure this code could be improved for performance and efficiency - my online mates already seem to think so - but this will work for our purposes for now.

# Generating the filter function

I chose to write [a TypeScript program](https://github.com/kriawastaken/routinator-aspa-json-to-bird2) with Deno to achieve what I'd like to do here. Deno also conveniently allows you to "compile" (really it's just bundling) the program to a binary that can run portably, which is handy. Though the binary does end up being around 130MB in size. Yikes!

The program's 100~ lines come mostly from error handling and CLI flag boilerplate. The actual logic is 14 lines consisting of a for loop with a nested one inside it:

```typescript
let txt = "";
for (const {customer, providers} of aspas) {
    const asn = customer.replace(LEADING_AS, '');

    txt += `   # does the AS path include ${customer}?\n`
    txt += `   if (bgp_path ~ [= * ${asn} * =]) then {\n`;
    txt += `       # does the AS path include [carrier's asn, ${customer}]?\n`
    for (const provider of providers) {
        const carrier = provider.replace(LEADING_AS, '');
        
        txt += `       if (bgp_path ~ [= * ${carrier} ${asn} * =]) then return true;\n`;
    }
    txt += '       return false;\n';
    txt += '   }\n\n'
}
```

There are a few more lines of boilerplate but you get the gist. The resulting function definition you get from running the program looks a little like this:

```rust
function is_aspa_valid () {
   # does the AS path include AS945?
   if (bgp_path ~ [= * 945 * =]) then {
       # does the AS path include [carrier's asn, AS945]?
       if (bgp_path ~ [= * 174 945 * =]) then return true;
       if (bgp_path ~ [= * 1299 945 * =]) then return true;
       if (bgp_path ~ [= * 3491 945 * =]) then return true;
       if (bgp_path ~ [= * 6461 945 * =]) then return true;
       if (bgp_path ~ [= * 6939 945 * =]) then return true;
       if (bgp_path ~ [= * 7018 945 * =]) then return true;
       if (bgp_path ~ [= * 7922 945 * =]) then return true;
       if (bgp_path ~ [= * 9002 945 * =]) then return true;
       if (bgp_path ~ [= * 32097 945 * =]) then return true;

       return false;
   }
   ...

   # to avoid breaking stuff, assume the path is valid if no ASA exists.
   return true;
}
```

With that, using the generated function is as easy as shoving it in a `/etc/bird/functions/aspa.conf` file, including it in `bird.conf` and using it for filtering!

Using the function should look a little like this:

```conf
if (!is_aspa_valid()) then reject "aspa: not ok";
```

Typing `birdc c` to reconfigure BIRD is a little underwhelming, though. Barely anyone publishes ASAs currently and the few that are won't be likely to push out ASPA invalid routes any time soon. Once in a while we might find a route leak prevented by this system but it won't be much for a *while*.

However, to have some fun, we can just poke at the ASPA function and comment out a few key lines. For example: during testing I removed the line permitting Hurricane Electric, AS6939, to carry AS945's routes. The result was successful in that it removed any routes from AS945 with AS6939 in the AS path from BIRDs version of the "RIB".

# Closing

I think this was a fun evening project. It didn't take too long and was pretty satisfying. It also taught me a bit about ASPA that I didn't know before.

If you'd like to do this for yourself I'd like to reiterate that ASPA is an incredibly new and unapproved standard that will undoubtedly change. I wouldn't do this in production quite yet. Regardless I [published my program on GitHub](https://github.com/kriawastaken/routinator-aspa-json-to-bird2) and I encourage you, if you have any improvements, to make a pull request <img src="/owiehappy.png" alt=":owiehappy:" height="24" style="display:inline-block;margin-bottom:-8px;margin-left:-3px;" />

# Some references

* [rpki-client console](https://console.rpki-client.org/)
* [Routinator](https://routinator.docs.nlnetlabs.nl/en/stable/)

# Questions/comments

If you'd like to ask me something or have a comment I'd be happy chat via [kria@kria.tel](mailto:kria@kria.tel).

<img src="/owiehappy.gif" alt=":owiehappy: emoji" width="48" />
