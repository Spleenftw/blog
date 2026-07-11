---
title: "How to evade your ISP ?"
date: 2025-10-16
slug: hide-your-traffic-from-your-isp
status: 
tags:
  - linux
  - dns
  - debian
  - dot
  - doh
  - vpn
  - network
  - blog
  - homelab
  - isp
  - unbound
  - privacy
description: "How to (partially) evade your ISP: encrypt your DNS with DoH and DoT, and learn where a VPN actually helps and where it's just security theater."
category: network
---

<h2>How to (partially) evade your ISP ?</h2>
<p>Ever thought about the fact that your ISP supposedly sees everything you're doing ?</p>
<p>Well, they <strong>kinda do</strong>. And they're definitely taking notes.</p>
<p>Every time you browse the web, stream a video, or download something, your ISP is sitting in the middle watching. Sure, they can't read your HTTPS traffic or crack your passwords, but they know which domains you visit, when you visit them, and how much data you're pulling. Reddit at 2am? Streaming service during work hours? My own blog ? They see it all.</p>
<p>Some ISPs sell this data to advertisers. Others throttle specific services they don't like. Many just keep logs that can be requested by authorities—or leaked in a breach. Either way, it's your browsing patterns sitting in someone else's database.
So what can they actually see? Let's get technical for a minute.</p>
<h2><u> Context</u></h2>
<p>To my knowledge, your ISP only sees a <em>few things</em> : protocols, sources &amp; destination IPs, traffic volume, packet size and unencrypted data (like DNS).  </p>
<p>All of those metadata could lead your ISP to know you're <strong>web-browsing</strong> with protocols like <em>quic, http/s and dns</em> or  even <strong>downloading stuff</strong> because of the <em>traffic volume and packet size</em> but not the <strong>content</strong> (<em>what you are downloading or what you really are browsing</em>).</p>
<p><u>Example for some web browsing :</u></p>
<p>Let's say our computer (10.100.40.10) is researching stuff about what the word <em>interlope</em> means and we find this website : <a href="https://www.cnrtl.fr/lexicographie/interlope">https://www.cnrtl.fr/lexicographie/interlope</a>.</p>
<p>We'll simply make the webrequest with a curl and listen with tcpdump or wireshark from our gateway (40.254) to see what the ISP is supposed to see.</p>
<pre><code class="language-shell">interlope@deb12-lab:~$ curl -v https://www.cnrtl.fr/lexicographie/interlope</code></pre>
<p>On the fake ISP side (my own gateway), this is the direct output of the tcpdump, we'll only keep the interesting network frames :</p>
<details>
    <summary><u>View the full tcpdump output</u></summary>
