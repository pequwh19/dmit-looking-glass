# DMIT Looking Glass: test LAX, HKG and TYO latency, routing and packet loss before you buy, with the full test IP list

Most people choose a VPS by looking at a spec sheet. Two cores, 2 GB RAM, 80 GB SSD, all of it identical across half a dozen providers. What actually decides whether the box is usable is which path your packets take — and that is the one thing a spec sheet never shows you.

DMIT sells its plans almost entirely on routing: CN2 GIA into mainland China on the Premium line, CMIN2 on Eyeball, plain international transit on Tier 1. If you are trying to work out whether the difference is worth 3–5x the price, the Looking Glass is where you find out. Not a review blog, not a benchmark chart — the actual routers, queried from inside the actual datacenters.

This is a walkthrough of DMIT's Looking Glass, the node list with test IPs, how to read the output without fooling yourself, and which plan the results point to.

## What the DMIT Looking Glass actually does

A Looking Glass is a diagnostic page that a hosting provider exposes from inside its own network. You give it a target address, it runs the test from the provider's side, and it shows you the result. DMIT runs one at `lg.dmit.sh`, covering Los Angeles, Hong Kong and Tokyo.

The direction matters more than most people realize. When you `ping` a provider's test IP from your laptop, you are measuring the **outbound** path — your ISP to their rack. The Looking Glass measures the **return** path, from their machines back toward you. For anyone hosting a website or a service, the return path is the one your visitors ride on, and asymmetric routing is common: a connection that looks like CN2 GIA on the way out can come home over the 163 backbone and fall apart at 9 pm.

The command selector on a DMIT node typically includes:

- **ping** — round-trip latency and loss, the blunt connectivity check
- **traceroute** — the hop-by-hop path, which is how you spot a route taking a detour through Europe
- **mtr** — traceroute plus continuous pinging, so you see per-hop loss instead of one aggregate number
- **nexttrace** — a newer traceroute implementation with better AS-level detail
- **iperf3** — throughput against a public iperf3 server, for measuring real transfer capacity

Several nodes also expose download test files, which is the only honest way to confirm that a "10 Gbps port" actually moves data at 10 Gbps from where you sit.

Each node also has its own dedicated endpoint on port 8443 (for example `lax-pro.lg.dmit.sh:8443`, with separate IPv6 variants), which is handy if you want to point a script at one specific location instead of clicking through the web UI.

## Every node in DMIT's Looking Glass, with test IPs

DMIT groups its plans by network profile, and the Looking Glass mirrors that structure. Here is the current set.

| Node | Routing profile | Test IPv4 | Looking Glass |
| --- | --- | --- | --- |
| Los Angeles Pro | Premium, three-network CN2 GIA return | 154.17.30.96 | `lg.dmit.sh/?server=lax-pro` |
| Los Angeles Eyeball | Eyeball, CMIN2 / 9929 return | 154.17.229.24 | `lg.dmit.sh/?server=lax-eb` |
| Los Angeles Premium Secure | Premium plus high-volume DDoS scrubbing | 45.88.194.187 | `lg.dmit.sh/?server=lax-spro` |
| Los Angeles Tier 1 | International transit, no China optimization | 174.136.204.135 | `lg.dmit.sh/?server=lax-t1` |
| Hong Kong Pro | Premium, CN2 GIA + Unicom 9929 + CMI | 103.117.100.199 | `lg.dmit.sh/?server=hkg-pro` |
| Hong Kong Eyeball | Eyeball, CMI primary | 154.12.188.3 | `lg.dmit.sh/?server=hkg-eb` |
| Hong Kong Tier 1 | International transit | 154.12.176.28 | `lg.dmit.sh/?server=hkg-t1` |
| Tokyo Pro | Premium, CN2 GIA + 9929 + CMI | 154.12.190.32 | `lg.dmit.sh/?server=tyo-pro` |
| Tokyo Tier 1 | International transit | 154.31.112.5 | `lg.dmit.sh/?server=tyo-t1` |

If you would rather skip the browser entirely, the test IPs above are the ones to drop into a terminal — `ping 154.17.30.96` from your own machine gives you the outbound half of the picture in about ten seconds.

Two things to know before you copy those numbers anywhere. First, IP addresses on a Looking Glass occasionally get rotated when DMIT re-racks hardware, so treat the LG page itself as the source of truth and these as a convenience. Second, DMIT's Tokyo Eyeball node is gone — the TYO.AS3.EB line was retired, which is why it is absent from the table rather than merely quiet.

## How to run a real pre-purchase test in about ten minutes

