# inetsim-py

A modern **Python 3 network-service simulator for malware-analysis labs**, inspired
by the original Perl-based INetSim. Run it on a Linux server, point an isolated
Windows or Linux client at it, and record network activity while returning
controlled local responses.

This rewrite was built to address the DNS instability encountered with Perl
INetSim in this lab and add capabilities needed for current analysis workflows:
asyncio DNS, HTTPS certificates and capture, Tor application-interface emulation,
a modern simulated web login page, and structured evidence. The recorded DNS
workload completed **120,000 queries with zero failures**. This measures this
implementation; a controlled comparison against Perl INetSim has not been performed.

## Features

| Area | Capabilities |
| --- | --- |
| DNS | UDP/TCP wildcard sinkholing, invented names, reusable/pipelined TCP connections, IPv4/IPv6 answers, configured records and DNS-over-HTTPS |
| HTTP/HTTPS | Configurable routes, keep-alive, bounded request-body capture, response hashes and harmless replacement downloads |
| TLS and MITM capture | Built-in lab CA, per-SNI certificates, signed CRL publication, and optional explicit HTTP/HTTPS interception through mitmproxy |
| Tor interfaces | SOCKS5/SOCKS4a onion HTTP/HTTPS responses, target logging and a Tor control-port emulator |
| Modern web response | Customizable enterprise sign-in/admin-style landing page with configurable branding and nginx-style headers |
| Other services | SMTP/STARTTLS, POP3/STLS, passive FTP/FTPS, controlled SMB exchanges, NTP, TFTP, QUIC UDP sink and TCP/UDP responders |
| Evidence | JSONL protocol logs, SHA-256-addressed artifacts, redacted protocol authentication fields and shutdown session reports |
| Lab controls | Client CIDR allow-lists, interface selection, optional local DNS/CA management and scoped transparent web redirects |

The web page replaces the familiar legacy simulator landing-page presentation
and avoids obvious INetSim strings in its generated HTML. This reduces simple
page-string fingerprints; it does not make a simulated network undetectable.
It is a **simulated response page**, with configuration managed through TOML and
the CLI. See [web responses and fingerprinting](docs/web-responses.md).

Tor support emulates the application's SOCKS/control interfaces. Real Tor
circuits, directory authorities and public onion services require a separate
Tor network. See [Tor setup and limits](docs/tor-setup.md).

## Requirements and installation

The tested server platform is **REMnux on Ubuntu 24.04**. The core requires
Python **3.11+** and has no third-party Python runtime dependencies. Automatic
certificate generation and CRLs require the `openssl` executable. Linux network
helpers use tools such as `ip`, `iptables` and, when requested, `resolvectl`.
Windows has been tested as a client; running the server on Windows has not been
validated.

On Ubuntu/REMnux, install the basic prerequisites, then run from the downloaded
project directory:

```bash
sudo apt update
sudo apt install python3 python3-venv openssl iproute2
cd inetsim
python3 -m venv .venv
.venv/bin/python -m pip install -e .
.venv/bin/inetsim-py check-config
.venv/bin/inetsim-py run
```

The default configuration binds to loopback with test ports such as DNS `1053`,
HTTP `18080` and HTTPS `18443`. Stop with Ctrl-C before starting a lab profile.
The source can also run directly with `python3 inetsim.py run`.

Install optional capabilities before isolating the lab:

```bash
# Explicit HTTP/HTTPS MITM proxy; the tested proxy setup uses Python 3.12.
.venv/bin/python -m pip install -e '.[mitm]'

# DNS workload validator and pytest.
.venv/bin/python -m pip install -e '.[validation,dev]'
```

`requirements.txt` installs the project with the mitmproxy extra. For an offline
installation, use the wheelhouse procedure in [lab setup](docs/lab-setup.md).

## Configure an isolated lab

Example topology:

```text
Windows or Linux client                 Linux simulator
10.10.20.10/24  ---------------------->  10.10.20.5/24
DNS: 10.10.20.5                          inetsim-py
          One private VM network; no external uplink
```

