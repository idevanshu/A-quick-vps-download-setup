<h2>aria2 + nginx Download Server on Ubuntu (DigitalOcean)</h2>

<p><b>A minimal setup</b> that installs <b>aria2</b> for fast multi-connection downloads and <b>nginx</b> with <b>autoindex</b> enabled, so every file you download is instantly browsable and downloadable from your droplet's IP address.</p>

<h2>Overview</h2>

<p>This guide turns a fresh Ubuntu droplet into a simple personal download box. You download files to the server using <b>aria2c</b> with <b>16 parallel connections</b>, and nginx serves the download folder as a clean directory listing over HTTP.</p>

<p><b>What you get:</b></p>

<p>
<b>1.</b> aria2 installed and ready to use from the command line<br>
<b>2.</b> nginx serving <code>/var/www/html</code> with a directory listing<br>
<b>3.</b> Human-readable file sizes and local timestamps in the index
</p>

<h2>Requirements</h2>

<p>
<b>OS:</b> Ubuntu 20.04, 22.04 or 24.04<br>
<b>Server:</b> A DigitalOcean droplet (any size, but make sure you have enough disk space)<br>
<b>Access:</b> SSH access with a user that has <code>sudo</code> privileges<br>
<b>Web root:</b> nginx default, <code>/var/www/html</code>
</p>

<h2>Quick Install (One Command)</h2>

<p>Run this on your droplet. It installs aria2 and nginx, keeps nginx's <b>default web root</b> (<code>/var/www/html</code>), removes the default welcome page so the file list shows, gives your user write access, and enables autoindex:</p>

```bash
sudo apt update && sudo apt install -y aria2 nginx && sudo rm -f /var/www/html/index.nginx-debian.html && sudo chown -R $USER:www-data /var/www/html && sudo chmod 755 /var/www/html && sudo tee /etc/nginx/sites-available/default >/dev/null <<'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/html;

    location / {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
    }
}
EOF
sudo nginx -t && sudo systemctl enable --now nginx && sudo systemctl reload nginx
```

<h2>Firewall</h2>

<p>If <b>UFW</b> is enabled on your droplet, allow SSH and HTTP traffic:</p>

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw enable
```

<p><b>Note:</b> If you also use a DigitalOcean Cloud Firewall, make sure inbound port <b>80</b> is allowed there too.</p>

<h2>Downloading Files</h2>

<p>Use aria2c with <b>16 connections</b> and save directly into the nginx default web root (<code>/var/www/html</code>):</p>

```bash
aria2c -x 16 -s 16 -k 1M -c -d /var/www/html "URL"
```

<p><b>Option reference:</b></p>

<p>
<b>-x 16</b> : maximum connections per server<br>
<b>-s 16</b> : split the download into 16 pieces<br>
<b>-k 1M</b> : minimum split size of 1 MB<br>
<b>-c</b> : resume an interrupted download<br>
<b>-d /var/www/html</b> : output directory
</p>

<h2>Useful aria2 Examples</h2>

<p><b>Save with a custom file name:</b></p>

```bash
aria2c -x 16 -s 16 -k 1M -c -d /var/www/html -o myfile.zip "URL"
```

<p><b>Download many links from a text file</b> (one URL per line):</p>

```bash
aria2c -x 16 -s 16 -k 1M -c -d /var/www/html -i links.txt
```

<p><b>Download a torrent or magnet link:</b></p>

```bash
aria2c -d /var/www/html --seed-time=0 "magnet:?xt=urn:btih:..."
```

<p><b>Run in the background</b> so the download continues after you close SSH:</p>

```bash
nohup aria2c -x 16 -s 16 -k 1M -c -d /var/www/html "URL" > /tmp/aria2.log 2>&1 &
```

<h2>Accessing Your Files</h2>

<p>Open your browser and visit:</p>

```
http://YOUR_DROPLET_IP/
```

<p>You will see a directory listing of everything inside <code>/var/www/html</code>. Files can also be fetched from another machine with:</p>

```bash
wget http://YOUR_DROPLET_IP/filename.zip
```

<h2>Security Warning</h2>

<p><b>This index is public.</b> Anyone who knows your droplet's IP address can view and download every file in the folder. Do not store private or sensitive data here without adding protection.</p>

<p><b>Add basic password protection:</b></p>

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd yourusername
```

<p>Then add these two lines inside the <code>location /</code> block of <code>/etc/nginx/sites-available/default</code>:</p>

```nginx
auth_basic "Restricted";
auth_basic_user_file /etc/nginx/.htpasswd;
```

<p>Apply the change:</p>

```bash
sudo nginx -t && sudo systemctl reload nginx
```

<p><b>Tip:</b> Basic auth sends credentials unencrypted over plain HTTP. For real protection, attach a domain and enable HTTPS with Let's Encrypt (<code>sudo apt install -y certbot python3-certbot-nginx</code>).</p>

<h2>Managing Files</h2>

<p><b>List downloaded files and sizes:</b></p>

```bash
ls -lh /var/www/html
```

<p><b>Delete a file:</b></p>

```bash
rm /var/www/html/filename.zip
```

<p><b>Check remaining disk space:</b></p>

```bash
df -h
```

<h2>Troubleshooting</h2>

<p><b>Default nginx page shows instead of the file list:</b> The welcome page is taking priority over autoindex. Remove it:</p>

```bash
sudo rm -f /var/www/html/index.nginx-debian.html /var/www/html/index.html
```

<p><b>Permission denied when downloading:</b> Your user cannot write to the web root. Fix ownership:</p>

```bash
sudo chown -R $USER:www-data /var/www/html
```

<p><b>403 Forbidden:</b> nginx cannot read the folder. Fix permissions:</p>

```bash
sudo chmod 755 /var/www/html
sudo chmod -R a+rX /var/www/html
```

<p><b>Page does not load:</b> Check that nginx is running and that port 80 is open in both UFW and the DigitalOcean firewall:</p>

```bash
sudo systemctl status nginx
sudo ufw status
```

<p><b>Config errors:</b> Test the nginx configuration and read the log:</p>

```bash
sudo nginx -t
sudo tail -f /var/log/nginx/error.log
```

<p><b>Downloads are slow or rejected:</b> Some servers limit connections per IP. Lower the value, for example <code>-x 8 -s 8</code>.</p>

<p><b>Disk full:</b> Large downloads can fill a small droplet. Attach a DigitalOcean Volume and use its mount path instead of <code>/var/www/html</code>.</p>

<h2>Uninstall</h2>

<p>To remove everything installed by this guide:</p>

```bash
sudo systemctl disable --now nginx
sudo apt remove --purge -y aria2 nginx
sudo apt autoremove -y
```

<p><b>Your downloaded files</b> in <code>/var/www/html</code> are not removed automatically. Delete them manually if you no longer need them.</p>

<h2>License</h2>

<p>This project is released under the <b>MIT License</b>. Feel free to use, modify and share it.</p>
