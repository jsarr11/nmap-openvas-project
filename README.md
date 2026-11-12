# Vulnerability Management Lab: Nmap & OpenVAS

A small, end-to-end vulnerability management exercise on a deliberately vulnerable host: discovery, assessment, prioritization, remediation and verification.

## Environment

| Component | Details |
|---|---|
| Scanner host | Ubuntu 24.04, Docker |
| Vulnerability scanner | OpenVAS / Greenbone Community Edition (Docker Compose) |
| Network scanner | Nmap 7.94 |
| Target | Metasploitable 2 (Docker container `tleemcjr/metasploitable2`, Ubuntu 8.04) |

The target container is attached to two Docker networks: the default bridge (`172.17.0.2`, scanned with Nmap from the host) and the Greenbone network (`172.18.0.18`, scanned by OpenVAS).

## Process

1. **Discovery**: Nmap service and OS detection (`nmap -sV -O`) to map exposed services and versions.
2. **Assessment**: OpenVAS scan with the *Full and fast* config (QoD ≥ 70%).
3. **Prioritization**: findings ranked by CVSS score, then by exploitability (backdoors and unauthenticated remote code execution first).
4. **Remediation**: fixed a selected set of critical findings on the target.
5. **Verification**: re-ran Nmap and OpenVAS against the same target and compared results.

## Initial findings

Nmap found **21 open TCP ports**, including FTP, Telnet, r-services, SMB, MySQL, PostgreSQL, VNC, IRC and Tomcat.

OpenVAS (scan duration 28 min):

| Critical | High | Medium | Low |
|---|---|---|---|
| 12 | 11 | 40 | 6 |

## Prioritization & remediation

Selected findings, ordered by risk:

| Finding | Port | CVSS | Why it was prioritized | Remediation |
|---|---|---|---|---|
| Ingreslock backdoor (root shell) | 1524 | 10.0 | Unauthenticated root shell | Disabled the service in xinetd |
| rexec / rlogin / rsh | 512–514 | 10.0 / 7.5 | Passwordless root login, cleartext | Disabled the services in xinetd |
| vsftpd 2.3.4 backdoor | 21, 6200 | 9.8 | Known backdoor, remote command execution | Disabled the FTP service |
| DistCC RCE (CVE-2004-2687) | 3632 | 9.3 | Unauthenticated remote command execution | Stopped the `distccd` process |
| PostgreSQL default credentials | 5432 | 9.0 | `postgres:postgres` login | Changed the `postgres` user password |

**Note:** the first attempt targeted `/etc/inetd.conf`, but `netstat -tlnp` showed the listeners were owned by **xinetd**. The service definitions were moved out of `/etc/xinetd.d/` and xinetd was reloaded (`kill -HUP`). Each change was verified with `netstat` before rescanning.

## Verification

Nmap after remediation: **15 open ports** (21, 512, 513, 514, 1524, 3632 and 6200 no longer listening).

OpenVAS after remediation (scan duration 22 min):

| | Critical | High | Medium | Low |
|---|---|---|---|---|
| Before | 12 | 11 | 40 | 6 |
| After | 5 | 8 | 38 | 6 |
| Change | −7 | −3 | −2 | 0 |

All targeted findings are absent from the second report.

## Remaining risk

Out of scope for this exercise and still open:

- **End-of-life OS** (Ubuntu 8.04): no vendor patches. The real fix is a rebuild on a supported OS.
- **Outdated web apps** on port 80 (TWiki, PHP-CGI, phpMyAdmin) and **Tomcat AJP "Ghostcat"** (8009).
- **dRuby RCE** (8787), **Java RMI** (1099), **UnrealIRCd** (6697), **VNC weak password** (5900).
- Weak TLS/SSH configuration (SSLv3, RC4, 1024-bit keys, CBC ciphers).

## Repository structure

```
scans/        Nmap output (before / after)
reports/      OpenVAS PDF reports (before / after)
screenshots/  Greenbone UI and terminal screenshots
docker-compose.yml  Greenbone Community Edition setup
```

## Reproduce

```bash
# Scanner
docker compose -f docker-compose.yml up -d

# Target
docker run -dit --name metasploitable tleemcjr/metasploitable2 sh -c "/bin/services.sh && bash"
docker network connect greenbone-community-edition_default metasploitable

# Discovery
sudo nmap -sV -O <target-ip> -oN scans/nmap_scan.txt
```

> Metasploitable is intentionally vulnerable. Keep it on an isolated network.