The mistake people make is opening the Looking Glass, running one ping, seeing a nice number and buying. Here is the sequence that actually tells you something.

**Step 1 — ping the test IP from your own connection.** This measures your ISP to the datacenter. For a mainland China Telecom connection hitting Los Angeles, something in the 150–170 ms range is normal; Hong Kong lands around 10–30 ms. Watch two numbers: latency and loss. Zero percent loss is the target. Anything sustained above roughly 5% and you are looking at a line that will fight you at peak hours.

**Step 2 — open the LG node and reverse the direction.** This is the part that confuses first-timers. On the Looking Glass page, the server's own IP is listed in the information panel — that is what *you* ping. The target box in the command area is the opposite: it is where the DMIT machine sends traffic, so you put **your own public IP** in there. Run traceroute or mtr against yourself.

**Step 3 — check the AS numbers in the return path.** This is where the marketing claims get settled. A few you will see repeatedly:

- **59.43.x.x / AS4809** — China Telecom's CN2 backbone. This is the GIA path you are paying for.
- **202.97.x.x** — Telecom's 163 network. Cheap, and famously congested after 8 pm.
- **AS9929** — China Unicom's premium international route.
- **AS58807 / CMIN2** — China Mobile's international service, the backbone of the Eyeball line.

If a node advertised as Premium returns through 202.97 instead of 59.43, that is a real finding, not a rounding error. If a Tier 1 node returns through 202.97, that is simply what Tier 1 is — it has no China optimization by design, and the LG is confirming it rather than failing.

**Step 4 — measure throughput, not just latency.** Run iperf3 or pull a download test file and watch what the transfer actually sustains. Latency and bandwidth are separate qualities, and providers know that ICMP echo requests are cheap to prioritize. One published test from the lax-pro node to a mainland China Telecom address returned 5/5 packets, 0% loss, 155.58 ms average and 0.05 ms jitter — good numbers, but that same test is also why you should not stop there.

**Step 5 — repeat it during China's evening peak.** Between 20:00 and 23:30 local time, the difference between a good line and a mediocre one stops being theoretical. A route that holds under 2% loss at peak is doing its job. This is the single most useful test you can run, and almost nobody runs it twice.

> Loss appearing at one intermediate hop and disappearing before the destination is usually a router rate-limiting ICMP replies, not a broken path. Loss that continues all the way to the last hop is the real thing.

## What the numbers tell you about which plan to buy

Once you have run the test against two or three nodes, the plan decision mostly makes itself.

**If your traceroute back home is full of 59.43.x.x and your measured latency is competitive with the physical distance** — roughly 150 ms to Los Angeles, 30–50 ms to Hong Kong from southern China — the Premium line is delivering what it promises. Los Angeles Pro is the cheaper way to get it; Hong Kong Pro costs several times more but cuts the round trip by a hundred milliseconds or more.

**If the Premium path looks fine but you are on China Mobile**, run the same test against the Eyeball node before spending the premium. The Eyeball line routes through CMIN2 and CMI, and for Mobile users the latency gap against Premium is often small while the price gap is not.

**If your test traffic is entirely international** — CI/CD runners, backups, a service for European or American users, a relay between two clouds — run the test against Tier 1 and stop worrying about China routing. These plans are the cheapest DMIT sells, and once traffic exceeds the monthly allowance they do not shut down: they throttle and keep running unmetered.

One caveat worth reading before you buy Hong Kong Eyeball: DMIT's own cart page labels that line as Beta, with a note that routing is still being tuned and that it is not recommended for workloads needing high stability. That is DMIT describing the product accurately, and it is the kind of disclosure you should take at face value.

