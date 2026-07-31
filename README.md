# Deployment and Diagnostics Guide
https://github.com/jamilahmed2/Deployment-guide/blob/main/Diagnostics.md
## Prerequisites
Before starting the deployment, ensure you have the necessary dependencies installed:

### 1. Install Nginx
```sh
sudo apt update
sudo apt install nginx -y
```

### 2. Install Node.js using NVM
Install NVM (Node Version Manager):
```sh
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
source ~/.bashrc
```
Verify installation:
```sh
nvm --version
```
Install Node.js (latest LTS version):
```sh
nvm install --lts
nvm use --lts
```
Verify Node.js and npm installation:
```sh
node -v
npm -v
```
### 2.1 Install Bun (Optional)

If your project uses **Bun** instead of **npm**, install it using the following commands.

#### Install Bun
```unzip
apt install unzip
```
```sh
curl -fsSL https://bun.sh/install | bash
```
#### Reload Your Shell
For Bash:
```sh
source ~/.bashrc
```

For Zsh:
```sh
source ~/.zshrc
```

#### Verify Installation
```sh
bun --version
```

#### Install Project Dependencies
```sh
bun install
```

#### Run Scripts
Development:
```sh
bun run dev
```

Build:
```sh
bun run build
```

Start:
```sh
bun run start
```

#### Install a Global Package
```sh
bun add -g <package-name>
```

#### Update Bun
```sh
bun upgrade
```

### 3. Install MySQL Server
```sh
sudo apt install mysql-server -y
```

Change the Database password

```sh
mysql -u root -p
```

```sh
press enter (if password not configured yet)
or type the password
```

update the pass if not

```sh
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '[YOUR PASSWORD]';
EXIT;
```


### 3.1 Install Postgres Server

```sh
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### 4 Check Postgres status
```sh
sudo systemctl status postgresql
```

### 5. Switch to postgres user
```sh
sudo -i -u postgres
```

### 6. Access PostgreSQL shell
```sh
psql
```

### 7. Change the password
```sh
\password postgres
```

### Exit psql
```sh
\q
```
or press Ctrl + D

## Step 0: Configure Swap Memory (Recommended for 2GB RAM Servers)

Small servers (1-2GB RAM) frequently run out of memory during `npm install`, `npm run build`, or `bun run build`, causing the process to be killed (`Killed` in the terminal, or exit code 137). Adding a swap file gives the kernel extra "overflow" space on disk so builds don't get OOM-killed.

### Check current memory and swap
```sh
free -h
swapon --show
```

### Create a 2GB swap file
```sh
sudo fallocate -l 2G /swapfile
```
If `fallocate` isn't supported on your filesystem, use `dd` instead:
```sh
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
```

### Secure and enable the swap file
```sh
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Verify it's active
```sh
sudo swapon --show
free -h
```

### Make swap permanent (survives reboot)
```sh
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Tune swappiness (optional, reduces unnecessary swapping)
Lower values make the kernel prefer RAM over swap until it's actually needed — good for a build/app server:
```sh
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

### Remove swap (if ever needed)
```sh
sudo swapoff /swapfile
sudo rm /swapfile
sudo sed -i '/\/swapfile/d' /etc/fstab
```

> **Tip:** For a 2GB RAM VPS, a 2GB swap file is a safe baseline. For 1GB RAM or heavier builds (large Remix/Next.js apps), consider 4GB swap.

## Step 0.1: Low-Memory Build/Run Commands (512MB RAM, No Swap)

If the server only has **512MB RAM and no swap configured yet**, standard `npm run build` / `bun run build` will often crash with an out-of-memory error. Cap the memory Node/Bun is allowed to use and disable memory-hungry features until you can set up swap (Step 0 above).

### npm / Node — low-memory build
Limit the V8 heap size (in MB) so the process fails gracefully or throttles instead of getting OOM-killed:
```sh
NODE_OPTIONS="--max-old-space-size=384" npm run build
```

Low-memory install (avoids parallel network/install overhead):
```sh
npm install --prefer-offline --no-audit --no-fund --maxsockets 1
```

Low-memory start (production):
```sh
NODE_OPTIONS="--max-old-space-size=384" npm run start
```

If using with Shopify/Remix env vars, combine both:
```sh
SHOPIFY_API_KEY=[API_KEY] NODE_OPTIONS="--max-old-space-size=384" npm run build
```

Low-memory PM2 start (so PM2-managed processes also respect the cap):
```sh
pm2 start npm --name "[handle]" --node-args="--max-old-space-size=384" -- run start
```

### Bun — low-memory build
Bun doesn't expose a heap flag like Node, so constrain it at the OS level with `ulimit` and let the smaller install/runtime footprint do the rest:
```sh
ulimit -v 524288   # limits virtual memory to ~512MB for this shell session
bun install
bun run build
```

Low-memory Bun start:
```sh
ulimit -v 524288
bun run start
```

Low-memory PM2 start for Bun:
```sh
pm2 start bun --name "[handle]" --node-args="" -- start
```

