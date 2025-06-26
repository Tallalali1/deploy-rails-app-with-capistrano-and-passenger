# Deploy Rails App with Capistrano and Passenger using rbenv

This guide covers deploying a Rails application on a DigitalOcean server using Capistrano, Passenger, and Nginx.

---

## 1. Create Deploy User (Can skip if using aws ec2 instance)

> **Note:** This step is for DigitalOcean servers. If you are using AWS EC2, you can skip this step and use the default `ubuntu` user.

```sh
sudo adduser deploy
sudo adduser deploy sudo
```

---

## 2. Install Dependencies

```sh
first run
sudo apt-get update
then run
sudo apt-get install curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates nodejs yarn
finally run
sudo apt install libpq-dev
```

---

## 3. Install rbenv & Ruby

1. **Install rbenv:**
    ```sh
    curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
    ```

2. **Add rbenv to your shell:**
    ```sh
    echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
    echo 'eval "$(rbenv init -)"' >> ~/.bashrc
    source ~/.bashrc
    ```
    > *Optional:* To manually verify, open your bash profile with:
    > ```sh
    > sudo nano ~/.bashrc
    > ```
    
3. **Install Ruby (replace `3.3.0` with your required version if needed):**
    ```sh
    rbenv install 3.3.0
    rbenv global 3.3.0
    ```
---

## 4. Install Bundler

```sh
gem install bundler
```

---

## 5. Install Passenger & Nginx

### (a) Install the PGP key

```sh
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
```

### (b) Add the APT repository

> **Note:** Only run the command that matches your server's Ubuntu version.

```sh
Step 1 choose command based on your ubuntu version
# For Ubuntu 18.04 (bionic)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list'
# For Ubuntu 22.04 (jammy)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger jammy main > /etc/apt/sources.list.d/passenger.list'
# For Ubuntu 24.04 (noble)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger noble main > /etc/apt/sources.list.d/passenger.list'
Step 2 run the selected command
sudo apt-get update
```

### (c) Install Passenger and Nginx module

```sh
sudo apt-get install -y libnginx-mod-http-passenger
```

### (d) Ensure configuration files exist

```sh
if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then \
  sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi

sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
# Open configurations
sudo nano /etc/nginx/conf.d/mod-http-passenger.conf
```

Set the `passenger_ruby` option:

```
# For AWS EC2 instances, use:
passenger_ruby /home/ubuntu/.rbenv/shims/ruby;

# For DigitalOcean servers, use:
passenger_ruby /home/deploy/.rbenv/shims/ruby;
```

---

## 6. Configure Nginx (deploy app before this step)

```sh
sudo apt install nginx-full
```
> **Before this step:**  
> You should configure Capistrano and deploy your app to the server.  
> If you need help setting up Capistrano, see [this guide](https://github.com/Tallalali1/setup-Capistrano-in-rails-app).

```sh
sudo nano /etc/nginx/sites-enabled/default
```

Example config:

```
if aws ec2 instance
server {
  listen 80;
  listen [::]:80;

  server_name _;
  root /home/ubuntu/yourappname/current/public;
  passenger_enabled on;
  passenger_app_env production;
}
or if digitalocean

server {
  listen 80;
  listen [::]:80;

  server_name _;
  root /home/deploy/yourappname/current/public;
  passenger_enabled on;
  passenger_app_env production;
}
```

Start Nginx:

```sh
sudo service nginx start
```

---

## 7. Setup PostgreSQL

```sh
sudo -i -u postgres
createuser -s a_new_user
psql
ALTER USER a_new_user WITH PASSWORD 'your_secure_password';
\q
createdb yourappname_production -O a_new_user
```

Or:

```sh
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql.service
sudo -i -u postgres
psql
```

Create user and database:

```sh
sudo -u postgres createuser --interactive
sudo -u postgres psql
psql=# alter user <username> with encrypted password '<password>';
createdb sammy
GRANT ALL PRIVILEGES ON DATABASE your_database_name TO your_new_username;
```

---

## 8. Deploy App with Capistrano

> **Note:**  
> If you did not deploy your app in step 6, then follow all these steps.  
> Otherwise, just follow step 9 if needed.

```sh
cap production deploy
```

---

## 9. Set Permissions (sometimes needed)

```sh
sudo chmod 755 /home/ubuntu
sudo chown -R ubuntu:ubuntu /home/ubuntu/yourappname
sudo chmod -R 755 /home/ubuntu/yourappname
sudo chown -R ubuntu:ubuntu /home/ubuntu/yourappname/current
sudo less /var/log/nginx/error.log
```

---

## 10. Example Nginx Config for Production

```sh
cd /etc/nginx/sites-enabled
sudo nano default
```

Example config:

```
server {
  listen 80;
  listen [::]:80;

  server_name _;
  root /home/ubuntu/yourappname/current/public;

  passenger_enabled on;
  passenger_app_env production;

  # Allow uploads up to 100MB in size
  client_max_body_size 100m;

  location ~ ^/(assets|packs) {
    expires max;
    gzip_static on;
  }
}
```

Restart Nginx:

```sh
sudo service nginx start
```
