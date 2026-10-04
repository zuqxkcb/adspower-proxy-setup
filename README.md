# adspower proxy setup: how to add, test, and batch-assign residential proxies without breaking your profiles

Most people searching for this already own a proxy and just need AdsPower to stop saying "connection failed." Or they're buying proxies today and want to know which field the host goes in. AdsPower splits proxy handling across three different screens — the saved proxy list, the profile's Proxy tab, and the Excel batch importer — and that split is where most setup mistakes come from. Get those three straight and the rest is copy-paste.

Below is the whole flow, plus the parts the vendor docs tend to skip: what format your credentials actually have to be in, why the proxy check fails when your credentials are fine, and how to pick a plan without overbuying IPs you'll never open.

## Where proxy settings actually live in AdsPower

Three locations, three jobs:

- **Proxy Management → Proxy List** is a warehouse of credentials. You add IPs here once, then reuse them across profiles. Bulk adds cap at 500 entries per batch.
- **New Profile → Proxy** is per-profile. You either paste credentials directly ("Custom") or pick from the warehouse ("Saved Proxies").
- **Batch create / Excel update** is the scale path. One column decides which IP a profile gets.

If you're doing two profiles, use the Custom field and skip the warehouse entirely. If you're doing fifty, skip the Custom field and build the warehouse first.

## Step 0: get credentials in the format AdsPower expects

AdsPower wants four values: host, port, username, password. Whether you fill them one by one or paste a single string, the underlying shape is the same:


host:port:username:password


Two practical notes. First, with a residential provider that supports session control, the username is often the config file — geo, session ID, and rotation timing all live inside it, not in separate fields. 9Proxy documents this exact pattern for its residential proxy, where the username is structured like:


<subaccount>-country-<country_code>-st-<state_code>-city-<city_code>-isp-<isp_code>-ssid-<session_id>-sst-<session_time>


So `9proxy-country-US-ssid-phTMYuotAy` means sub-account `9proxy`, country US, and a session ID that keeps you on the same IP while it lasts. The password is your sub-account password. Nothing about the city or ISP is set anywhere except that username string, which is why a typo there produces a live proxy in the wrong country rather than an error message.

Second, 9Proxy runs two different products that reach AdsPower by different routes:

- **GB-based residential proxies** work straight from the dashboard — you generate an endpoint, get host/port/user/pass, paste it in. No software on your machine.
- **IP-based residential proxies** require the 9Proxy desktop app. It forwards your purchased IPs to local ports, so what you type into AdsPower is `127.0.0.1` plus a forwarded port.

That distinction matters more than any other detail in this guide, because it changes what "Host" means. People who buy IP-based packages and then go hunting for an IP address in their dashboard usually end up confused, since the address is local.