Use an internal VM network for a detonation lab. The prepared Windows lab routes
the client's off-subnet traffic to REMnux for interception and capture; follow
the [Windows routing guide](docs/windows-client-lab.md) for that setup. DNS
sinkholing alone does not intercept hard-coded IPs or a localhost Tor proxy.

Copy the sample configuration:

```bash
cp configs/lab-subnet.toml configs/my-lab.toml
```

Edit the **existing sections** in `configs/my-lab.toml` to match your server and
subnet; do not append duplicate TOML tables. For the example topology:

```toml
[global]
listen_host = "10.10.20.5"
allowed_clients = ["10.10.20.0/24"]
tls_crl_url = "http://10.10.20.5/pki/lab-ca.crl"

[services.dns]
sinkhole_ip = "10.10.20.5"
sinkhole_ipv4 = "10.10.20.5"

[services.https]
prewarm_hosts = ["example.test"]
block_on_unknown_sni = true
```

The CRL URL must be reachable by clients over HTTP. Prewarming or synchronous
first-seen SNI generation lets strict clients validate the first connection;
synchronous generation briefly blocks the event loop. The sample profile uses
standard service ports:

| Service | Lab port |
| --- | --- |
| DNS | UDP/TCP 53 |
| HTTP / HTTPS | TCP 80 / 443 |
| Tor SOCKS / control emulators | TCP 9050 / 9051 |
| Optional mitmproxy | TCP 8080; disabled until enabled in the config |

Validate and run from the project directory. Replace `enp0s3` with the lab
interface:

```bash
.venv/bin/inetsim-py --config configs/my-lab.toml check-config
sudo .venv/bin/inetsim-py --interface enp0s3 --config configs/my-lab.toml run
```

`--interface` overrides the listen address, allowed CIDR and DNS sinkhole address
for that run. It does not change the configured CRL URL, static DNS records, IP
addresses, client DNS or firewall. Keep those settings consistent with the
selected interface. Low service ports require elevated privileges. If a listener
is occupied, inspect `sudo ss -lntup`; another INetSim instance may already be
running on REMnux.

Set the client's DNS server to the simulator IP. Configure routing, packet
capture and containment separately as described in the lab guides. The prepared
Windows lab has no external route from REMnux, IPv4 forwarding disabled and
unsupported off-subnet traffic blocked.

## TLS trust and MITM capture

Start the server once to generate its CA. Copy only its **public certificate**
`artifacts/tls/inetsim-ca.crt` and the appropriate helper from `scripts/` to the
lab client.

Linux client:

```bash
sudo bash install-ca-linux.sh install inetsim-ca.crt inetsim-https-lab
curl --noproxy '*' https://example.test/
```