<pre><code class="language-shell">root@GW01:~# tcpdump -i br40 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br40, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:12:33.292699 IP 10.100.40.10.ntp > 10.100.20.123.ntp: NTPv4, Client, length 48
18:12:33.292935 IP 10.100.20.123.ntp > 10.100.40.10.ntp: NTPv4, Server, length 48
18:12:34.990540 IP 10.100.40.10.42913 > 10.100.20.53.domain: 58930+ A? www.cnrtl.fr. (30)
18:12:34.990746 IP 10.100.40.10.42913 > 10.100.20.53.domain: 63792+ AAAA? www.cnrtl.fr. (30)
18:12:34.991164 IP 10.100.20.53.domain > 10.100.40.10.42913: 58930 1/0/0 A 193.54.6.12 (46)
18:12:34.991256 IP 10.100.20.53.domain > 10.100.40.10.42913: 63792 0/1/0 (110)
18:12:34.991612 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [S], seq 2204792747, win 62720, options [mss 8960,sackOK,TS val 1604861790 ecr 0,nop,wscale 7], length 0
18:12:35.013759 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [S.], seq 2301100463, ack 2204792748, win 65160, options [mss 1460,sackOK,TS val 471095553 ecr 1604861790,nop,wscale 7], length 0
18:12:35.013997 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 1, win 490, options [nop,nop,TS val 1604861813 ecr 471095553], length 0
18:12:35.016673 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [P.], seq 1:518, ack 1, win 490, options [nop,nop,TS val 1604861815 ecr 471095553], length 517
18:12:35.038771 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], ack 518, win 506, options [nop,nop,TS val 471095578 ecr 1604861815], length 0
18:12:35.048850 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 1:1449, ack 518, win 506, options [nop,nop,TS val 471095588 ecr 1604861815], length 1448
18:12:35.049009 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 1449:2897, ack 518, win 506, options [nop,nop,TS val 471095588 ecr 1604861815], length 1448
18:12:35.049089 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 1449, win 479, options [nop,nop,TS val 1604861848 ecr 471095588], length 0
18:12:35.049132 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 2897:3771, ack 518, win 506, options [nop,nop,TS val 471095588 ecr 1604861815], length 874
18:12:35.049143 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 2897, win 468, options [nop,nop,TS val 1604861848 ec 471095588], length 0
18:12:35.049231 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 3771, win 462, options [nop,nop,TS val 1604861848 ecr 471095588], length 0
18:12:35.051226 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [P.], seq 518:582, ack 3771, win 462, options [nop,nop,TS val 1604861850 ecr 471095588], length 64
18:12:35.052397 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [P.], seq 582:668, ack 3771, win 462, options [nop,nop,TS val 1604861851 ecr 471095588], length 86
18:12:35.053400 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [P.], seq 668:747, ack 3771, win 462, options [nop,nop,TS val 1604861852 ecr 471095588], length 79
18:12:35.073227 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 3771:3832, ack 582, win 506, options [nop,nop,TS val 471095612 ecr 1604861850], length 61
18:12:35.073538 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [P.], seq 747:778, ack 3832, win 462, options [nop,nop,TS val 1604861872 ecr 471095612], length 31
18:12:35.074236 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 3832:3898, ack 668, win 506, options [nop,nop,TS val 471095613 ecr 1604861851], length 66
18:12:35.083188 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 3898:4150, ack 747, win 506, options [nop,nop,TS val 471095622 ecr 1604861852], length 252
18:12:35.083249 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 4150:5598, ack 747, win 506, options [nop,nop,TS val 471095622 ecr 1604861852], length 1448
18:12:35.083273 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 5598:7046, ack 747, win 506, options [nop,nop,TS val 471095622 ecr 1604861852], length 1448
18:12:35.083294 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 7046:8494, ack 747, win 506, options [nop,nop,TS val 471095622 ecr 1604861852], length 1448
18:12:35.083312 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 8494:9942, ack 747, win 506, options [nop,nop,TS val 471095622 ecr 1604861852], length 1448
18:12:35.083331 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 9942:11390, ack 747, win 506, options [nop,nop,TS val 471095623 ecr 1604861852], length 1448
18:12:35.083357 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 11390:12838, ack 747, win 506, options [nop,nop,TS val 471095623 ecr 1604861852], length 1448
18:12:35.083415 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 12838:14286, ack 747, win 506, options [nop,nop,TS val 471095623 ecr 1604861852], length 1448
18:12:35.083508 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 14286, win 443, options [nop,nop,TS val 1604861882 ecr 471095613], length 0
18:12:35.094718 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 14286:15734, ack 778, win 506, options [nop,nop,TS val 471095634 ecr 1604861872], length 1448
18:12:35.094744 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 15734:17182, ack 778, win 506, options [nop,nop,TS val 471095634 ecr 1604861872], length 1448
18:12:35.094967 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 17182, win 443, options [nop,nop,TS val 1604861894 ecr 471095634], length 0
18:12:35.104710 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 17182:18630, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104735 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 18630:20078, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104755 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 20078:21526, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104773 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 21526:22974, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104791 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 22974:24422, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104809 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 24422:25870, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104832 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [.], seq 25870:27318, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 1448
18:12:35.104838 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [P.], seq 27318:27547, ack 778, win 506, options [nop,nop,TS val 471095644 ecr 1604861882], length 229
18:12:35.104907 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 20078, win 443, options [nop,nop,TS val 1604861903 ecr 471095644], length 0
18:12:35.104937 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 22974, win 443, options [nop,nop,TS val 1604861904 ecr 471095644], length 0
18:12:35.104960 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 25870, win 443, options [nop,nop,TS val 1604861904 ecr 471095644], length 0
18:12:35.104980 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [.], ack 27547, win 443, options [nop,nop,TS val 1604861904 ecr 471095644], length 0
18:12:35.124684 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [F.], seq 778, ack 27547, win 443, options [nop,nop,TS val 1604861923 ecr 471095644], length 0
18:12:35.146335 IP cnrtl.fr.https > 10.100.40.10.42836: Flags [F.], seq 27547:27571, ack 779, win 506, options [nop,nop,TS val 471095685 ecr 1604861923], length 24
18:12:35.146580 IP 10.100.40.10.42836 > cnrtl.fr.https: Flags [R], seq 2204793526, win 0, length 0
^C
50 packets captured
50 packets received by filter
0 packets dropped by kernel
</code></pre>
</details>
<p><br>
Let's see what we got : NTP, DNS, and HTTPS</p>
<pre><code>18:12:33.292699 IP 10.100.40.10.ntp &gt; 10.100.20.123.ntp: NTPv4, Client, length 48
18:12:33.292935 IP 10.100.20.123.ntp &gt; 10.100.40.10.ntp: NTPv4, Server, length 48
18:12:34.990540 IP 10.100.40.10.42913 &gt; 10.100.20.53.domain: 58930+ A? www.cnrtl.fr. (30)
18:12:34.990746 IP 10.100.40.10.42913 &gt; 10.100.20.53.domain: 63792+ AAAA? www.cnrtl.fr. (30)
18:12:34.991164 IP 10.100.20.53.domain &gt; 10.100.40.10.42913: 58930 1/0/0 A 193.54.6.12 (46)
18:12:34.991256 IP 10.100.20.53.domain &gt; 10.100.40.10.42913: 63792 0/1/0 (110)
18:12:34.991612 IP 10.100.40.10.42836 &gt; cnrtl.fr.https: Flags [S], seq 2204792747, win 62720, options [mss 8960,sackOK,TS val 1604861790 ecr 0,nop,wscale 7], length 0
18:12:35.013759 IP cnrtl.fr.https &gt; 10.100.40.10.42836: Flags [S.], seq 2301100463, ack 2204792748, win 65160, options [mss 1460,sackOK,TS val 471095553 ecr 1604861790,nop,wscale 7], length 0</code></pre>
<p>We can see that our <code>deb12-lab</code> asks for time to one of my local ntp relays :</p>
<pre><code class="language-shell">18:12:33.292699 IP 10.100.40.10.ntp &gt; 10.100.20.123.ntp: NTPv4, Client, length 48
18:12:33.292935 IP 10.100.20.123.ntp &gt; 10.100.40.10.ntp: NTPv4, Server, length 48</code></pre>
<p>He is also asking one of my local dns server where the <code>www.cnrtl.fr</code> points to :
(A records being IPV4 and AAAA records being IPV6)</p>
<pre><code class="language-shell">18:12:34.990540 IP 10.100.40.10.42913 &gt; 10.100.20.53.domain: 58930+ A? www.cnrtl.fr. (30)
18:12:34.990746 IP 10.100.40.10.42913 &gt; 10.100.20.53.domain: 63792+ AAAA? www.cnrtl.fr. (30)
18:12:34.991164 IP 10.100.20.53.domain &gt; 10.100.40.10.42913: 58930 1/0/0 A 193.54.6.12 (46)
18:12:34.991256 IP 10.100.20.53.domain &gt; 10.100.40.10.42913: 63792 0/1/0 (110)</code></pre>
<p>And then, the most interesting one, the curl :</p>
<pre><code class="language-shell">18:12:34.991612 IP 10.100.40.10.42836 &gt; cnrtl.fr.https: Flags [S], seq 2204792747, win 62720, options [mss 8960,sackOK,TS val 1604861790 ecr 0,nop,wscale 7], length 0
18:12:35.013759 IP cnrtl.fr.https &gt; 10.100.40.10.42836: Flags [S.], seq 2301100463, ack 2204792748, win 65160, options [mss 1460,sackOK,TS val 471095553 ecr 1604861790,nop,wscale 7], length 0</code></pre>
<p>So with what I'm seeing, all I could tell is that the 10.100.40.10 (our <code>deb12-lab</code>) is visiting the website www.cnrtl.fr, but not that he went to see the <code>/lexicographie/interlope</code>. So he has an idea but doesn't know what I'm really doing.</p>
<p>So what choices do we have to achieve the greatest possible anonymity and be a fully <em>net-ja</em> (network ninja haha.. you got it right ? right ?..) ? There's 3 easy choices : encrypting your DNS with HTTPS, encrypting your DNS with TLS or sending your entire traffic through a vpn-tunnel. For every case, you'll need to bring in and trust a third-party. </p>
<h2><u> DoH (DNS over HTTPS)</u></h2>
<p>This one is quite easy and forwarding. We'll install another local dns called <code>pihole</code> and configure it to use DoH with Cloudflare acting as a third-party. This means our ISP won't be able to see our dns queries since we'll ask cloudflare via https to be our resolver. </p>
<h3><u> Architecture overview </u></h3>
<pre><code>Client (lab-deb12) → Local DNS Server (lab-dns01) → [DoH Encrypted] → Cloudflare → Internet</code></pre>
<pre><code>[lab-deb12]          [lab-dns01]              [Cloudflare]        [Internet]
10.100.40.10         10.100.40.53             104.16.249.249      
                     (Pi-hole + cloudflared)
     |                    |                         |                  |
     |--DNS Query--------&gt;|                         |                  |
     |  Port 53           |                         |                  |
     |  Plain Text        |                         |                  |
     |  "youtube.fr?"     |                         |                  |
     |                    |                         |                  |
     |                    |--DoH Query-------------&gt;|                  |
     |                    |  Port 443 (HTTPS)       |                  |
     |                    |  Encrypted              |                  |
     |                    |                         |                  |
     |                    |                         |                  |
     |                    |                         |--DNS Query------&gt;|
     |                    |                         |  Plain Text      |
     |                    |                         |                  |
     |                    |                         |&lt;--DNS Response---|
     |                    |&lt;--DoH Response----------|                  |
     |&lt;--DNS Response-----|                         |                  |</code></pre>
