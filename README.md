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

First, run:

```sh
sudo apt-get update
```

Then, install the required packages:

```sh
sudo apt-get install curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates nodejs yarn
```

Finally, install the PostgreSQL development library:

```sh
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
    > nano ~/.bashrc
    > ```

    To verify if rbenv is installed correctly, run:
    ```sh
    type rbenv
    ```
    The output should be similar to:
    ```
    rbenv is a function
    rbenv ()
    {
        local command;
        command="${1:-}";
        if [ "$#" -gt 0 ]; then
            shift;
        fi;
        case "$command" in
            rehash | shell)
                eval "$(rbenv "sh-$command" "$@")"
            ;;
            *)
                command rbenv "$command" "$@"
            ;;
        esac
    }
    ```
    This confirms rbenv is installed

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

First, choose the command for your Ubuntu version and run it:

```sh
# For Ubuntu 18.04 (bionic)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list'
# For Ubuntu 22.04 (jammy)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger jammy main > /etc/apt/sources.list.d/passenger.list'
# For Ubuntu 24.04 (noble)
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger noble main > /etc/apt/sources.list.d/passenger.list'
```

Then, update your package list:

```sh
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

## 6. Add SSH Key to Server

Before deploying, you need to add your SSH key to the server:

1. Generate a new SSH key (press Enter for all prompts, no need to add anything):
    ```sh
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    ```

2. Display your public SSH key:
    ```sh
    cat ~/.ssh/id_rsa.pub
    ```

3. Copy the output and add it to your GitHub account (the one you use to deploy the app) under **Settings > SSH and GPG keys > New SSH key**.

---


---

## 7. Setup PostgreSQL

```sh
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql.service
sudo -i -u postgres
psql
```
If run correctly, use `\q` to quit psql and `exit` to quit the postgres user session.
```sh
\q
exit
```

Make a user by:

```sh
sudo -i -u postgres
createuser -s a_new_user
psql
ALTER USER a_new_user WITH PASSWORD 'your_secure_password';
\q
createdb yourappname_production -O a_new_user
```

Or:


Create user and database:

```sh
sudo -u postgres createuser --interactive
sudo -u postgres psql
psql=# alter user <username> with encrypted password '<password>';
createdb sammy
GRANT ALL PRIVILEGES ON DATABASE your_database_name TO your_new_username;
```

---

## 8. Configure Nginx (deploy app before this step)

```sh
sudo apt install nginx-full
```
> ⚠️ **Before this step:**  
> You should configure Capistrano and deploy your app to the server.  
> If you need help setting up Capistrano on your local machine, see [this guide the setup-Capistrano-in-rails-app](https://github.com/Tallalali1/setup-Capistrano-in-rails-app).

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
you might need to stop and start nginx again by running
```sh
sudo service nginx stop
sudo service nginx start
```

## 9. Deploy App with Capistrano

> **Note:**  
> If you did not deploy your app in step 7, then follow all these steps.  
> Otherwise, just follow step 10 if needed.

```sh
cap production deploy
```

---

## 10. Set Permissions (sometimes needed)

```sh
sudo chmod 755 /home/ubuntu
sudo chown -R ubuntu:ubuntu /home/ubuntu/yourappname
sudo chmod -R 755 /home/ubuntu/yourappname
sudo chown -R ubuntu:ubuntu /home/ubuntu/yourappname/current
sudo less /var/log/nginx/error.log
```

---

## 11. Example Nginx Config for Production

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
