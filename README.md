# dehydrated 

![](docs/logo.jpg)

Dehydrated is a client for signing certificates with an ACME-server (e.g. Let's Encrypt) implemented as a relatively simple (zsh-compatible) bash-script.
This client supports both ACME v1 and the new ACME v2 including support for wildcard certificates!

It uses the `openssl` utility for everything related to actually handling keys and certificates, so you need to have that installed.

Other dependencies are: cURL, sed, grep, awk, mktemp (all found pre-installed on almost any system, cURL being the only exception).

Current features:
- Signing of a list of domains (including wildcard domains!)
- Signing of a custom CSR (either standalone or completely automated using hooks!)
- Renewal if a certificate is about to expire or defined set of domains changed
- Certificate revocation

Please keep in mind that this software, the ACME-protocol and all supported CA servers out there are relatively young and there might be a few issues. Feel free to report any issues you find with this script or contribute by submitting a pull request,
but please check for duplicates first (feel free to comment on those to get things rolling).

## Getting started

For getting started I recommend taking a look at [docs/domains_txt.md](docs/domains_txt.md), [docs/wellknown.md](docs/wellknown.md) and the [Usage](#usage) section on this page (you'll probably only need the `-c` option).

Generally you want to set up your WELLKNOWN path first, and then fill in domains.txt.

**Please note that you should use the staging URL when experimenting with this script to not hit Let's Encrypt's rate limits.** See [docs/staging.md](docs/staging.md).

If you have any problems take a look at our [Troubleshooting](docs/troubleshooting.md) guide.

## Config

dehydrated is looking for a config file in a few different places, it will use the first one it can find in this order:

- `/etc/dehydrated/config`
- `/usr/local/etc/dehydrated/config`
- The current working directory of your shell
- The directory from which dehydrated was run

Have a look at [docs/examples/config](docs/examples/config) to get started, copy it to e.g. `/etc/dehydrated/config`
and edit it to fit your needs.

## Usage:

```text
Usage: ./dehydrated [-h] [command [argument]] [parameter [argument]] [parameter [argument]] ...

Default command: help

Commands:
 --version (-v)                   Print version information
 --register                       Register account key
 --account                        Update account contact information
 --cron (-c)                      Sign/renew non-existent/changed/expiring certificates.
 --signcsr (-s) path/to/csr.pem   Sign a given CSR, output CRT on stdout (advanced usage)
 --revoke (-r) path/to/cert.pem   Revoke specified certificate
 --cleanup (-gc)                  Move unused certificate files to archive directory
 --help (-h)                      Show help text
 --env (-e)                       Output configuration variables for use in other scripts

Parameters:
 --accept-terms                   Accept CAs terms of service
 --full-chain (-fc)               Print full chain when using --signcsr
 --ipv4 (-4)                      Resolve names to IPv4 addresses only
 --ipv6 (-6)                      Resolve names to IPv6 addresses only
 --domain (-d) domain.tld         Use specified domain name(s) instead of domains.txt entry (one certificate!)
 --alias certalias                Use specified name for certificate directory (and per-certificate config) instead of the primary domain (only used if --domain is specified)
 --keep-going (-g)                Keep going after encountering an error while creating/renewing multiple certificates in cron mode
 --force (-x)                     Force renew of certificate even if it is longer valid than value in RENEW_DAYS
 --no-lock (-n)                   Don't use lockfile (potentially dangerous!)
 --lock-suffix example.com        Suffix lockfile name with a string (useful for with -d)
 --ocsp                           Sets option in CSR indicating OCSP stapling to be mandatory
 --privkey (-p) path/to/key.pem   Use specified private key instead of account key (useful for revocation)
 --config (-f) path/to/config     Use specified config file
 --hook (-k) path/to/hook.sh      Use specified script for hooks
 --out (-o) certs/directory       Output certificates into the specified directory
 --alpn alpn-certs/directory      Output alpn verification certificates into the specified directory
 --challenge (-t) http-01|dns-01  Which challenge should be used? Currently http-01 and dns-01 are supported
 --algo (-a) rsa|prime256v1|secp384r1 Which public key algorithm should be used? Supported: rsa, prime256v1 and secp384r1
```

## Install

`apt install joe mc git dnsutils bind9`
`cd opt`
`git clone https://github.com/arthur7373/dehydrated`
`cd dehydrated/mkdir /etc/dehydrated/`
`nano /etc/dehydrated/config`
Add below lines:

```
CHALLENGETYPE="dns-01"
DOMAINS_TXT="${BASEDIR}/domains.txt"
```

Create domain file to get wildcard certificate '*.<domain>' with an alternative name '<domain>'

`nano /etc/dehydrated/domains.txt`

```
# Create a certificate for '*.<domain>' with an alternative name '<domain>'
# and store it in the directory ${CERTDIR}/star.<domain>
# NOTE: It INCLUDES both wildcard certificate for subdomains
# and the plain domian name
#*.example.net example.net > star.example.net
```

Create Hook script using for _dns 01_ challenge via BIND9
This hook script uses the nsupdate utility from the bind package to solve dns-01 challenges.

nano /opt/dehydrated/hook2

# Code

```bash
#!/usr/bin/env bash

#
# Example how to deploy a DNS challenge using nsupdate
#

set -e
set -u
set -o pipefail

NSUPDATE="nsupdate -k /opt/dehydrated/Kdnsupdatekey.private"
DNSSERVER="127.0.0.1"
TTL=300

case "$1" in
    "deploy_challenge")
        printf "server %s\nupdate add _acme-challenge.%s. %d in TXT \"%s\"\nsend\n" "${DNSSERVER}" "${2}" "${TTL}" "${4}" | $NSUPDATE
        ;;
    "clean_challenge")
        printf "server %s\nupdate delete _acme-challenge.%s. %d in TXT \"%s\"\nsend\n" "${DNSSERVER}" "${2}" "${TTL}" "${4}" | $NSUPDATE
        ;;
    "deploy_cert")
        # optional:
        # /path/to/deploy_cert.sh "$@"
        ;;
    "unchanged_cert")
        # do nothing for now
        ;;
    "startup_hook")
        # do nothing for now
        ;;
    "exit_hook")
        # do nothing for now
        ;;
esac

exit 0
```

chmod +x hook2
ln -s  /opt/dehydrated/hook2 /usr/local/bin/hook2


nano /opt/dehydrated/Kdnsupdatekey.private


The file `/opt/dehydrated/Kdnsupdatekey.private` looks like this:

```
key "<keyname>" {
  algorithm hmac-sha512;
  secret "<key>";
};
```

To avoid making your entire production DNS subject to dynamic DNS updates, then for each certificate domain you want. This is a secure approach because each host will have its own key, and hence can only obtain certificates for those domains you have explicitly authorized it for. Use /dev/random as an argument for dnssec-keygen for key generation to increase security further.

1. In your main DNS infrastructure create a delegation: `_acme-challenge.<domain>. NS <your-nameserver>.`
2. Create a new zone `_acme-challenge.<domain>` on `<your-nameserver>`, with an empty zonefile (just an SOA and NS record), writeable by the nameserver

`nano /etc/bind/_acme-challenge.<domain>`


```
$ORIGIN .
$TTL 604800     ; 1 week
_acme-challenge.<domain> IN SOA ns.<domain>. root.<domain>. (
                                27         ; serial
                                604800     ; refresh (1 week)
                                86400      ; retry (1 day)
                                2419200    ; expire (4 weeks)
                                604800     ; minimum (1 week)
                                )
                        NS      ns.<domain>.
```
`chown bind:bind /etc/bind/_acme-challenge.<domain>`
`systemctl restart bind9`

3. Create a new TSIG key: 
```
dnssec-keygen -r /dev/urandom -a hmac-sha512 -b 512 -n HOST <keyname>
```

4. Enable dynamic updates on the `_acme-challenge.<domain>` zone with this key

e.g. for bind9:

~~~
key "<keyname>" {
  algorithm hmac-sha512;
  secret "<key>";
};
zone "_acme-challenge.example.net" {
  type master;
  file "/etc/bind/_acme-challenge.example.net";
  masterfile-format text;
  allow-update { key "<keyname>"; };
};

~~~




## Intital run
`dehydrated --register --accept-terms`

`dehydrated  -c -k hook2 --register --accept-terms`

## Manual run

Run
`dehydrated  -c -k hook2`

Force gettting new certificates even if valid
`dehydrated  -c -k hook2 -x`