<h3><u> Technical overview </u></h3>
<p>We'll simply follow the pihole documentation to install it on our <code>lab-dns01 | 10.100.40.53</code>. We change our <code>/etc/resolv.conf</code> to point to it.
We check that he's the one resolving our dns queries :</p>
<pre><code class="language-shell">interlope@lab-deb12:~$ nslookup google.fr
Server:         10.100.40.53
Address:        10.100.40.53#53

Non-authoritative answer:
Name:   google.fr
Address: 172.217.171.227
Name:   google.fr
Address: 2a00:1450:4006:804::2003</code></pre>
<p>Currently, there's no configuration for DoH, my pihole is pointing at cloudflare dns servers to resolve our query :</p>
<pre><code class="language-shell">root@GW01:~# tcpdump -i br40 'port 53'
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br40, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:18:29.875157 IP 10.100.40.53.51369 &gt; one.one.one.one.domain: 35695+ [1au] A? google.fr. (38)
14:18:29.884800 IP one.one.one.one.domain &gt; 10.100.40.53.51369: 35695 1/0/1 A 142.251.37.195 (54)</code></pre>
<p>We'll follow this <a href="https://docs.pi-hole.net/guides/dns/cloudflared/">procedure</a> to configure DoH via cloudflare.d as the third party actor.</p>
<pre><code class="language-shell">root@lab-dns01:/home/interlope# cloudflared -v
cloudflared version 2025.10.0 (built 2025-10-14-19:01 UTC)</code></pre>
<p>And we can now see that our pihole is doing <strong>DNS over HTTPS</strong>, so two things : we can't see what I looked for (it's not plain text anymore asking for <em>google.fr</em>) and it isn't considered as DNS.  </p>
<pre><code class="language-shell">root@GW01:~# tcpdump -i br40
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br40, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:24:13.867038 IP 10.100.40.53.52118 &gt; 104.16.249.249.https: Flags [.], ack 2691802441, win 447, options [nop,nop,TS val 371196123 ecr 1130067145], length 0
14:24:13.873862 IP 104.16.249.249.https &gt; 10.100.40.53.52118: Flags [.], ack 1, win 16, options [nop,nop,TS val 1130082249 ecr 371165918], length 0</code></pre>
<p>We cannot see the dns exchange between the <code>lab-deb12</code> and <code>lab-dns01</code> from the <code>GW01</code> since they are on the same vlan, this is how it looks like :</p>
<details>
    <summary><u>View the full tcpdump output</u></summary>
