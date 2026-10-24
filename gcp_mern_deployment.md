# Deploy a MERN Application on Google Cloud Compute Engine

A step-by-step guide to creating an Ubuntu VM on Google Cloud, configuring SSH and firewalls, installing Nginx and Node.js, deploying a React frontend and Node.js API, connecting a domain, and enabling HTTPS.

> **Example values used in this guide**
>
> - VM name: `mern-server`
> - Zone: `us-central1-f`
> - Network: `default`
> - Website directory: `/var/www/dbinfo`
> - Node.js API port: `8800`
> - Example domain: `dbinfo.com`
>
> Replace these values with your own. Do not paste example IP addresses, domain names, repository URLs, or environment variables into production unchanged.

---

## 1. Create a Compute Engine VM

1. Open the [Google Cloud Console](https://console.cloud.google.com/).
2. Select or create a Google Cloud project and make sure billing is enabled.
3. Open **Compute Engine → VM instances**.
4. Click **Create instance**.
5. Configure the VM:
   - **Name:** `mern-server`
   - **Region/zone:** choose a region close to your users; this guide uses `us-central1-f`
   - **Machine type:** choose one suitable for your app (for example, `e2-medium`)
   - **Boot disk:** Ubuntu LTS (choose a currently supported LTS version)
   - **Firewall:** check **Allow HTTP traffic** and **Allow HTTPS traffic** if you intend to serve a public website on ports 80 and 443.
6. Create the VM.
7. On **Compute Engine → VM instances**, note the VM's **External IP** and zone.

A VM needs an external IP for direct public access. For a production service, consider reserving a static external IP so the address does not change after a VM stop/start.

### Connect to the VM

The simplest option is to click **SSH** beside the VM in the Google Cloud Console.

Alternatively, open **Cloud Shell** and run:

```bash
gcloud compute instances list
gcloud compute ssh mern-server --zone=us-central1-f
```

Google Cloud OS Login or project/instance SSH keys may control access. You generally do not need to create a root password or allow root SSH login. Use your normal VM user with `sudo`.

---

## 2. Understand the two firewall layers

Google Cloud and Ubuntu have separate firewall controls:

1. **GCP VPC firewall rules** control traffic reaching the VM's network interface.
2. **Ubuntu UFW** controls traffic on the VM operating system.

For a public web server, allow inbound TCP ports **80 (HTTP)** and **443 (HTTPS)** in both layers if UFW is enabled. Keep **22 (SSH)** restricted to trusted source IPs where possible. Do not open port 21 unless you deliberately operate an FTP service. You usually do not need to expose the Node.js API port publicly when Nginx proxies to it over `127.0.0.1`.

### 2.1 Create GCP firewall rules

You can check existing rules in the Console under **VPC network → Firewall** or in Cloud Shell:

```bash
gcloud compute firewall-rules list \
  --filter="network:default" \
  --format="table(name,network,direction,allowed,sourceRanges,targetTags)"
```

If the VM uses the `default` network, create an HTTP rule:

```bash
gcloud compute firewall-rules create allow-http \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=0.0.0.0/0
```

Create an HTTPS rule for TLS websites:

```bash
gcloud compute firewall-rules create allow-https \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0
```

These example rules have no target tag, so they apply to all VM instances on the `default` network. To limit the rules to this VM, use a network tag instead:

```bash
gcloud compute instances add-tags mern-server \
  --zone=us-central1-f \
  --tags=mern-server
```

Then add `--target-tags=mern-server` to the relevant firewall rule creation commands. If you created a broad rule already, consider replacing it with a tag-targeted rule after verifying the replacement works.

**If your VM is on a different VPC network, replace `default` with the actual network name.** A GCP firewall rule cannot make a service reachable if the VM has no public IP, the service is not listening, or another firewall/policy blocks the traffic.

### 2.2 SSH firewall access

The default network often has an SSH rule, but verify it exists:

```bash
gcloud compute firewall-rules list \
  --filter="name~allow-ssh" \
  --format="table(name,allowed,sourceRanges,targetTags)"
```

If you need to create one, restrict the source range to your trusted public IP rather than opening SSH to everyone:

```bash
gcloud compute firewall-rules create allow-ssh-from-my-ip \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=YOUR_PUBLIC_IP/32 \
  --target-tags=mern-server
```

Replace `YOUR_PUBLIC_IP` with your own public IPv4 address, without adding `https://`. Google Cloud browser SSH may use Google-managed connection methods; if you use IAP TCP forwarding, configure the corresponding IAP firewall access instead.

### 2.3 Configure Ubuntu UFW

Run these commands **inside the VM over SSH**, not in Cloud Shell:

```bash
sudo apt update
sudo apt install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
```

Only after confirming SSH is allowed, enable UFW:

```bash
sudo ufw enable
sudo ufw status
```

Keep your current SSH session open while testing a second SSH connection, so you can recover if access is accidentally blocked.

> Do not run `ufw enable` before allowing SSH if you rely on SSH to access the VM. Do not open port 3000 or 8800 publicly when Nginx can reach the app locally.

---

## 3. Update Ubuntu and install Nginx

Run inside the VM:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx git curl
```

If Apache is installed and you intend to use Nginx on port 80, check first:

```bash
sudo systemctl status apache2 --no-pager
```

If Apache is not needed, stop and disable it:

```bash
sudo systemctl disable --now apache2
sudo apt remove -y apache2
sudo apt autoremove -y
```

If Apache is not installed, the commands may report that the service/package is absent; that is fine.

Enable Nginx at boot and start it:

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

Test Nginx locally:

```bash
curl -I http://localhost
```

Once the GCP HTTP rule and any UFW rules are in place, visit:

```text
http://YOUR_EXTERNAL_IP/
```

If it times out, see [Troubleshooting](#13-troubleshooting).

---

## 4. Create the website directory and a test page

Create the directory:

```bash
sudo mkdir -p /var/www/dbinfo
sudo chown -R "$USER":www-data /var/www/dbinfo
sudo chmod -R 755 /var/www/dbinfo
```

Create a test HTML page:

```bash
cat <<'EOF' | sudo tee /var/www/dbinfo/index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>DBInfo</title>
</head>
<body>
  <h1>Nginx is working on Google Cloud!</h1>
</body>
</html>
EOF
```

### Configure Nginx for the test page

Create a site configuration:

```bash
sudo nano /etc/nginx/sites-available/dbinfo
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name _;

    root /var/www/dbinfo;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Enable the site and test the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/dbinfo /etc/nginx/sites-enabled/dbinfo
```

If the default site conflicts with this catch-all configuration, disable it:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Then:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Check locally and from your browser:

```bash
curl -I http://localhost
```

Open `http://YOUR_EXTERNAL_IP/`. You should see the test page.

---

## 5. Install Node.js and npm

Use a currently supported Node.js LTS version that is compatible with your application. One option is to install Node.js from the official Node.js distribution or a trusted package source approved for your environment.

Check what is installed:

```bash
node --version
npm --version
```

On Ubuntu, the default `apt install nodejs npm` packages may not provide the Node.js version your project requires. Check your project's `engines` field and use a supported installation method/version.

---

## 6. Clone your repository

Choose a deployment directory. For example:

```bash
sudo mkdir -p /opt/dbinfo
sudo chown "$USER":"$USER" /opt/dbinfo
cd /opt/dbinfo
git clone YOUR_REPOSITORY_URL .
```

Replace `YOUR_REPOSITORY_URL` with your repository URL. For a private repository, configure authentication securely; do not put access tokens directly into shell history or commit them to Git.

A common project structure is:

```text
/opt/dbinfo/
├── api/
└── client/
```

Your actual structure may differ. Adjust the following paths to match your repository.

---

## 7. Configure and run the Node.js API

Go to the backend directory:

```bash
cd /opt/dbinfo/api
npm install
```

Create or edit the environment file:

```bash
nano .env
```

Add your real environment variables. Do not commit `.env` to Git or publish its secrets.

Make the backend listen on the loopback interface when Nginx is on the same VM. For example, an Express app can use:

```javascript
app.listen(8800, "127.0.0.1", () => {
  console.log("API listening on port 8800");
});
```

Use the port and startup file your application actually requires. For production, use the project's documented start script if one exists.

Test the API using its real health endpoint or a known route:

```bash
curl -i http://127.0.0.1:8800/
```

A `404` response can still show that the server is reachable, if `/` is not a defined route. A connection-refused error usually means the process is not listening on that address and port.

### Keep the API running with PM2

Install PM2:

```bash
sudo npm install -g pm2
```

Start the API using your actual entry file. For example:

```bash
cd /opt/dbinfo/api
pm2 start index.js --name api
pm2 status
pm2 logs api
```

Configure startup on reboot:

```bash
pm2 startup
```

PM2 prints a command to run with `sudo`. Copy and execute the exact command it prints, then save the process list:

```bash
pm2 save
```

If your project uses `npm start` rather than `index.js`, start the npm script instead:

```bash
pm2 start npm --name api -- start
```

Do not start both commands for the same API.

---

## 8. Configure Nginx to proxy API requests

Edit the Nginx site:

```bash
sudo nano /etc/nginx/sites-available/dbinfo
```

For a React single-page application and an API on the same domain, use a configuration like this:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name _;

    root /var/www/dbinfo/client;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8800;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

This example expects the API to handle paths beginning with `/api/`. With the `proxy_pass` shown above (no trailing slash after `8800`), the `/api/...` path is passed through unchanged. If your backend expects a different path, adjust the Nginx location and `proxy_pass` accordingly.

If your API needs WebSockets, add the appropriate `Upgrade` and `Connection` headers to the relevant location.

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Do not expose port 8800 publicly if Nginx is proxying to `127.0.0.1:8800`; the backend can remain local.

---

## 9. Build and deploy the React frontend

Go to the frontend directory:

```bash
cd /opt/dbinfo/client
```

Install dependencies:

```bash
npm install
```

Set the production API URL according to your frontend framework and its environment variable conventions. When frontend code is built, environment variables are often embedded in the static assets. Never put server secrets in frontend environment variables.

For a same-domain deployment, configure the frontend to call relative API paths such as `/api/...` where appropriate.

Build the app:

```bash
npm run build
```

Build output differs by framework:
- Create React App commonly outputs `build/`
- Vite commonly outputs `dist/`

Check which directory was created:

```bash
ls -la
```

For Create React App:

```bash
sudo mkdir -p /var/www/dbinfo/client
sudo cp -a build/. /var/www/dbinfo/client/
```

For Vite:

```bash
sudo mkdir -p /var/www/dbinfo/client
sudo cp -a dist/. /var/www/dbinfo/client/
```

Set readable permissions:

```bash
sudo find /var/www/dbinfo -type d -exec chmod 755 {} \;
sudo find /var/www/dbinfo -type f -exec chmod 644 {} \;
```

Validate and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Visit `http://YOUR_EXTERNAL_IP/`.

> **Important:** Do not run `rm -rf /var/www/dbinfo/*` unless you are certain you no longer need the existing files. It can delete files that are serving your site.

---

## 10. Add a domain name

1. Register or use a domain you control.
2. In the domain's DNS provider, create an `A` record for the root domain pointing to the VM's static external IPv4 address.
3. Optionally create another `A` record for `www` pointing to the same IP, or a `CNAME` for `www` pointing to your root domain, depending on your DNS provider.
4. Wait for DNS changes to propagate.
5. Confirm the records resolve to the intended IP before requesting a certificate.

For a production website, reserve a static external IP in Google Cloud and attach it to the VM so that the address remains stable.

### Nginx configuration for a domain

Edit:

```bash
sudo nano /etc/nginx/sites-available/dbinfo
```

Use your real domain names:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name dbinfo.com www.dbinfo.com;

    root /var/www/dbinfo/client;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8800;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

If you also have an admin frontend, add a separate `server` block with its own `server_name` and document root. If you use `api.dbinfo.com` as a separate API hostname, configure a separate `server` block that proxies to the backend.

Test and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 11. Enable HTTPS with Let's Encrypt

Make sure:
- Your domain's DNS records resolve to this VM's public IP.
- GCP allows inbound TCP 80 and 443.
- UFW allows inbound TCP 80 and 443 if UFW is enabled.
- Nginx is running and the domain's HTTP server block is configured.

Install Certbot:

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Request and install the certificate:

```bash
sudo certbot --nginx -d dbinfo.com -d www.dbinfo.com
```

Replace the domain names with your own. Follow the prompts and choose HTTPS redirection if offered and appropriate.

Check renewal:

```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

Let's Encrypt certificates typically last 90 days; Certbot's renewal mechanism should renew them automatically when configured correctly.

---

## 12. Useful operational commands

### Nginx

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo systemctl reload nginx
sudo journalctl -u nginx -n 100 --no-pager
sudo tail -n 100 /var/log/nginx/error.log
```

### Node.js / PM2

```bash
pm2 status
pm2 logs api
pm2 restart api
pm2 save
```

### Check listening ports

```bash
sudo ss -lntp
```

### Test services locally

```bash
curl -I http://localhost
curl -i http://127.0.0.1:8800/
```

Use a real API route if `/` is not defined by your app.

### Check GCP VM and firewall configuration from Cloud Shell

```bash
gcloud compute instances list
gcloud compute firewall-rules list \
  --format="table(name,network,direction,allowed,sourceRanges,targetTags)"
```

---

## 13. Troubleshooting

### Browser shows `ERR_CONNECTION_TIMED_OUT`

1. Verify the VM is running and has an external IP.
2. Confirm a GCP VPC ingress rule allows TCP 80 or 443 on the VM's network.
3. Confirm the rule's network and target tags/service account match the VM.
4. If UFW is enabled, verify `sudo ufw status verbose` allows the required port.
5. Check `sudo systemctl status nginx --no-pager`.
6. Check `sudo ss -lntp` to see whether Nginx is listening on `0.0.0.0:80`, `[::]:80`, or another suitable address.
7. Test `curl -I http://localhost` from inside the VM.
8. Check whether an organization policy, custom route, or other firewall policy blocks the traffic.

### Browser shows `502 Bad Gateway`

Nginx is reachable but cannot successfully contact the upstream API. Check:

```bash
pm2 status
pm2 logs api
curl -i http://127.0.0.1:8800/
```

Confirm that the upstream port in `proxy_pass` matches the actual Node.js listening port.

### Browser shows `403 Forbidden` or `404 Not Found`

Check that the configured `root` points to the correct directory and that `index.html` exists:

```bash
ls -la /var/www/dbinfo/client
```

Ensure the files are readable by Nginx and that the correct site configuration is enabled.

### Nginx reports a configuration error

```bash
sudo nginx -t
```

Fix the file and line reported by the command before reloading Nginx.

### Firewall rule already exists

If GCP reports that a rule name already exists, list the rule and inspect its network, allowed ports, source ranges, and target tags rather than creating duplicate rules.

### Port 80 is open in GCP but the site still times out

Opening the GCP firewall does not start Nginx. Check the OS firewall and service status, and confirm the app is bound to a reachable interface. Also ensure you are using the correct external IP.

---

## 14. Recommended production security checklist

- [ ] Reserve a static external IP for the website.
- [ ] Keep SSH access restricted to trusted IPs or use a suitable Google Cloud access method such as IAP.
- [ ] Expose only the ports you need: normally 80 and 443 for the website and restricted SSH access.
- [ ] Do not expose MongoDB or the Node.js API port publicly unless there is a specific, secured requirement.
- [ ] Keep `.env` files and secrets out of Git and frontend builds.
- [ ] Run the API as a non-root user where practical.
- [ ] Use HTTPS and verify automatic certificate renewal.
- [ ] Keep Ubuntu, Node.js, npm dependencies, and your app updated.
- [ ] Back up important application data and database contents.
- [ ] Review GCP firewall rules periodically and remove rules that are no longer needed.

---

## Quick checklist for a working website

1. The VM is running and has the correct external IP.
2. A GCP ingress firewall rule allows TCP 80 (and 443 for HTTPS).
3. Ubuntu UFW allows TCP 80 (and 443 if enabled).
4. Nginx is running and passes `sudo nginx -t`.
5. `curl -I http://localhost` succeeds inside the VM.
6. The Nginx document root contains the deployed `index.html`.
7. The Node.js API is running under PM2 and Nginx proxies to the correct local port.
8. Domain DNS records point to the VM's external IP before enabling HTTPS.
