*Originally a product of w2c/letsencrypt-esxi. Modified for those of us that are either unable or unwilling to expose our ESXi management interfaces to the Internet.*

# Let's Encrypt for VMware ESXi

`acme-esxi` is a lightweight open-source solution to automatically obtain and renew Let's Encrypt or private ACME CA certificates on standalone VMware ESXi servers. Packaged as a _VIB archive_ or _Offline Bundle_, install/upgrade/removal is possible directly via the web UI or, alternatively, with just a few SSH commands.

Features:

- **Fully-automated**: Requesting and renewing certificates without user interaction
- **Auto-renewal**: A cronjob runs once a week to check if a certificate is due for renewal
- **Persistent**: The certificate, private key and all settings are preserved over ESXi upgrades
- **Configurable**: Customizable parameters for renewal interval, Let's Encrypt (ACME) backend, etc
- **Can be used with any ACME CA**: [LabCA](https://github.com/hakwerk/labca) is a great example.

_Successfully tested with all currently supported versions of ESXi (6.5, 6.7, 7.0)._

## Why?

Many ESXi servers are accessible over the Internet and use self-signed X.509 certificates for TLS connections. This situation not only leads to annoying warnings in the browser when calling the Web UI, but can also be the reason for serious security problems. Despite the enormous popularity of [Let's Encrypt](https://letsencrypt.org) and ACME, there is no convenient way to automatically request, renew or remove certificates in ESXi.

*No user should get used to ignoring a certificate warning from any browser, even self-signed or local.*

## Prerequisites

Before installing `acme-esxi`, ensure the following preconditions are met:

- Your server is publicly reachable over the Internet
- A _Fully Qualified Domain Name (FQDN)_ is set in ESXi. Something like `localhost.localdomain` will not work
- The hostname you specified can be resolved via A and/or AAAA records in the corresponding DNS zone
- If you're running a private CA, the CA is reachable and you have any certificates you may need bundled.

**Note:** As soon as you install this software, any existing, non Let's Encrypt certificate gets replaced!

## Install

`acme-esxi` can be installed via SSH or the Web UI (= Embedded Host Client).

### SSH on ESXi

```bash
$ wget -O /tmp/acme-esxi.vib https://github.com/NateTheSage/acme-esxi/releases/latest/download/acme-esxi.vib

$ esxcli software vib install -v /tmp/acme-esxi.vib -f
Installation Result
   Message: Operation finished successfully.
   Reboot Required: false
   VIBs Installed: natethesage_bootbank_acme-esxi_1.0.0-0.0.0
   VIBs Removed:
   VIBs Skipped:

$ esxcli software vib list | grep acme
acme-esxi  1.0.0-0.0.0  natethesage  CommunitySupported  2022-05-29

$ cat /var/log/syslog.log | grep acme
2022-05-29T20:01:46Z /etc/init.d/acme-esxi: Running 'start' action
2022-05-29T20:01:46Z /opt/acme-esxi/renew.sh: Starting certificate renewal.
2022-05-29T20:01:46Z /opt/acme-esxi/renew.sh: Existing cert for example.com not issued by Let's Encrypt. Requesting a new one!
2022-05-29T20:02:02Z /opt/acme-esxi/renew.sh: Success: Obtained and installed a certificate from Let's Encrypt.
```

### Web UI (= Embedded Host Client)

1. _Storage -> Datastores:_ Use the Datastore browser to upload the [VIB file](https://github.com/acme/acme-esxi/releases/latest/download/acme-esxi.vib) to a datastore path of your choice.
2. _Manage -> Security & users:_ Set the acceptance level of your host to _Community_.
3. _Manage -> Packages:_ Switch to the list of installed packages, click on _Install update_ and enter the absolute path on the datastore where your just uploaded VIB file resides.
4. While the VIB is installed, ESXi requests a certificate from Let's Encrypt. If you reload the Web UI afterwards, the newly requested certificate should already be active. If not, see the [Wiki](https://github.com/acme/acme-esxi/wiki) for troubleshooting.

### Optional Configuration

If you want to try out the script before putting it into production, you may want to test against the [staging environment](https://letsencrypt.org/docs/staging-environment/) of Let's Encrypt. Probably, you also do not wish to renew certificates once in 30 days but in longer or shorter intervals. Most variables of `renew.sh` can be adjusted by creating a `renew.cfg` file with your overwritten values.

`vi /opt/acme-esxi/renew.cfg`

For a non-default (i.e. private CA) configuration, you can also store your config file in `/etc/acme-esxi` and run `/sbin/auto-backup.sh` to persist your configuration.

`vi /etc/acme-esxi/renew.cfg`
`/sbin/auto-backup.sh`

```bash
# Request a certificate from the staging environment. This can also be set to a private CA.
DIRECTORY_URL="https://acme-staging-v02.api.letsencrypt.org/directory"
# Change your CA bundle if your CA is private, otherwise acme-tiny will complain about untrusted certs.
SSL_CERT_FILE="$LOCALDIR/ca-certificates.crt"
# Make sure to also change your OU, default is Let's Encrypt.
OU="O=Let's Encrypt"
# Set the renewal interval to 15 days
RENEW_DAYS=15
```

To apply your modifications, run `/etc/init.d/acme-esxi start`

## Uninstall

Remove the installed `acme-esxi` package via SSH:

```bash
$ esxcli software vib remove -n acme-esxi
Removal Result
   Message: Operation finished successfully.
   Reboot Required: false
   VIBs Installed:
   VIBs Removed: natethesage_bootbank_acme-esxi_1.0.0-0.0.0
   VIBs Skipped:
```

This action will purge `acme-esxi`, undo any changes to system files (cronjob and port redirection) and finally call `/sbin/generate-certificates` to generate and install a new, self-signed certificate.

## Usage

Usually, fully-automated. No interaction required.

### Hostname Change

If you change the hostname on our ESXi instance, the domain the certificate is issued for will mismatch. In that case, either re-install `acme-esxi` or simply run `/etc/init.d/acme-esxi start`, e.g.:

```bash
$ esxcfg-advcfg -s new-example.com /Misc/hostname
Value of HostName is new-example.com

$ /etc/init.d/acme-esxi start
Running 'start' action
Starting certificate renewal.
Existing cert issued for example.com but current domain name is new-example.com. Requesting a new one!
Generating RSA private key, 4096 bit long modulus
...
```

### Force Renewal

You already have a valid certificate from Let's Encrypt but nonetheless want to renew it now:
```bash
rm /etc/vmware/ssl/rui.crt
/etc/init.d/acme-esxi start
```

## How does it work?

* Checks if the current certificate is issued by Let's Encrypt and due for renewal (_default:_ 30d in advance)
* Generates a 4096-bit RSA keypair and CSR
* Instructs `rhttpproxy` to route all requests to `/.well-known/acme-challenge` to a custom port
* Configures ESXi firewall to allow outgoing HTTP connections
* Uses [acme-tiny](https://github.com/diafygi/acme-tiny) for all interactions with Let's Encrypt
* Starts an HTTP server on a non-privileged port to fulfill Let's Encrypt challenges
* Installs the retrieved certificate and restarts all services relying on it
* Adds a cronjob to check periodically if the certificate is due for renewal (_default:_ weekly on Sunday, 00:00)

## Demo

Here is a sample output when invoking the script manually via SSH:

```bash
$ /etc/init.d/acme-esxi start

Running 'start' action
Starting certificate renewal.
Existing cert for example.com not issued by Let's Encrypt. Requesting a new one!
Generating RSA private key, 4096 bit long modulus
***************************************************************************++++
e is 65537 (0x10001)
Serving HTTP on 0.0.0.0 port 8120 ...
Parsing account key...
Parsing CSR...
Found domains: example.com
Getting directory...
Directory found!
Registering account...
Already registered!
Creating new order...
Order created!
Verifying example.com...
127.0.0.1 - - [29/May/2022 13:14:14] "GET /.well-known/acme-challenge/Ps8VO0v9YzohqfHgnW1xQkHuOKnY0nDakmV9QnrVnVE HTTP/1.1" 200 -
127.0.0.1 - - [29/May/2022 13:14:16] "GET /.well-known/acme-challenge/Ps8VO0v9YzohqfHgnW1xQkHuOKnY0nDakmV9QnrVnVE HTTP/1.1" 200 -
127.0.0.1 - - [29/May/2022 13:14:17] "GET /.well-known/acme-challenge/Ps8VO0v9YzohqfHgnW1xQkHuOKnY0nDakmV9QnrVnVE HTTP/1.1" 200 -
127.0.0.1 - - [29/May/2022 13:14:17] "GET /.well-known/acme-challenge/Ps8VO0v9YzohqfHgnW1xQkHuOKnY0nDakmV9QnrVnVE HTTP/1.1" 200 -
127.0.0.1 - - [29/May/2022 13:14:21] "GET /.well-known/acme-challenge/Ps8VO0v9YzohqfHgnW1xQkHuOKnY0nDakmV9QnrVnVE HTTP/1.1" 200 -
example.com verified!
Signing certificate...
Certificate signed!
Success: Obtained and installed a certificate from Let's Encrypt.
hostd signalled.
rabbitmqproxy is not running
VMware HTTP reverse proxy signalled.
sfcbd-init: Getting Exclusive access, please wait...
sfcbd-init: Exclusive access granted.
vpxa signalled.
vsanperfsvc is not running.
/etc/init.d/vvold ssl_reset, PID 2129283
vvold is not running.
```

## Troubleshooting

See the [Wiki](https://github.com/NateTheSage/acme-esxi/wiki) for possible pitfalls and solutions.

## License

    acme-esxi is free software;
    you can redistribute it and/or modify it under the terms of the
    GNU General Public License as published by the Free Software Foundation,
    either version 3 of the License, or (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