<pre><code class="language-shell">root@lab-dns01:~# tcpdump 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens18, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:29:46.488255 IP 10.100.40.10.37965 > lab-dns01.hlb1.lan.domain: 8769+ A? youtube.fr. (28)
14:29:46.488738 IP lab-dns01.hlb1.lan.52118 > 104.16.249.249.https: Flags [P.], seq 2488306043:2488306084, ack 2691803761, win 443, options [nop,nop,TS val 371528746 ecr 1130404040], length 41
14:29:46.488776 IP lab-dns01.hlb1.lan.52118 > 104.16.249.249.https: Flags [P.], seq 41:111, ack 1, win 443, options [nop,nop,TS val 371528746 ecr 1130404040], length 70
14:29:46.496251 IP 104.16.249.249.https > lab-dns01.hlb1.lan.52118: Flags [.], ack 111, win 16, options [nop,nop,TS val 1130414872 ecr 371528746], length 0
14:29:46.510071 IP lab-dns01.hlb1.lan.54978 > dns01.hlb1.lan.domain: 24679+ PTR? 10.40.100.10.in-addr.arpa. (43)
14:29:46.511176 IP dns01.hlb1.lan.domain > lab-dns01.hlb1.lan.54978: 24679 NXDomain* 0/0/0 (43)
14:29:46.511440 IP lab-dns01.hlb1.lan.56613 > dns01.hlb1.lan.domain: 58152+ PTR? 249.249.16.104.in-addr.arpa. (45)
14:29:46.511959 IP dns01.hlb1.lan.domain > lab-dns01.hlb1.lan.56613: 58152 NXDomain 0/1/0 (140)
14:29:46.513302 IP 104.16.249.249.https > lab-dns01.hlb1.lan.52118: Flags [P.], seq 1:81, ack 111, win 16, options [nop,nop,TS val 1130414890 ecr 371528746], length 80
14:29:46.513332 IP 104.16.249.249.https > lab-dns01.hlb1.lan.52118: Flags [P.], seq 81:167, ack 111, win 16, options [nop,nop,TS val 1130414890 ecr 371528746], length 86
14:29:46.513377 IP lab-dns01.hlb1.lan.52118 > 104.16.249.249.https: Flags [.], ack 167, win 443, options [nop,nop,TS val 371528770 ecr 1130414890], length 0
14:29:46.513800 IP lab-dns01.hlb1.lan.domain > 10.100.40.10.37965: 8769 1/0/0 A 216.58.205.206 (54)
14:29:46.515365 IP 10.100.40.10.54375 > lab-dns01.hlb1.lan.domain: 19750+ AAAA? youtube.fr. (28)
</code></pre>
</details>
<br>
Which we could split into "4" queries :
- 1. DNS query from ``lab-deb12`` to ``lab-dns01`` : ``4:29:46.488255 IP 10.100.40.10.37965 > lab-dns01.hlb1.lan.domain: 8769+ A? youtube.fr. (28)``
<br>
- 2. DoH query from ``lab-dns01`` to ``cloudflare`` : ``14:29:46.488738 IP lab-dns01.hlb1.lan.52118 > 104.16.249.249.https: Flags [P.], seq 2488306043:2488306084, ack 2691803761, win 443, options [nop,nop,TS val 371528746 ecr 1130404040], length 41``
<ul>
<li>
<ol start="3">
<li>Doh response from <code>cloudflare</code> to <code>lab-dns01</code> :  <code>14:29:46.496251 IP 104.16.249.249.https &gt; lab-dns01.hlb1.lan.52118: Flags [.], ack 111, win 16, options [nop,nop,TS val 1130414872 ecr 371528746], length 0</code></li>
</ol>
</li>
<li>
<ol start="4">
<li>DNS response from <code>lab-dns01</code> to <code>lab-deb12</code> : <code>14:29:46.513800 IP lab-dns01.hlb1.lan.domain &gt; 10.100.40.10.37965: 8769 1/0/0 A 216.58.205.206 (54)</code></li>
</ol>
</li>
</ul>
<p>So our goal to hide our DNS queries from our ISP is achieved. Woohoo</p>
<h2><u> DoT (DNS over TLS)</u></h2>
<p>DNS over TLS works in a different way than DoH, you'll need a third party DNS resolver and use certificates to encrypt your queries.
We well simulate the use of a VPS with a virtual machine in another vlan, so that we can see what goes through our firewall.</p>
<h3><u> Architecture overview </u></h3>
<pre><code class="language-shell">Client (lab-deb12) → Local DNS Server (lab-dns01) → [DoT Encrypted] → VPS (dmz-dot) → Internet</code></pre>
<pre><code>[lab-deb12]          [lab-dns01]              [dmz-dot]           [Internet]
10.100.40.10         10.100.40.53             (Your VPS)          
     |                    |                         |                  |
     |--DNS Query--------&gt;|                         |                  |
     |  Port 53           |                         |                  |
     |  Plain Text        |                         |                  |
     |  "youtube.fr?"     |                         |                  |
     |                    |                         |                  |
     |                    |--DoT Query-------------&gt;|                  |
     |                    |  Port 853 (TLS)         |                  |
     |                    |  Encrypted              |                  |
     |                    |                         |                  |
     |                    |                         |                  |
     |                    |                         |--DNS Query------&gt;|
     |                    |                         |  Plain Text      |
     |                    |                         |                  |
     |                    |                         |&lt;--DNS Response---|
     |                    |&lt;--DoT Response----------|                  |
     |&lt;--DNS Response-----|                         |                  |</code></pre>
