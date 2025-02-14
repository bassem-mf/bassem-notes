# Get and Auto-Renew TLS Certificates

This article explains how to configure a Linux server to get and auto-renew domain-validated TLS certificates. The following software and services are used:

* [Let's Encrypt](https://letsencrypt.org/): Certificate authority and ACME server
* [Certbot](https://certbot.eff.org/): ACME client
* [Cloudflare](https://www.cloudflare.com/): DNS hosting service

Here are the steps:

1. [Install Certbot](https://certbot.eff.org/instructions) and the [certbot-dns-cloudflare](https://certbot-dns-cloudflare.readthedocs.io/) plugin

2. [Cloudflare](https://www.cloudflare.com/) offers free DNS hosting even if you did not purchase the domain from them. You will need to sign up for a Cloudflare account if you do not already have one. Then [add your domain to Cloudflare](https://developers.cloudflare.com/fundamentals/setup/manage-domains/add-site/). Add all the DNS records that you need for this domain. The DNS records do not need to be "Proxied" unless you want to use other Cloudflare services. After you complete the Cloudflare DNS setup, update your domain registrar to use the Cloudflare nameservers.

3. [Create a Cloudflare API token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) for the domain you just added. On the "Create Token" form, under "Permissions", select "Zone, DNS, Edit". And under "Zone Resources" select "Include, Specific zone, your-domain.com". And under "Client IP Address Filtering" select "Is in, your server IP address(es)".

4. The [certbot-dns-cloudflare plugin](https://certbot-dns-cloudflare.readthedocs.io/) reads the Cloudflare API token from a ".ini" configuration file. So we need to create this file on the server.

    ```bash
    sudo mkdir -p /etc/.secrets/certbot
    read -sp "Enter the Cloudflare API token: " token && echo
    echo "dns_cloudflare_api_token = $token" \
        | sudo tee /etc/.secrets/certbot/cloudflare-example.com.ini \
        > /dev/null
    sudo chmod 600 /etc/.secrets/certbot/cloudflare-example.com.ini
    ```

    This API token can be used to edit the DNS records. So it is a sensitive piece of information that should be protected. We did not place the token in the `echo` command directly to avoid having it saved in the shell history file. Instead, we read the token from the standard input, which is safer. We also changed the ".ini" file permissions to make the file accessible only by the owner (which is "root").

5. Run Certbot with the certbot-dns-cloudflare plugin to get the certificate and setup auto-renewal.

    ```bash
    sudo certbot certonly \
        --dns-cloudflare \
        --dns-cloudflare-credentials /etc/.secrets/certbot/cloudflare-example.com.ini \
        -d example.com -d *.example.com
    ```

    If your server software needs to be notified or restarted every time the certificate is renewed, then you will need to add the `--deploy-hook` option and provide the path to the script that should be run after every successful renewal. In my case, the server software that I am using polls the certificate files and automatically reloads them when they change. So I do not need a deploy hook.

6. Next, we need to make the certificate files accessible to the server software.

    There are two important directories we need to learn about:

    * `/etc/letsencrypt/archive/` This directory contains all the certificates that were ever obtained. When a certificate is renewed, the new certificate files are added to this directory and the old certificate files remain there.
    * `/etc/letsencrypt/live/` This directory contains symbolic links to the latest certificate files in the "archive" directory. When a certificate is renewed, the symbolic links are updated to point to the latest certificate files. The server software should be given the paths of the files in the "live" directory.

    For historical reasons, Certbot creates these two directories with permissions of "700". Meaning that certificates are accessible only to servers that run as the root user. This is too restrictive. The files in these two directories, with the exception of the private key files, are public information. So let's change the directory permissions to allow any user to read them.

    ```bash
    sudo chmod 755 /etc/letsencrypt/{live,archive}
    ```

    The private key files are still accessible only to the root user. But the server software will need to read them. So let's create a group for the users who will be allowed to read the private key files.

    ```bash
    sudo addgroup tls-server
    ```

    And change the group of the private key file to the "tls-server" group we just created.

    ```bash
    sudo chgrp tls-server /etc/letsencrypt/live/<domain>/privkey.pem
    ```

    And change the group permission on the private key file to allow reading.

    ```bash
    sudo chmod 640 /etc/letsencrypt/live/<domain>/privkey.pem
    ```

    And add the user running the server software to the "tls-server" group.

    ```bash
    sudo adduser <user> tls-server
    ```

7. Configure the server software to use the certificate files in "/etc/letsencrypt/live/\<domain\>/". Most server software will use "fullchain.pem" and "privkey.pem". The other files are less commonly used.

## References

* [Certbot User Guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
* [certbot-dns-cloudflare plugin documentation](https://certbot-dns-cloudflare.readthedocs.io/en/stable/)
