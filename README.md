# Hosting NGINX on Amazon EC2 with Route 53 DNS

I deployed NGINX on an Amazon Linux EC2 instance and connected it to my own domain using Amazon Route 53. The result was the default NGINX page at **`http://nginx-server.asad-ahmed.ca`** during this setup.

## How the request reaches the server

```text
Browser → Route 53 A record → EC2 public IPv4 → security group (TCP 80) → NGINX
```

| Component | Configuration |
| --- | --- |
| Domain | `asad-ahmed.ca` |
| Hostname | `nginx-server.asad-ahmed.ca` |
| DNS | Route 53 public hosted zone and A record |
| Compute | EC2 instance running Amazon Linux 2023 |
| Web server | NGINX default welcome page |
| Access | HTTP on TCP 80; SSH on TCP 22 from my IP |

## 1. Register the domain and create a hosted zone

I registered `asad-ahmed.ca` and used a Route 53 public hosted zone to manage its DNS records. I chose a subdomain, `nginx-server.asad-ahmed.ca`, so the service would have a descriptive address while leaving the root domain available for other uses.

![Successful domain registration notification](screenshots/domain-registration.png)

![Route 53 public hosted zone for asad-ahmed.ca](screenshots/route53-hosted-zone.png)

## 2. Launch EC2 and configure access

I launched an Amazon Linux 2023 EC2 instance. Amazon Linux gave me a straightforward environment for installing NGINX and practicing remote Linux administration. I allowed inbound HTTP on TCP port 80 so visitors could reach the web server.

I initially omitted SSH. Once I needed to connect to install NGINX, I added inbound TCP port 22 with my IP address as the source. Limiting SSH to my IP reduced the number of networks that could attempt to connect. The screenshot shows the final security group rules.

![EC2 instance and its inbound HTTP and SSH rules](screenshots/ec2-security-group.png)

## 3. Connect and install NGINX

I restricted access to the private key on my computer, connected to EC2 as `ec2-user`, and installed NGINX:

```bash
chmod 400 ssh-ec2.pem
ssh -i ssh-ec2.pem ec2-user@<EC2_PUBLIC_IP>
sudo yum install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

The key permission change protects the local key file; the security group separately controls whether SSH traffic can reach the instance. I used NGINX's default page to verify the hosting and networking path before adding any custom site content. The SSH screenshot records the connection and installation command; the browser result below confirms the server was serving the page.

![Setting private key file permissions](screenshots/key-permissions.png)

![SSH connection to Amazon Linux and NGINX installation command](screenshots/ssh-nginx-install.png)

## 4. Point the hostname to EC2

I created a Route 53 **A record** for `nginx-server.asad-ahmed.ca` pointing to the instance's public IPv4 address. An A record was the direct way to connect this hostname to an IPv4 server. At the time of the screenshots, the address was `98.92.227.114`.

![Route 53 A record pointing the subdomain to EC2](screenshots/route53-a-record.png)

## 5. Verify DNS and the web page

I checked what IP address the hostname resolved to:

```bash
nslookup nginx-server.asad-ahmed.ca
```

The result matched the A record, `98.92.227.114`. This confirmed DNS resolution independently of the browser.

![nslookup result for nginx-server.asad-ahmed.ca](screenshots/dns-lookup.png)

I then opened **`http://nginx-server.asad-ahmed.ca`** and saw the **Welcome to nginx!** page. This verified that the browser could resolve the hostname, reach port 80 on the instance, and receive a response from NGINX.

![NGINX welcome page loaded through the custom hostname over HTTP](screenshots/nginx-browser.png)

## Challenges and fixes

### The browser attempted HTTPS

I first tried to reach the site over HTTPS. This setup served HTTP on port 80 and had no TLS configuration on port 443. Depending on browser settings and how an address is entered, a browser may try HTTPS. I explicitly entered `http://nginx-server.asad-ahmed.ca`, which loaded the page. This showed me that a working DNS record does not by itself enable HTTPS.

### SSH could not reach the instance

I ran `chmod 400 ssh-ec2.pem` but still could not connect because the security group had no inbound SSH rule. After I added TCP port 22 from my IP, the SSH connection succeeded. I learned to check both the local key permissions and the inbound network rule when troubleshooting SSH.

## What I learned

- DNS maps the hostname to an IP address; `nslookup` shows whether that mapping resolves as expected.
- The security group decides which inbound connections can reach EC2. NGINX must also be installed and running to answer HTTP requests.
- HTTP (80), HTTPS (443), and SSH (22) serve different purposes. Enabling one does not enable the others.
- Testing DNS and HTTP separately makes it easier to isolate where a connection fails.

## How I would improve this setup

1. **Add HTTPS:** Configure a TLS certificate and port 443, then redirect HTTP to HTTPS so visitors get an encrypted connection without typing the scheme manually.
2. **Keep the IP stable:** Assign an Elastic IP or use an architecture with a stable DNS target. A normal EC2 public IPv4 address can change after a stop/start, leaving the A record pointing at the old address.
3. **Replace the default page:** Deploy a small custom site and configure an NGINX server block for the hostname. The welcome page proves the path works but is not the final content.
4. **Automate and observe:** Define the instance, security group, and DNS record as code, and add basic service monitoring. That would make the setup easier to reproduce and problems easier to detect.

> The public IP and screenshots show the state at the time of this setup. The live domain may change or stop working if the instance is stopped, replaced, or removed.