<h3><u> Technical overview </u></h3>
<p>First, let's install unbound and openssl on both our local DNS and our VPS.</p>
<pre><code class="language-shell">sudo apt install unbound openssl</code></pre>
<p>Then we'll need to generate our self-signed certificate with openssl and store them somewhere nice and warm like <code>/etc/unbound/certs</code> :</p>
<pre><code class="language-shell">sudo mkdir -p /etc/unbound/certs &amp;&amp; cd /etc/unbound/certs
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout unbound-server.key -out unbound-server.pem -subj "/CN=hostnamevps.domain" -addext "subjectAltName=DNS:hostnamevps.domain,IP:VPS-public-ip"</code></pre>
<p>You'll probably need to set some permissions on them (a good old 640 should be enough). Don't forget to copy the .pem file, you'll need it on your local DNS server.</p>
<p>On your VPS, you'll need to set up <em>unbound</em> and <em>DoT</em> with this file : <code>/etc/unbound/unbound.conf.d/dot-server.conf</code></p>
<pre><code> server:
    interface: 127.0.0.1
    interface: 0.0.0.0@853

    tls-service-key: "/etc/unbound/unbound_server.key"
    tls-service-pem: "/etc/unbound/unbound_server.pem"
    tls-port: 853

    # Here you can allow only your public IP to use the unbound service
    #access-control: 0.0.0.0/0 allow

    # Security
    hide-identity: yes
    hide-version: yes

    # Performance
    num-threads: 4
    msg-cache-size: 16m
    rrset-cache-size: 32m
    prefetch: yes

