# truenas-iocage-zenphoto
Script to create an iocage jail on TrueNAS for the latest Zenphoto release, including Caddy 2.x, MariaDB, and Let's Encrypt

This script will create an iocage jail on TrueNAS CORE 13.0 with the latest release of Zenphoto, along with its dependencies.  It will obtain a trusted certificate from Let's Encrypt for the system, install it, and configure it to renew automatically.  It will create the database and generate a strong root password and user password for the database system.  It will configure the jail to store the data and database outside the jail, so it will not be lost in the event you need to rebuild the jail.

## Status
This script will work with TrueNAS CORE 13.0.  Due to the EOL status of FreeBSD 12.0, it is unlikely to work reliably with earlier releases of FreeNAS.

## Usage

### Prerequisites (Let's Encrypt)
This script works best when your installation is able to obtain a certificate from [Let's Encrypt](https://letsencrypt.org/).  When you use it this way, Caddy is able to handle all of the TLS-related configuration for you, obtain and renew certificates automatically, etc.  In order for this to happen, you must meet the two requirements below:

* First, you must own or control a real Internet domain name.  This script obtains a TLS encryption certificate from Let's Encrypt, who will only issue for public domain names.  Thus, domains like `zenphoto.local`, `zenphoto.lan`, or `zenphoto.home` won't work.

* Second, one of these two conditions must be met in order for Let's Encrypt to validate your control over the domain name:

  * You must be able and willing to open ports 80 and 443 from the entire Internet to the jail, and leave them open.  If this applies, do it **before** running this script.
  * DNS hosting for the domain name needs to be with a provider that Caddy supports.  At this time, only Cloudflare is supported.

[Cloudflare](https://www.cloudflare.com/) provides DNS hosting at no cost, and it's well-supported by Caddy.  Cloudflare also provides Dynamic DNS service, if your desired Dynamic DNS client supports their API.  If it doesn't, [DNS-O-Matic](https://dnsomatic.com/) is a Dynamic DNS provider that will interface with many DNS hosts including Cloudflare, has a much simpler API that's more widely supported, and is also free of charge.

If you aren't able or willing to obtain a certificate from Let's Encrypt, this script also supports configuring Caddy with a self-signed certificate, or with no certificate (and thus no HTTPS) at all.

### Prerequisites (Other)
You will need to create 
- 1 Dataset named `zenphoto`
- 2 subdatasets underneath the above mentioned dataset called `data` and `db`, which will store the data and database.
If these are not present, a directory `/zenphoto` will be created in `$POOL_PATH` with the obove mentioned subdirectories. You will want to create the datasets, otherwise the directories will just be created. Datasets make it easy to do snapshots etc...

### Installation
Download the repository to a convenient directory on your TrueNAS system by changing to that directory and running `git clone https://github.com/tschettervictor/truenas-iocage-zenphoto`.  Then change into the new `truenas-iocage-zenphoto` directory and create a file called `zenphoto-config` with your favorite text editor.  In its minimal form, it would look like this:
```
JAIL_IP="192.168.1.199"
DEFAULT_GW_IP="192.168.1.1"
POOL_PATH="/mnt/tank"
HOST_NAME="YOUR_FQDN"
NO_CERT=1
```
Many of the options are self-explanatory, and all should be adjusted to suit your needs, but only a few are mandatory.  The mandatory options are:

* JAIL_IP is the IP address for your jail.  You can optionally add the netmask in CIDR notation (e.g., 192.168.1.199/24).  If not specified, the netmask defaults to 24 bits.  Values of less than 8 bits or more than 30 bits are invalid.
* DEFAULT_GW_IP is the address for your default gateway
* POOL_PATH is the path for your data pool.
* HOST_NAME is the fully-qualified domain name you want to assign to your installation.  If you are planning to get a Let's Encrypt certificate (recommended), you must own (or at least control) this domain, because Let's Encrypt will test that control.  If you're using a self-signed cert, or not getting a cert at all, it's only important that this hostname resolve to your jail inside your network.
* DNS_CERT, STANDALONE_CERT, SELFSIGNED_CERT, and NO_CERT determine which method will be used to generate a TLS certificate (or, in the case of NO_CERT, indicate that you don't want to use SSL at all).  DNS_CERT and STANDALONE_CERT indicate use of DNS or HTTP validation for Let's Encrypt, respectively.  One **and only one** of these must be set to 1.
* DNS_PLUGIN: If DNS_CERT is set, DNS_PLUGIN must contain the name of the DNS validation plugin you'll use with Caddy to validate domain control.  At this time, the only valid value is `cloudflare` (but see below).
* DNS_TOKEN: If DNS_CERT is set, this must be set to a properly-scoped Cloudflare API Token.  You will need to create an API token through Cloudflare's dashboard, which must have "Zone / Zone / Read" and "Zone / DNS / Edit" permissions on the zone (i.e., the domain) you're using for your installation.  See [this documentation](https://github.com/libdns/cloudflare) for further details.
* CERT_EMAIL: If you're obtaining a cert from Let's Encrypt (i.e., either DNS_CERT or STANDALONE_CERT is set to 1), this must be set to a valid email address.  You'll only receive mail there if your cert is about to expire (which should never happen), or if there are significant announcements from Let's Encrypt (which is unlikely to result in more than a few emails per year).
 
In addition, there are some other options which have sensible defaults, but can be adjusted if needed.  These are:

* JAIL_NAME: The name of the jail, defaults to "zenphoto"
* DB_PATH. This is the path to your database files. It defaults to POOL_PATH/zenphoto/db
* INTERFACE: The network interface to use for the jail.  Defaults to `vnet0`.
* JAIL_INTERFACES: Defaults to `vnet0:bridge0`, but you can use this option to select a different network bridge if desired.  This is an advanced option; you're on your own here.
* VNET: Whether to use the iocage virtual network stack.  Defaults to `on`.
* CERT_EMAIL is the email address Let's Encrypt will use to notify you of certificate expiration, or for occasional other important matters.  This is optional.  If you **are** using Let's Encrypt, though, it should be set to a valid address for the system admin.

If you're going to open ports 80 and 443 from the outside world to your jail, do so before running the script, and set STANDALONE_CERT to 1.  If not, but you use a DNS provider that's supported by Caddy, set DNS_CERT to 1.  If neither of these is true, use either NO_CERT (if you want to run without SSL at all) or SELFSIGNED_CERT (to generate a self-signed certificate--this is also the setting to use if you want to use a certificate from another source).

Also, HOST_NAME needs to resolve to your jail from **inside** your network.  You'll probably need to configure this on your router, or on whatever other device provides DNS for your LAN.  If you're unable to do so, you can edit the hosts file on your client computers to achieve this result, but consider installing something like [Pi-Hole](https://pi-hole.net/) to give you control over your DNS.

### Execution
Once you've downloaded the script and prepared the configuration file, run this script (`script zenphoto.log ./zenphoto-jail.sh`).  The script will run for several minutes.  When it finishes, your jail will be created, Zenphoto will be installed and configured, and you will be ready to run the setup.

### Obtaining a trusted Let's Encrypt cert
This configuration generated by this script will obtain certs from a non-trusted certificate authority by default.  This is to prevent you from exhausting the [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/) while you're testing things out.  Once you're sure things are working, you'll want to get a trusted cert instead.  To do this, you can use a simple script that's included.  As long as you haven't changed the default jail name, you can do this by running `iocage exec zenphoto /root/remove-staging.sh` (if you have changed the jail name, replace "zenphoto" in that command with the jail name).

This script has only been tested with Cloudflare, which works well.

Visit the [Caddy download page](https://caddyserver.com/download) to see the DNS authentication plugins currently available.  To build Caddy with your desired plugin, use the last part of the "Package" on that page as DNS_PLUGIN in your `zenphoto-config` file.  E.g., if the package name is `github.com/caddy-dns/cloudflare`, you'd set `DNS_PLUGIN=cloudflare`.  From that page, there are also links to the documentation for each plugin, which will describe what credentials are needed.  If your provider needs only an API token (as is the case with Cloudflare, and apparently with DNSPod and Gandi), you'll likely be able to set `DNS_TOKEN=long_api_token` in the `zenphoto-config` file and not need to do anything else.  If your provider requires different credentials, you'll need to modify the Caddyfile to account for them.

### Notes
- Reinstalls work as expected when the previous database is present.
- Reinstalling replaces the 'zp-core', 'themes' and 'index.php' files according to the documentation. If you have modified a theme or installed a custom theme, please be sure to back them up before reinstalling.