Windows client, from Administrator PowerShell:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\install-ca-windows.ps1 -Action Install -CertPath .\inetsim-ca.crt -Name inetsim-https-lab
curl.exe --noproxy "*" https://example.test/
```

With signed CRL publication enabled, the tested Windows Schannel curl client
uses normal certificate and revocation validation without an offline-revocation
bypass. Windows caches certificate-chain and CRL results; see the
[certificate guide](docs/certificates.md) for renewal, revocation and cache tests.
Pinned clients still require a matching certificate/key or client-specific
configuration.

For explicit proxy capture, install the mitmproxy extra and set
`services.mitmproxy.enabled = true`. Configure the client proxy as
`10.10.20.5:8080`, then install the **separate** public mitmproxy CA from
`artifacts/mitmproxy/conf/mitmproxy-ca-cert.pem`. Upstream access is blocked by
default; requests receive local synthetic responses. See
[certificate setup](docs/certificates.md) and [features](docs/features.md).

## Modern web page and response profiles

Customize these keys in the existing `[services.http]` section:

```toml
server_header = "nginx/1.24.0"
login_page_enabled = true
login_page_path = "/"
login_page_brand = "Northwind"
login_page_title = "Administration sign in"
login_page_realm = "Authorized staff access"
```

HTTP and HTTPS share the response profile. Exact routes, status codes, download
extensions and extensionless download paths are configurable. The generated
page is self-contained and needs no external assets. A different-looking page
cannot prevent every environment check: WannaCry's observed kill-switch check
was satisfied by a successful HTTP connection, independent of page branding.

## What was tested

Recorded results from **2026-10-18**. INetSim ran on Linux throughout the native
Windows tests. Counts below refer to distinct passes, not one combined run.

| OS / role | Test | Result |
| --- | --- | --- |
| Ubuntu 24.04.5 host, Python 3.12.3 | Latest source regression suite | **216 passed, 9 skipped** |
| REMnux / Ubuntu 24.04, Python 3.12.3 | Latest deployed regression suite | **215 passed, 10 skipped** |
| REMnux server + separate REMnux client | DNS load at concurrency 32 | **100,000 UDP + 20,000 TCP queries; zero failures** |
| REMnux server + separate REMnux client | Expanded protocol matrix | **31 groups passed** |
| REMnux server + separate REMnux client | SOCKS/onion application workflow | **23 checks; 1,000 sessions, including 250 TLS; zero failures** |
| Windows 11 Pro, Tiny11 B1 build 22621.525; Python 3.12.10 x64/x86 | Native protocol and routing checks | **17 protocol groups and 12 routing checks passed** |
| Same native Windows client | Harmless downloads and execution across three runs | **45 exact download matches; 33 executions passed; 12 execution skips** |
| Same native Windows client | SOCKS/onion application workflow | **22 checks; 400 sessions, including 100 TLS; zero failures** |
| Same native Windows client after CRL update | Default curl, certutil, valid/revoked leaves and public-key pins | **4 positive/pin checks and 2 revoked-leaf checks passed; 12 routing checks passed after reboot** |
| Disposable Windows clone | Supplied PMAT WannaCry sample | **DNS + HTTP kill-switch contact captured; sample exited after 0.625 s** |

Earlier Linux-host Wine tests also exercised Windows payload formats. They are
reported separately from the native Windows results. Skips and historical test
counts are explained in [testing results](docs/testing-results.md).

The WannaCry run received HTTP 200 for its kill-switch request and exited with
code 0. Its execution-window capture contains 11 packets. It did not reach a
Tor or TLS stage in the observation window; this is not an encryption or real
Tor bootstrap validation. See [sample evidence and explanation](docs/compatibility-results.md).

## Repeat checks and inspect evidence

From a separate client with the project installed and the public lab CA trusted:

```bash
.venv/bin/python scripts/validate-lab.py 10.10.20.5 --ca inetsim-ca.crt --system-trust
.venv/bin/python -m pytest -q
```

The first command probes the running server; pytest tests the local source
checkout. The full protocol, DNS workload, download and Tor validators need
settings documented in [validation](docs/validation.md).
The native Windows revocation validator is documented in
[certificate setup](docs/certificates.md).

On the server:

```bash
tail -f logs/dns.jsonl logs/http.jsonl logs/https.jsonl logs/tor_socks.jsonl
```

Logs correlate requests, response hashes and captured artifact paths. A GET log
proves a download was served; client-side observation is needed to establish
execution. Packet capture is a separate lab service, not a built-in recorder.

## Scope and documentation

Use this project on isolated analysis networks. Emulation covers selected
protocol behavior: SMB has no real shares, signing or encryption; the QUIC
service is a sink; Office/MSI replacements are download containers; there is no
DHCP service or automatic complete gateway setup. CA trust, application pins,
revocation policy and protocol-specific checks still affect compatibility.

- [Documentation index](docs/README.md)
- [Installation and offline lab setup](docs/lab-setup.md)
- [Windows client routed through REMnux](docs/windows-client-lab.md)
- [DNS configuration](docs/dns-setup.md)
- [TLS certificates, CRLs and MITM](docs/certificates.md)
- [Web responses and fingerprinting](docs/web-responses.md)
- [Downloads and execution](docs/download-workflow.md)
- [Tor configuration and limits](docs/tor-setup.md)
- [OS coverage and testing results](docs/testing-results.md)
- [Validation commands](docs/validation.md)
- [Revocation, pinning and WannaCry evidence](docs/compatibility-results.md)
- [CLI reference](docs/cli-reference.md)
- [Troubleshooting](docs/troubleshooting.md)
- [Architecture](docs/architecture.md)