# Forward to public DNS like Cloudflare
forward-zone:
    name: "."
    forward-addr: 1.1.1.1
        forward-addr: 1.1.1.2</code></pre>
<p><em>Don't forget to enable and start it with systemctl</em></p>
<p>Then we jump on our local dns server, install the <code>unbound</code> and <code>openssl</code> packages, configure it to forward to the VPS :</p>
<pre><code> server:
    # Listen on all interfaces
    interface: 0.0.0.0

    # Here you can allow your local network to query
    access-control: 10.100.40.0/24 allow

    # Privacy settings
    hide-identity: yes
    hide-version: yes

    # Performance
    num-threads: 2
    msg-cache-size: 8m
    rrset-cache-size: 16m
    prefetch: yes

    # THIS IS WHERE YOU PUT YOUR VPS CERT .PEM FILE
    tls-cert-bundle: "/etc/unbound/vps_unbound_server.pem"

forward-zone:
    name: "."

    # Forward to your VPS over DoT
    forward-addr: vps-ip@853#hostnamevps.domain
    forward-tls-upstream: yes</code></pre>
<p><em>bis :Don't forget to enable and start it with systemctl</em></p>
<h3><u> Testing </u></h3>
<p>On our local dns server <code>lab-dns01</code>, we'll use the dig command to test the local resolution and forward :</p>
<pre><code>interlope@lab-dns01:~$ dig @localhost google.com
; &lt;&lt;&gt;&gt; DiG 9.18.33-1~deb12u2-Debian &lt;&lt;&gt;&gt; @localhost google.com
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 29592
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             86      IN      A       172.217.19.142

;; Query time: 48 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Wed Oct 15 23:42:16 CEST 2025
;; MSG SIZE  rcvd: 55</code></pre>
<p>Then we'll check the logs for TLS connection :</p>
<pre><code>interlope@lab-dns01:/etc/unbound/certs$ sudo journalctl -u unbound | grep -E "reply from|ssl handshake"

Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:04:58 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:02 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:02 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:02 lab-dns01 unbound[1068]: [1068:0] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:19 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:19 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:07:19 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:42:16 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:42:16 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853</code></pre>
<p>Now we can make a query from our <code>lab-deb12</code>, check <code>lab-dns01</code> unbound logs and listen on our firewall <code>GW01</code> to see what the fictional ISP could see :</p>
<pre><code>interlope@lab-deb12:~$ dig amazon.com

; &lt;&lt;&gt;&gt; DiG 9.18.33-1~deb12u2-Debian &lt;&lt;&gt;&gt; amazon.com
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 62591
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;amazon.com.                    IN      A

;; ANSWER SECTION:
amazon.com.             730     IN      A       205.251.242.103
amazon.com.             730     IN      A       52.94.236.248
amazon.com.             730     IN      A       54.239.28.85

