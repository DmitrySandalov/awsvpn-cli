# awsvpn-cli

Headless AWS Client VPN client for **SAML-federated** endpoints (for example
Entra ID sign-in). Python 3 stdlib driving `openvpn3`: no pip packages, no GUI,
**no sudo**.

Verified end to end on **Linux** and on **Android** (Termux plus OpenVPN for
Android from Google Play); see [Android](#android-termux--openvpn-for-android).

For **Azure** point-to-site gateways with Entra ID sign-in, see the sibling
project [azvpn-cli](https://github.com/DmitrySandalov/azvpn-cli). It has the same
design: stdlib Python, openvpn3 on Linux, Termux plus OpenVPN for Android on
the phone.

> **Unaffiliated** with Amazon Web Services, Microsoft or OpenVPN Inc. "AWS",
> "Entra" and "OpenVPN" are trademarks of their respective owners. The protocol
> details here were measured against live Client VPN endpoints, for
> interoperability. Use at your own risk.

**Nothing organisation-specific is embedded.** You supply the endpoint's own
`.ovpn` with `--profile`.

## Contents

- [Why](#why)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Usage](#usage)
- [Split DNS](#split-dns)
- [Android (Termux + OpenVPN for Android)](#android-termux--openvpn-for-android)
- [Files it writes](#files-it-writes)
- [Verified / not verified](#verified--not-verified)
- [Gotchas](#gotchas)
- [Troubleshooting](#troubleshooting)

## Why

SAML sign-in on AWS Client VPN is an OpenVPN *dynamic challenge* that only the
AWS VPN Client implements, and that client exists for Windows, macOS and Ubuntu
x86_64 only: a GUI on Linux, nothing on Android. Stock `openvpn` 2.x cannot
finish the sign-in either: the final password is about 10 kB and its
control-channel buffer is 2048 bytes.

`awsvpn-cli` does the sign-in itself and hands the result to `openvpn3`.

## How it works

Two connections to **the same gateway IP**:

| Step | What happens |
|---|---|
| Leg 1 | Connect with username `N/A`, password `ACS::35001`. The gateway refuses with `AUTH_FAILED,CRV1:R:<sid>:<user>:<sign-in URL>` |
| Sign-in | The browser opens the URL. The IdP posts the SAML response to `http://127.0.0.1:35001`, where this tool listens |
| Leg 2 | Connect again with password `CRV1::<sid>::<SAMLResponse>`. The gateway lets you in |

Leg 1 is a minimal OpenVPN control channel written in Python (UDP or TCP, TLS
through `ssl.MemoryBIO`). It verifies the gateway certificate against the
profile's `<ca>`, `verify-x509-name` and the serverAuth key usage, and stops as
soon as the challenge arrives: no tunnel, no root, no `openvpn` binary.

Leg 2 is a real `openvpn3` session. The state id is bound to the gateway node
that issued it, and an endpoint resolves to one address per associated subnet,
so the tool resolves once and pins that IP for both legs.

## Requirements

- Python 3.9+
- **`openvpn3`** (openvpn3-linux), required. It is not in the Ubuntu/Debian
  archives; add OpenVPN Inc.'s repository first:
  <https://openvpn.net/as-docs/openvpn3-linux.html>
- No sudo: openvpn3 runs the tunnel through its own D-Bus service.
- The endpoint's client configuration, from the Client VPN console or:

  ```bash
  aws ec2 export-client-vpn-client-configuration \
    --client-vpn-endpoint-id cvpn-endpoint-0123456789abcdef0 --output text > vpn.ovpn
  ```

  The profile must contain `auth-federate`; certificate- or AD-authenticated
  endpoints are not handled.

## Usage

```bash
awsvpn-cli check  --profile ~/vpn/dev.ovpn   # leg 1 only: reachability, cert, challenge
awsvpn-cli up     --profile ~/vpn/dev.ovpn   # sign in + connect
awsvpn-cli status --profile ~/vpn/dev.ovpn
awsvpn-cli logs   --profile ~/vpn/dev.ovpn
awsvpn-cli down   --profile ~/vpn/dev.ovpn
```

To skip `--profile`, set `AWSVPN_PROFILE` or place the file at
`~/.config/awsvpn-cli/profile.ovpn`. The connection name is the file name
(override with `--name`), so several endpoints can be up at once as long as
their client CIDRs do not overlap.

| Command | Does |
|---|---|
| `check` | Leg 1 only. Proves reachability, the certificate pins and that the endpoint is SAML. No sign-in, no tunnel |
| `gen` | Render the openvpn3 config and stop |
| `up` | Sign in, render, import into openvpn3, connect |
| `down` | Disconnect |
| `status` | Profile, tunnel, interface, route count |
| `logs` | Attach to the openvpn3 session log |
| `password` | Sign in and print the single-use leg-2 password, for another client |

On Android, `up` signs in and hands the profile to OpenVPN for Android instead;
the app owns the tunnel, so `down` and `logs` refuse to run there.

`--dns-domain` (repeatable) or `AWSVPN_DNS_DOMAINS` adds routing domains; see
below.

## Split DNS

The endpoint pushes a DNS server (the VPC resolver) but **no domains**. With
`--dns-scope tunnel` alone, nothing would be routed to it, and AWS private
endpoints, which keep their public names, would resolve to their public
addresses. The EKS API of a private cluster then times out.

`awsvpn-cli` declares routing domains as client-side `dhcp-option DOMAIN`
lines and sets `--dns-scope tunnel`. By default, for the endpoint's region:

```
<region>.eks.amazonaws.com
<region>.rds.amazonaws.com
<region>.compute.internal
```

Add private hosted zones with `--dns-domain example.internal`. Everything else
keeps your normal DNS.

## Android (Termux + OpenVPN for Android)

Android has no openvpn3 CLI. The tunnel is run by **OpenVPN for Android**
(ics-openvpn, Google Play) with its **OpenVPN 3 core**; awsvpn-cli runs in
[Termux](https://termux.dev) and does the sign-in.

Requirements: Termux with `pkg install python termux-tools` (`termux-tools`
is usually there already), and OpenVPN for Android with **Settings >
"OpenVPN 3 Core"** checked (it is unchecked by default). Keep the profile in
the Termux home, not on `/sdcard`.

Each connect, **in the Termux app** (not over SSH: Android only lets the app in
front open another app):

```bash
awsvpn-cli up --profile ~/vpn/dev.ovpn
```

1. The phone browser opens the sign-in; Entra posts the response to Termux on
   port 35001.
2. Switch back to Termux and press Enter. Termux opens the chooser; pick
   OpenVPN for Android and import with the dialog defaults.
3. Tap the new profile to connect, within a few minutes of signing in.
4. Next time, delete the old profile in the app and run `up` again: the
   password inside it is spent.

### Why the password goes inline, and never through a text field

The leg-2 password is about 9.7 kB. Android limits a single-line `EditText`
with no explicit `maxLength` to **5000 characters**, and the app's connect
dialog and profile editor are such fields. Measured with a stand-in server on
the phone and a known 9694-character password:

| Path into the app (0.7.69, core `3.12_git`) | Received by the server |
|---|---|
| Pasted into the connect dialog | first 5000 characters |
| Inline `<auth-user-pass>`, profile opened in the editor before connecting | first 5000 characters |
| Inline `<auth-user-pass>`, connected straight after import | all 9694 characters |
| openvpn3-linux 27.1 (core 3.11.7) | all 9694 characters |

The gateway answers a cut password with `AUTH_FAILED: Invalid username or
password`. So `up` renders the profile with username and password inline, the
app's importer copies them into the profile untouched, and the core receives
them whole. Opening the profile's editor, even without changing anything,
saves the password back through its field and cuts it.

Shrinking the SAML response is no way around it: a sample was 7220 bytes of
XML (9628 in base64), with two 2174-byte signatures and nine attributes.
Signing the assertion only and dropping the name claims still leaves about
5700 characters.

The import also drops `cipher` (it keeps it only in compatibility modes), so
the rendered profile declares `data-ciphers AES-256-GCM:AES-128-GCM`, which the
app keeps.

DNS while connected: Android sends every lookup to the pushed VPC resolver,
so private names resolve without the routing domains.

## Files it writes

| Path | Mode | Contents |
|---|---|---|
| `~/.config/awsvpn-cli/rendered/<name>.ovpn` | 0600 | rendered config: pinned IP, routing domains, the profile's CA |
| `~/.config/awsvpn-cli/rendered/<name>.ovpn` (Android) | 0600 | as above plus the single-use password inline; deleted 120 s after the hand-over |

On Linux the SAML response and the leg-2 password never touch the disk: the
password goes to `openvpn3 session-start` on stdin. On Android it has to travel
in the profile, which lives for two minutes; the copy the app keeps is spent
once used. The profile itself holds
no secret (the endpoint name and the public CA), but keep it to the people who
need it.

## Verified / not verified

Against four live endpoints, all UDP 443 and split tunnel: three
VPC-associated endpoints signing in through an Entra ID SAML app, and one
transit-gateway endpoint signing in through IAM Identity Center (public ACM
server certificate). Clients: Ubuntu 22.04 with openvpn3-linux 27.1, and
Android with Termux and OpenVPN for Android 0.7.69 (Play Store). Nothing in
the tool is IdP-specific: the browser goes wherever the challenge points.

- ✅ leg 1 in Python: CA, CN pin and serverAuth checked; a wrong CA and a wrong
  CN are refused
- ✅ leg 1 from Termux (Python 3.13, Android), on Wi-Fi and on mobile data
- ✅ sign-in through the browser, SAML response caught on 127.0.0.1:35001
- ✅ leg 2 through openvpn3: pushed route and DNS, `Client connected`
- ✅ split DNS: a private EKS API name resolves to its VPC address and answers
- ✅ `down` idempotent, `tun0` removed
- ✅ IAM Identity Center endpoint, on Ubuntu and Android: sign-in, connect,
  routes into the attached VPCs. Names resolve through the endpoint's DNS
  servers. Through a hub resolver, an EKS API name gives the public address
  while the cluster's public endpoint is on, and the private one once it is
  off. Until then, point kubectl at the private address and keep TLS on the
  real name (`server: https://<private-ip>` plus `tls-server-name: <api-name>`
  in the kubeconfig), which works without root on Android too; on Linux an
  `/etc/hosts` entry also does
- ✅ Android leg 2 with OpenVPN for Android 0.7.69 (Play Store), OpenVPN 3
  core: connected; a private EKS API name resolves to its VPC address and answers
- ⬜ TCP endpoints: implemented, not tested against a live endpoint

## Gotchas

- **The leg-2 password is single use.** openvpn3 replays it on a reconnect and
  the gateway refuses it, so a dropped tunnel (Wi-Fi change, suspend) does not
  recover by itself. `status` notes it; run `down` and `up`.
- **Both legs must hit one gateway IP.** An endpoint can resolve to several
  addresses. The tool resolves once and pins that IP into the rendered config.
- **The username is the literal `N/A`**, on both legs. Your identity travels
  in the SAML response.
- **Port 35001 must be free during sign-in.** The AWS VPN Client uses it too;
  quit it first.
- **Client CIDRs must not overlap** if you keep several endpoints up at once.
- **Android: the app may stick at "Building configuration..."** right after an
  import, logging `Used x 101 tries to get current version (-1/...)`: its VPN
  process has not seen the new profile yet. Force-stop the app, reopen it, and
  tap the profile again.
- **Android: run `up` in the Termux app, not over SSH.** From the background,
  `termux-open` returns success and shows nothing.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `not a SAML (federated) Client VPN profile` | no `auth-federate` in the profile | export the federated endpoint's configuration |
| `cannot bind 127.0.0.1:35001` | AWS VPN Client running | quit it |
| `leg 1 ... no answer from the gateway in time` | UDP 443 blocked, or wrong address | `check`; try another network |
| `gateway cert CN is ..., the profile pins ...` | profile from another endpoint, or the server cert changed | re-export the profile |
| `CERTIFICATE_VERIFY_FAILED` | the profile's `<ca>` does not match | re-export the profile |
| IdP shows `AADSTS50105` | not assigned to the IdP application | ask for access |
| `session-start did not report a connection` | sign-in took too long, or the password reached another node | run `up` again |
| Connected, private names resolve to public IPs | domain not routed to the VPC resolver | `resolvectl domain tun0`; add it with `--dns-domain` |
| Endless reconnect after a network change | single-use password replayed | `down`, then `up` |
| Android: `AUTH_FAILED: Invalid username or password` | password cut to 5000 characters by a text field | delete the profile, run `up`, connect without opening the editor or a password dialog |
| Android: no chooser after the sign-in | Termux was in the background | press Enter in the Termux app; never run `up` over SSH |
