---
layout: post
title: Fixing slow wifi on my Arch install
categories: [Miscellaneous]
---

Our wifi at work was slow (~16 Mbit/s), so I ordered us a new modem. Lo and behold, our speed went almost up to 120...on my colleagues computer. I was still stuck at 16. What gives?

I instantly thought "firmware", but I really wasn't sure what could be wrong. `dmesg | grep firmware` revealed this:

` [    3.087847] platform regulatory.0: Direct firmware load for regulatory.db failed with error -2`

That seems wrong. A quick search of the [Arch Wiki](https://wiki.archlinux.org/index.php/Wireless_network_configuration#Respecting_the_regulatory_domain) revealed that I had the wrong regulatory domain set. I followed the instructions on the wiki to set my regdomain as US.

~~~~
$ yay crda
$ reboot # and log back in
$ dmesg | grep cfg80211
~~~~

I don't precisely know what the output of that `dmesg` is supposed to be, but I got:
~~~~
[    2.110639] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[    2.135368] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
~~~~

I continued with the wiki:
~~~~
$ iw reg set US
$ iw reg get # to check that country is set to "US"

This helped, my download speed went up to about 20 Mbit/s. But this is still nowhere near 120! Something else was wrong.

I googled around a little more, and found a [suggestion](https://wiki.archlinux.org/index.php/Wireless_network_configuration#iwlwifi) on the Wiki (the answer is always the wiki!).

>If you have problems connecting to networks in general or your link quality is very poor, try to disable 802.11n, and perhaps also enable software encryption:
>`/etc/modprobe.d/iwlwifi.conf`
>`options iwlwifi 11n_disable=1 swcrypto=1`
>
>If you have a problem with slow uplink speed in 802.11n mode, for example 20Mbps, try to enable antenna aggregation:
>`/etc/modprobe.d/iwlwifi.conf`
>`options iwlwifi 11n_disable=8`

So I set my /etc/modprobe/d/iwlwifi.conf to be:
`options iwlwifi 11n_disable=8 swcrypto=1 lin_disable=8`

Bam! 114 Mbit/s. There we go!