> **Warning:** Building large Remix/Next.js/Shopify apps on 512MB RAM without swap is risky even with these caps — the build may still fail intermittently. These commands buy you some headroom, but setting up swap (Step 0) is the more reliable fix. Treat these as a stopgap, not a permanent solution.

## Step 1: Create Nginx Configuration
Navigate to Nginx Sites-Available Directory:
```sh
cd /etc/nginx/sites-available
```

Create a Configuration File:
```sh
sudo nano [your-domain]
```

Add the following configuration:
```nginx
server {
    listen 80;
    server_name [your-domain];

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Save and exit (`CTRL + X`, then `Y`, then `Enter`).

## Step 2: Create Symbolic Link
```sh
sudo ln -s /etc/nginx/sites-available/[your-domain] /etc/nginx/sites-enabled/
```

Test Nginx configuration:
```sh
sudo nginx -t
```

Reload Nginx:
```sh
sudo systemctl restart nginx
```

## Step 3: Apply SSL Certificate
Install Certbot:
```sh
sudo apt-get install certbot python3-certbot-nginx -y
```

Apply SSL Certificate:
```sh
sudo certbot --nginx -d [your-domain]
```

Install SSL Certificate (if needed):
```sh
sudo certbot install --cert-name [your-domain]
```

Auto-renew SSL certificate:
```sh
sudo certbot renew --dry-run
```

## Step 4: Deploy Your Project
Clone Your Project:
```sh
cd /var/www/html/
git clone [your-repo-url]
```

Install Dependencies:
```sh
cd [your-project-directory]
npm install
```

create an .env file

```sh
sudo nano .env
```

fill up these values in .env

```sh
DATABASE_URL=mysql://host:[YOUR PASSWORD]@localhost:3306/database  # (if using mysql)
SHOPIFY_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx     # (get it from in partner dashboard app config mentioned as Client ID)
SHOPIFY_API_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # (get it from in partner dashboard app config mentioned as Client secret)
SHOPIFY_APP_URL=http://app.fontmarket.com/          # ([your-domain])
SCOPES=read_files,write_files,write_products        # ([scopes needed])
```

Sync MySQL with Remix:
```sh
npm run setup
```

## Step 5: Install and Configure PM2
Install PM2:
```sh
npm install pm2 -g
```

Build Your Application npm or bun:
```sh
SHOPIFY_API_KEY=[API_KEY] npm run build
```
```sh
bun run build
```

> **Low on RAM?** See Step 0 (swap setup) for 2GB servers, or Step 0.1 (memory-capped commands) if you're on 512MB with no swap yet.

Start Application with PM2:
```sh
pm2 start npm --name "[handle]" -- run start
```

```sh
pm2 start bun --name "[handle]" -- start
```

## Final Step: Restart Nginx
```sh
sudo systemctl restart nginx
```

## Additional Commands
### Reset MySQL Root Password
```sh
sudo mysql
```
Inside MySQL shell:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '[YOUR PASSWORD]';
FLUSH PRIVILEGES;
EXIT;
```

### Fix `.env` Not Loading Issue
```sh
rm -rf node_modules/
npm install
```

### Check for OOM Kills (Out of Memory)
If a build or process died unexpectedly on low-RAM servers, check the kernel log to confirm it was memory-related:
```sh
dmesg -T | grep -i "killed process"
```

## Summary
1. Install Nginx, Node.js (via NVM), and MySQL.
2. (If low on RAM) Configure swap memory, or use low-memory build/run commands.
3. Create and configure the Nginx reverse proxy.
4. Apply SSL with Certbot.
5. Deploy your Remix project.
6. Use PM2 for process management.
7. Update extension configurations.
8. Restart Nginx.

Your **Remix app** should now be deployed, secured with SSL, and running on **PM2** with **Nginx** as a reverse proxy.

## 🧩 Extras

### 🔹 Import File from Your Server to Local Machine
Use `scp` to securely copy a file from your remote server to your local system:
```sh
scp user@your-server-ip:<path-on-server> <local-path>
```

**Example:**
```sh
scp root@123.456.789.00:/root/.pm2/logs/translation-worker-error.log ~/Downloads/logs/
```

This will copy the `translation-worker-error.log` file from your server to your local `Downloads/logs` directory.

---

### 🔹 Run Prisma Studio or Development Server Remotely via SSH Tunnel
If you're using **Prisma** and want to access Prisma Studio (or run your dev server) locally while the app runs on the server, use SSH port forwarding:

```sh
ssh -L <local-port>:<remote-host>:<remote-port> user@your-server-ip
```

**Example (for Prisma Studio):**
```sh
ssh -L 5555:localhost:5555 root@123.456.789.00
```

Now you can run Prisma Studio on your local browser:
```
npx prisma studio
```
and access it at [http://localhost:5555](http://localhost:5555).

**Example (for Remix Dev Server):**
```sh
ssh -L 3000:localhost:3000 root@123.456.789.00
```

Then, visit [http://localhost:3000](http://localhost:3000) to access your running Remix app directly from your server environment.