👉 [Generate your first 9Proxy residential endpoint](https://bit.ly/9-Proxy)

## Step 1: add the proxy to the proxy list (or don't)

Inside AdsPower: **Proxy Management → Proxy List → Add Proxy**.

Two input modes:

1. **Line-by-line batch entry.** One proxy per line, up to 500 at once. If you pick a proxy type in the dropdown first, you can paste bare strings. If you skip the dropdown, include the protocol: `socks5://127.0.0.1:4000:user:password`.
2. **Single entry** via the form fields.

A few behaviours worth knowing before you hit confirm:

- **IPv6 hosts need square brackets**: `[2001:db8::1]:8000:user:password`. Skip the brackets and the parser reads it as garbage.
- **Duplicate validation is automatic.** AdsPower compares new entries against existing ones on save, so re-importing a spreadsheet twice won't double your list.
- **Pick your IP checker deliberately.** AdsPower lets you choose the service used to resolve a proxy's location (IP2Location, IPAPI, and others). They disagree with each other on edge cases, so if the geo shown doesn't match what you bought, change the checker before you blame the proxy.
- **Protocol support**: HTTP, HTTPS, SSH, SOCKS5.

Then run **Check Proxy** on the entry. Green means the handshake works from your machine.

## Step 2: bind it to a profile

**New Profile → Proxy.** Fastest path: choose Custom, paste the full `host:port:user:password` string into the Host:Port field, and let AdsPower auto-fill the rest. Otherwise switch to **Saved Proxies** and select from the warehouse, or click the shuffle icon to let AdsPower assign a random one from your list.

Set the proxy type to match what your provider issued — HTTPS or SOCKS5 for a 9Proxy endpoint, per its own AdsPower integration guide. Choosing SOCKS5 when you were handed an HTTP endpoint (or the reverse) is a common source of failed checks.

For volume, use **Batch Create → Quick Create** and turn your attention to the proxy section:

- **Saved Proxies → Custom** assigns the *same* IP to every profile in the batch. Useful when you intentionally want several profiles behind one clean residential IP.
- **Saved Proxies → Random** spreads the batch across your warehouse automatically. This is what you want when each profile is a separate identity.

If you're managing this through Excel export/update instead, the `proxyid` column does the work: put the proxy's ID from your warehouse in it to hard-bind that specific IP, or write `random` to let AdsPower pick. One warning echoed in AdsPower's own docs — don't delete or rename the `acc_id` and `id` columns, or the update file won't map to anything. Also relevant here: `proxytype`, `ipchecker`, `proxy`, `countrycode`, `regioncode`, `citycode` are all writable columns, so you can rotate a whole farm's geography from a spreadsheet.

## Step 3: check, save, open

Click **Check Proxy** in the profile editor. On success, click **OK**, go to the Profiles list, and hit **Open**. The environment launches with the proxy already in place.

### When the check fails but the proxy is fine

The single most common cause isn't your provider. It's your own network. If you're on a local connection that can't reach the destination country directly, the handshake never leaves your machine. The fix is either running your local tunnel in global mode, or going into AdsPower's settings and enabling the option that lets the browser route its proxy connection through the local system channel. AdsPower's Chinese documentation says this outright; the English help centre is quieter about it.

Other things to check, in order:

- Protocol mismatch (SOCKS5 endpoint entered as HTTP, or the reverse)
- A typo in the username string, especially around the `country-` or `ssid-` segments
- Port not yet forwarded, for IP-based 9Proxy packages using the desktop app
- Expired sub-account password, if you rotated it recently

## IP-based vs GB-based: which 9Proxy product fits an AdsPower workflow

AdsPower's whole model is one browser fingerprint, one identity, one profile. That pushes you toward sticky IPs rather than per-request rotation — an account that logs in from three countries in ninety seconds is exactly the pattern anti-fraud systems look for.

|  | Residential by IPs | Residential by GB |
| --- | --- | --- |
| Billing unit | Per IP, fixed packages | Per GB, fixed packages |
| Bandwidth | Unlimited while the IP is active | Metered by purchased GB |
| IP duration | A few hours up to ~24h | Rotates per request, or sticky per configured session time |
| Endpoints | 1 IP = 1 forwarding slot | Unlimited endpoints generated |
| Expiry | Unused IPs never expire | 180 days (unlimited on Enterprise) |
| Auth | 9Proxy desktop app, local port forwarding | Username/password or IP whitelist |
| Setup | Desktop app required | Dashboard only |

For AdsPower specifically: if you're running accounts that live for weeks — ad accounts, marketplace seller profiles, social identities — the IP-based model is the straightforward fit, because you're renting addresses, not traffic. Each forwarded port becomes a profile's permanent-looking home.

If your AdsPower usage is mostly automation through the browser rather than long-lived account grooming, the GB model gets you there without installing anything on the host machine, and it plays better with cloud servers where you can't easily run a desktop app.

The awkward case is bursty work — a product launch or a campaign week where you need thirty profiles for four days and don't touch them again for a month. That's the bundle packages' territory, since the bundled traffic carries 180-day validity and doesn't punish you for uneven usage.

👉 [Compare 9Proxy residential proxy models](https://bit.ly/9-Proxy)

## Full plan list and current pricing

9Proxy raised prices on IP-based and bundle packages on 1 June 2026 — the first change in its history — while GB-based prices stayed exactly where they were. All numbers below are the post-adjustment rates.

### Residential proxies by IP (unlimited bandwidth, IPs don't expire)

| Package | Effective rate | Price | Buy |
| --- | --- | --- | --- |
| 100 IPs | $0.24/IP | $24 | [Get 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | $0.144/IP | $72 | [Get 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | $0.084/IP | $126 | [Get 1,500 IPs](https://bit.ly/9-Proxy) |
| 2,500 IPs | $0.084/IP | $210 | [Get 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | $0.072/IP | $360 | [Get 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | $0.048/IP | $720 | [Get 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | $0.035/IP | $863 | [Get 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | $0.029/IP | $1,438 | [Get 50,000 IPs](https://bit.ly/9-Proxy) |
| Business: 100,000 IPs | $0.023/IP | $2,300 | [Get Business tier](https://bit.ly/9-Proxy) |
| Business: 200,000 IPs | $0.021/IP | $4,140 | [Get Business tier](https://bit.ly/9-Proxy) |
| Business: 500,000 IPs | $0.018/IP | $8,625 | [Get Business tier](https://bit.ly/9-Proxy) |

### Residential proxies by GB (180-day validity)

| Package | Rate | Price | Validity | Buy |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00/GB | $15 | 180 days | [Get 5 GB](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10/GB | $105 | 180 days | [Get 55 GB](https://bit.ly/9-Proxy) |
| 100 GB | $1.50/GB | $150 | 180 days | [Get 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | $1.00/GB | $200 | 180 days | [Get 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80/GB | $800 | 180 days | [Get 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75/GB | $1,500 | 180 days | [Get 2,000 GB](https://bit.ly/9-Proxy) |
| Enterprise 3,000 GB | $0.72/GB | $2,160 | Unlimited | [See Enterprise plans](https://bit.ly/9-Proxy) |

Larger Enterprise tiers exist beyond 3,000 GB and keep the unlimited-validity term; the tier list above covers what's publicly posted.

### Bundle packages (IPs + traffic)

| Package | Contents | Price | Buy |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | [Get the Pro bundle](https://bit.ly/9-Proxy) |

Bundled traffic follows the same 180-day validity as the GB packages.

## How many IPs you actually need

The math is simpler than it looks, because IP-based packages don't expire. If you buy 100 IPs and burn through 20 a week, you have five weeks of supply — you're not losing anything to a monthly clock. So the question isn't "how many IPs per month," it's "how many profiles do I run in parallel at peak."

Rough guide for AdsPower users:

- **10–30 profiles**, side projects: the 100-IP pack is already more than you need, and the surplus sits there for later.
- **50–200 profiles**, small team: 500 IPs, or 1,000+500 if you're also testing new geos.
- **300+ profiles**, agency or reseller: the 5,000 tier is where the per-IP cost drops below $0.08, which changes the economics of reselling access to clients.
- **Scraping through AdsPower rather than account management**: ignore the IP tables and go GB-based. Per-request rotation is what you want there, and 100 GB at $150 covers a lot of SERP checks.

One anti-pattern worth naming: people buy 100 IPs, then point five profiles at the same IP to "save" them. In AdsPower that links the fingerprints you paid good money to separate. One IP per profile isn't a rule 9Proxy enforces — it's just how the math of multi-accounting works out.

## FAQ

**Does AdsPower support SOCKS5?** Yes — HTTP, HTTPS, SSH, and SOCKS5. HTTPS and SOCKS5 are the two you'll use with a 9Proxy residential endpoint.

**Can I paste the whole credential string somewhere and let AdsPower split it?** Yes, in the profile's Proxy tab, in the Host:Port field. Format: `host:port:username:password`. This is faster and less error-prone than filling four boxes by hand.

**Why does my proxy check fail when the credentials test fine elsewhere?** Local network restrictions, most often. Enable the system-proxy connection option in AdsPower's settings, or route your local tunnel globally.

**My proxy connects but shows the wrong country.** Check the IP checker selection first, then re-read the username string. Geo is encoded in the username for session-based endpoints, so a wrong `country-` or `city-` code follows you into every profile using that credential.

**Do unused 9Proxy IPs expire?** On the IP-based packages, no. On GB-based packages, traffic is valid for 180 days, or indefinitely on Enterprise.

**Does 9Proxy work with other anti-detect browsers?** It documents integrations for AdsPower, Dolphin Anty, and Multilogin, among others — the credential format is standard `host:port:user:pass`, so anything that accepts a SOCKS5 or HTTP endpoint will take it.

👉 [Start a 9Proxy account and wire it into AdsPower](https://bit.ly/9-Proxy)
