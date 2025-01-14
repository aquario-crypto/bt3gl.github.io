Title:  CSAW CTF 2014 - Networking 100: "Big Data"
Date: 2014-09-22 8:20
Category: Networking
Tags: CTF, CSAW, Wireshark, Chaosreader





This is the only networking problem, and it is only 100 points, so it turned out to be very easy.

The problem starts with the following text:

> Something, something, data, something, something, big
>
> Written by HockeyInJune
>
> [pcap.pcapng]




## Inspecting the Wireshark File

The file extension [.pcapng] correspond to files for *packet capture*. They usually contain a dump of data packets captured over a network. This type of files holds blocks or data, and they can be used to rebuild captured packets into recognizable data.

We can open this file with [Wireshark], which is an open-source packet analyzer, or using [chaosreader], a freeware tool to trace TCP and UDP sessions. We choose the first. There are several things that we could explore and look for in this file:

        - We could search for all the interesting protocols inside and analyze them.

        - We could go to *Statistics*-> *Protocol Hierarchy* and look at the traffic patterns.

        - We could search in packet bytes, looking for specific strings such as login or password.

        - We could try to find something interesting in *Conversations*.


## Searching for the String *Password*

It turned out that all we need was to look for the string *password*. To do this we followed these steps in Wireshark:

         1. Go to *Edit*

         2. Go to *Find Packet*

         3. Search for **password** choosing the options *string* and *packet bytes*.


Yay! We found something over a **telnet** protocol:

![cyber](http://i.imgur.com/mUN4b1n.png)


____

## Following the TCP Stream

Now, all we need to do is to right-click in the line and choose *Follow TCP Stream*. This  returns:

```
..... .....'...........%..&..... ..#..'..$..%..&..#..............$.. .....'.............P...... .38400,38400....'.......XTERM.......".....!.....".....!............
Linux 3.13.0-32-generic (ubuntu) (pts/0)

..ubuntu login: j.ju.ul.li.ia.an.n
.
..Password: flag{bigdataisaproblemnotasolution}
.
.
Login incorrect
..ubuntu login:
```

And we find our flag: **bigdataisaproblemnotasolution**!


**Hack all the things!**

Edited: If you had decided to use *chaosreader* to process the pcapng file instead, the solution [from this write-up] is also cool:
```sh
for f in pcap.pcapng-chaosreader/*.html; do cat "${f}" | w3m -dump -T text/html "${f}"; done | egrep "flag{"
```

[from this write-up]: http://evandrix.github.io/ctf/2014-csaw-networking-100-bigdata.html
[pcap.pcapng]:https://github.com/ctfs/write-ups/blob/master/csaw-ctf-2014/big-data/pcap.pcapng
[.pcapng]: https://appliance.cloudshark.org/blog/5-reasons-to-move-to-pcapng/
[Wireshark]: https://www.wireshark.org/
[chaosreader]:http://chaosreader.sourceforge.net/