👉 [Check current LAX, HKG and TYO availability on the order page](https://www.dmit.io/aff.php?aff=12046)

## The full DMIT plan list and current prices

Prices below are the monthly or annual figures DMIT's cart was displaying when this was checked, in USD. DMIT has three hardware generations in circulation — AS3 (EPYC 7003), AN4 (EPYC 9004) and AN5 (EPYC 9005) — and identical-sounding plans carry different prices depending on which one they sit on. Stock moves constantly; at the time of writing the cart flagged large parts of the Los Angeles Pro/Eyeball and Tokyo lines as out of stock, while the Tier 1 plans were largely available.

### Los Angeles — Premium and Eyeball

| Plan | Platform | CPU / RAM / SSD | Transfer @ port | After cap | Price | Order |
| --- | --- | --- | --- | --- | --- | --- |
| LAX.Pro.TINY | AS3 | 1 / 2 GB / 20 GB | 1000 GB @ 1 Gbps | 4 Mbps | $10.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.Pro.Pocket | AS3 | 2 / 2 GB / 40 GB | 1500 GB @ 4 Gbps | 4 Mbps | $16.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.Pro.STARTER | AS3 | 2 / 2 GB / 80 GB | 3000 GB @ 10 Gbps | 4 Mbps | $34.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.Pro.MINI | AS3 | 4 / 4 GB / 80 GB | 5000 GB @ 10 Gbps | 8 Mbps | $62.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.Pro.MICRO | AS3 | 4 / 4 GB / 160 GB | 7000 GB @ 10 Gbps | 8 Mbps | $87.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.Pro.MEDIUM | AS3 | 6 / 8 GB / 160 GB | 14000 GB @ 10 Gbps | 10 Mbps | $199.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.Pro.MINI | AN4 | 4 / 4 GB / 80 GB | 5000 GB @ 10 Gbps | 8 Mbps | $72.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.Pro.MICRO | AN4 | 4 / 4 GB / 160 GB | 7000 GB @ 10 Gbps | 8 Mbps | $102.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.Pro.MEDIUM | AN4 | 6 / 8 GB / 160 GB | 14000 GB @ 10 Gbps | 10 Mbps | $239.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.Pro.LARGE | AN4 | 8 / 16 GB / 320 GB | 25000 GB @ 10 Gbps | 10 Mbps | $459.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.Pro.GIANT | AN4 | 12 / 24 GB / 640 GB | 50000 GB @ 10 Gbps | 10 Mbps | $929.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.Pro.MINI | AN5 | 4 / 4 GB / 80 GB | 5000 GB @ 10 Gbps | 8 Mbps | $79.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.Pro.MICRO | AN5 | 4 / 4 GB / 160 GB | 7000 GB @ 10 Gbps | 8 Mbps | $110.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.Pro.MEDIUM | AN5 | 6 / 8 GB / 160 GB | 14000 GB @ 10 Gbps | 10 Mbps | $289.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.Pro.LARGE | AN5 | 8 / 16 GB / 320 GB | 25000 GB @ 10 Gbps | 10 Mbps | $499.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.Pro.GIANT | AN5 | 12 / 24 GB / 640 GB | 50000 GB @ 10 Gbps | 10 Mbps | $1009.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.TINY | AS3 | 1 / 2 GB / 20 GB | 1500 GB @ 2 Gbps | 4 Mbps | $10.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.Pocket | AS3 | 2 / 2 GB / 40 GB | 3000 GB @ 4 Gbps | 4 Mbps | $16.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.STARTER | AS3 | 2 / 2 GB / 80 GB | 5000 GB @ 10 Gbps | 4 Mbps | $34.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.MINI | AS3 | 4 / 4 GB / 80 GB | 10000 GB @ 10 Gbps | 8 Mbps | $62.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.MICRO | AS3 | 4 / 4 GB / 160 GB | 14000 GB @ 10 Gbps | 8 Mbps | $87.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.EB.MEDIUM | AS3 | 6 / 8 GB / 160 GB | 30000 GB @ 10 Gbps | 10 Mbps | $199.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.EB.MINI | AN4 | 4 / 4 GB / 80 GB | 10000 GB @ 10 Gbps | 8 Mbps | $72.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.EB.MICRO | AN4 | 4 / 4 GB / 160 GB | 14000 GB @ 10 Gbps | 8 Mbps | $102.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.EB.MEDIUM | AN4 | 6 / 8 GB / 160 GB | 30000 GB @ 10 Gbps | 10 Mbps | $239.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.EB.LARGE | AN4 | 8 / 16 GB / 320 GB | 50000 GB @ 10 Gbps | 10 Mbps | $459.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN4.EB.GIANT | AN4 | 12 / 24 GB / 640 GB | 100000 GB @ 10 Gbps | 10 Mbps | $929.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.EB.MINI | AN5 | 4 / 4 GB / 80 GB | 5000 GB @ 10 Gbps | 8 Mbps | $79.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.EB.MICRO | AN5 | 4 / 4 GB / 160 GB | 14000 GB @ 10 Gbps | 8 Mbps | $110.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.EB.MEDIUM | AN5 | 6 / 8 GB / 160 GB | 30000 GB @ 10 Gbps | 10 Mbps | $289.90/mo | [Order](https://www.dmit.io/aff.php?aff=12046) |
| LAX.AN5.EB.LARGE | AN5 | 8 / 16 GB / 320 GB | 50000 GB @ 10 Gbps | 10 Mbps | $499.90/mo | [Order](https://www.dmit |