;; Query time: 52 msec
;; SERVER: 10.100.40.53#53(10.100.40.53) (UDP)
;; WHEN: Wed Oct 15 23:44:16 CEST 2025
;; MSG SIZE  rcvd: 87</code></pre>
<pre><code>interlope@lab-dns01:/etc/unbound/certs$ sudo journalctl -u unbound | tail -20
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: 10.100.40.10 amazon.com. A IN
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: resolving amazon.com. A IN
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: response for amazon.com. A IN
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: query response was ANSWER
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: resolving amazon.com. DS IN
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: response for amazon.com. DS IN
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: reply from &lt;.&gt; 10.100.25.54#853
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: query response was nodata ANSWER
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: NSEC3s for the referral proved no DS.
Oct 15 23:44:16 lab-dns01 unbound[1068]: [1068:1] info: Verified that unsigned response is INSECURE</code></pre>
<details>
    <summary><u>View the full tcpdump output</u></summary>
<pre><code class="language-shell">root@GW01:~# tcpdump -i br40 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br40, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23:47:11.350937 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [S], seq 3549119131, win 62720, options [mss 8960,sackOK,TS val 1011458771 ecr 0,nop,wscale 7], length 0
23:47:11.351287 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [S.], seq 2449201382, ack 3549119132, win 62636, options [mss 8960,sackOK,TS val 2741938261 ecr 1011458771,nop,wscale 7], length 0
23:47:11.351383 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 1, win 490, options [nop,nop,TS val 1011458771 ecr 2741938261], length 0
23:47:11.351591 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 1:323, ack 1, win 490, options [nop,nop,TS val 1011458772 ecr 2741938261], length 322
23:47:11.351869 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [.], ack 323, win 487, options [nop,nop,TS val 2741938261 ecr 1011458772], length 0
23:47:11.352919 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1:1411, ack 323, win 487, options [nop,nop,TS val 2741938262 ecr 1011458772], length 1410
23:47:11.353088 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 1411, win 479, options [nop,nop,TS val 1011458773 ecr 2741938262], length 0
23:47:11.353591 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 323:403, ack 1411, win 479, options [nop,nop,TS val 1011458774 ecr 2741938262], length 80
23:47:11.353868 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1411:1666, ack 403, win 487, options [nop,nop,TS val 2741938263 ecr 1011458774], length 255
23:47:11.353942 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 403:555, ack 1666, win 478, options [nop,nop,TS val 1011458774 ecr 2741938263], length 152
23:47:11.354068 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1666:1921, ack 555, win 486, options [nop,nop,TS val 2741938263 ecr 1011458774], length 255
23:47:11.395579 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 1921, win 477, options [nop,nop,TS val 1011458816 ecr 2741938263], length 0
23:47:11.395812 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1921:2413, ack 555, win 486, options [nop,nop,TS val 2741938305 ecr 1011458816], length 492
23:47:11.395929 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 2413, win 474, options [nop,nop,TS val 1011458816 ecr 2741938305], length 0
23:47:11.396138 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 555:707, ack 2413, win 474, options [nop,nop,TS val 1011458816 ecr 2741938305], length 152
23:47:11.396409 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 2413:3373, ack 707, win 485, options [nop,nop,TS val 2741938306 ecr 1011458816], length 960
23:47:11.439598 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 3373, win 467, options [nop,nop,TS val 1011458860 ecr 2741938306], length 0
^C
20 packets captured
20 packets received by filter
0 packets dropped by kernel
</code></pre>
</details>
<br>
Which we can also split in 4 main steps :
- 1. TCP handshake from ``lab-dns01`` to our ``"VPS"`` (`S`, `S.` and `.` flags stands for `SYN`, `SYN+ACK`, `ACK`) : 
```
23:47:11.350937 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [S], seq 3549119131, win 62720, options [mss 8960,sackOK,TS val 1011458771 ecr 0,nop,wscale 7], length 0
23:47:11.351287 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [S.], seq 2449201382, ack 3549119132, win 62636, options [mss 8960,sackOK,TS val 2741938261 ecr 1011458771,nop,wscale 7], length 0
23:47:11.351383 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 1, win 490, options [nop,nop,TS val 1011458771 ecr 2741938261], length 0
```
- 2. TLS handshake from ``lab-dns01`` to our ``"VPS"`` (`P` and `P.` flags stands for `PUSH`, `PUSH+ACK`): 
```
23:47:11.351591 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 1:323, ack 1, win 490, options [nop,nop,TS val 1011458772 ecr 2741938261], length 322
23:47:11.352919 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1:1411, ack 323, win 487, options [nop,nop,TS val 2741938262 ecr 1011458772], length 1410
23:47:11.353088 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [.], ack 1411, win 479, options [nop,nop,TS val 1011458773 ecr 2741938262], length 0
```
- 3. Encrypted DoT query from ``lab-dns01`` to the ``"VPS`` :
```
23:47:11.353591 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 323:403, ack 1411, win 479, options [nop,nop,TS val 1011458774 ecr 2741938262], length 80
23:47:11.353942 IP 10.100.40.53.42910 > 10.100.25.54.domain-s: Flags [P.], seq 403:555, ack 1666, win 478, options [nop,nop,TS val 1011458774 ecr 2741938263], length 152
```
- 4. Encrypted DoT response from the ``"VPS"`` to ``lab-dns01`` :
```
23:47:11.353868 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1411:1666, ack 403, win 487, options [nop,nop,TS val 2741938263 ecr 1011458774], length 255
23:47:11.354068 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1666:1921, ack 555, win 486, options [nop,nop,TS val 2741938263 ecr 1011458774], length 255
23:47:11.395812 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 1921:2413, ack 555, win 486, options [nop,nop,TS val 2741938305 ecr 1011458816], length 492
23:47:11.396409 IP 10.100.25.54.domain-s > 10.100.40.53.42910: Flags [P.], seq 2413:3373, ack 707, win 485, options [nop,nop,TS val 2741938306 ecr 1011458816], length 960
```
<p>Guys, we've done it again. DoT is working, we are escaping the ISP eyes !</p>
<h2><u>VPN </u></h2>
<p>This one will need either a firewall so that you can route all of your traffic through the vpn tunnel. This will also cost a few bucks per months to rent either a <strong>VPS</strong> (the smallest one with like 2/4c and 2/4G RAM is way more than enough) or directly a <strong>VPN connection</strong> like <a href="https://airvpn.org">airvpn</a> (this is absolutely not an ad, but they have a few endpoints in Switzerland which means a better net-neutrality / law enforcements than in my country).</p>
<p>You can basically do two things with your vpn :</p>
<ul>
<li>do a whole policy-based routing config to route whatever you want through your vpn tunnel. Which would got a kill-switch if you ever lose the vpn connection, so that no data would be leaked to your ISP.</li>
<li>or less complicated, install the vpn on your machine and route everything through it.</li>
</ul>
<p>Either way, you'll simply "come out" from the endpoint of your vpn provider and your ISP will only see encrypted data going to the provider. It would look like this where we can only see the DoH query and the entire traffic from the 10.100.40.10 going through an openvpn tunnel.</p>
<pre><code class="language-shell">root@lab-deb12:~# ip r l
0.0.0.0/1 via 10.100.100.49 dev tun0
default via 10.100.40.254 dev ens18 onlink
10.100.5.45 via 10.100.40.254 dev ens18
10.100.40.0/24 dev ens18 proto kernel scope link src 10.100.40.10
10.100.100.48/28 dev tun0 proto kernel scope link src 10.100.100.50
128.0.0.0/1 via 10.100.100.49 dev tun0</code></pre>
<pre><code class="language-shell">root@GW01:~# tcpdump -i br40 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br40, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:18:15.388886 IP 10.100.40.10.57179 &gt; 10.100.5.45.openvpn: UDP, length 108
17:18:16.412912 IP 10.100.40.10.57179 &gt; 10.100.5.45.openvpn: UDP, length 108
17:18:17.436915 IP 10.100.40.10.57179 &gt; 10.100.5.45.openvpn: UDP, length 108
17:18:18.460885 IP 10.100.40.10.57179 &gt; 10.100.5.45.openvpn: UDP, length 108
17:18:19.484919 IP 10.100.40.10.57179 &gt; 10.100.5.45.openvpn: UDP, length 108
17:18:21.456453 IP 10.100.5.45.openvpn &gt; 10.100.40.10.57179: UDP, length 40
17:18:22.249888 IP 10.100.40.53.60362 &gt; 104.16.249.249.https: Flags [.], ack 1710117042, win 451, options [nop,nop,TS val 381644507 ecr 288606289], length 0
17:18:22.256605 IP 104.16.249.249.https &gt; 10.100.40.53.60362: Flags [.], ack 1, win 16, options [nop,nop,TS val 288621393 ecr 381614276], length 0</code></pre>
<h2><u> The Trust Paradox </u></h2>
<p>Here's the truth, we didn't really escape surveillance, we simply outsourced it. Your ISP ain't seeing your DNS queries or explicit traffic, someone else does. Someone like Cloudflare, your VPS provider or the VPN provider. Don't fool yourself thinking you're completely anonymous now, you are not. We are just switching the trust from one provider to another. Pick your third-party carefully, look at their logging policy, their history with the government. </p>
<p>That's all for today, be safe on the Internet and <strong>fuck Chat Control</strong>.</p>
<p>Thanks for reading me,</p>
<p>spleenftw</p>