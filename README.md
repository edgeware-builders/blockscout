## Edgscan: Blockscout for Edgeware

This repository is a snapshot of the Blockscout EVM explorer for
deployment to Edgeware. Please see the upstream Blockscout repo
for the unmodified README: https://github.com/blockscout/blockscout

For simplicity, this runbook does not include `sudo` instructions.
You should make sure to use sudo where appropriate (assuming you're
setting up Blockscout as a different user than root).

### Setup Instructions

Recommended server configuration: Minimum of 2048 MB of RAM, 50GB of disk space.

Install dependencies:
```
apt update
apt install -y automake autoconf libtool nodejs npm ruby jq postgresql-12 make g++ inotify-tools
```

Install Rust:
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

Install an up-to-date version of Erlang and Elixir. Make sure the Erlang version is 23 and Elixir version is 1.11.x. To see available versions, use for example `apt list esl-erlang -a`.

Make sure to run the commands in this order, or you may end up with Erlang OTP 24 which is not compatible with some packages.
```
wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb && sudo dpkg -i erlang-solutions_2.0_all.deb
apt update
apt install -y elixir=1.11.1-1
apt install -y esl-erlang=1:23.1-1
```

Set up a database:
```
sudo -i -u postgres
psql postgres -c "CREATE ROLE blockscout WITH LOGIN PASSWORD 'blockscout'; ALTER ROLE blockscout CREATEDB; ALTER ROLE blockscout WITH superuser;"
psql postgres -h 127.0.0.1 -U blockscout -c "CREATE DATABASE blockscout"
```

The password is 'blockscout', which you will have to enter at the end of the database setup.

Now quit out of the postgres user shell.

Install Blockscout (in the last step, Mix will prompt you to install Hex, which you should do):
```
git clone https://github.com/edgeware-builders/blockscout
cd blockscout
mix do deps.get, local.rebar --force, deps.compile, compile
```

Create and migrate a database (to reset the database, use `mix do ecto.drop, ecto.create, ecto.migrate`):
```
export DATABASE_URL=postgresql://blockscout:blockscout@localhost:5432/blockscout
mix do ecto.create, ecto.migrate
```

Build the web app:
```
cd apps/block_scout_web/assets
npm install
node --max_old_space_size=4096 node_modules/webpack/bin/webpack.js --mode production
cd -
```

Build static assets for deployment:
```
mix phx.digest
```

Enable HTTPS. The server must run with HTTPS.
This will create a self-signed certificate for localhost:
```
cd apps/block_scout_web
mix phx.gen.cert blockscout blockscout.local
cd -
```

Generate a secret:
```
mix phx.gen.secret
export SECRET_KEY_BASE=[the secret that was just generated]
```

Set up environment variables. We start the server on port 4000 to add
an SSL proxy later:
```
export PORT=4000
export ETHEREUM_JSONRPC_VARIANT=parity
export ETHEREUM_JSONRPC_HTTP_URL=http://beresheet1.edgewa.re:9933
export ETHEREUM_JSONRPC_WS_URL=ws://beresheet1.edgewa.re:9944
export BLOCKSCOUT_HOST=edgscan.com
export DATABASE_URL=postgresql://blockscout:blockscout@localhost:5432/blockscout
export NETWORK=EDG
export SUBNETWORK="Beresheet"
export CHAIN_ID=2022
export COIN=EDG
export COINGECKO_COIN_ID=edgeware
export RELEASE_LINK=https://github.com/edgeware-builders/edgscan
export LOGO=/images/blockscout_logo.svg
export LOGO_FOOTER=/images/blockscout_logo.svg
export SHOW_PRICE_CHART=false
export SHOW_TXS_CHART=true
export ENABLE_TXS_STATS=true
export SUPPORTED_CHAINS='[ {"title": "Edgeware", "url": "https://mainnet.edgscan.com"}, {"title": "Beresheet", "url": "https://beresheet.edgscan.com", "test_net?": true }]'
```

Start the server to make sure everything works:
```
mix phx.server
```

It will take some time to index the current chain.

In the meantime, you should set up Blockscout to run on boot using systemd:
```
echo "[Unit]
Description=Blockscout
[Service]
Type=simple
User=$USER
Group=$USER
Restart=on-failure
Environment=MIX_ENV=prod
Environment=LANG=en_US.UTF-8
Environment=PORT=4000
Environment=ETHEREUM_JSONRPC_VARIANT=parity
Environment=ETHEREUM_JSONRPC_HTTP_URL=http://beresheet1.edgewa.re:9933
Environment=ETHEREUM_JSONRPC_WS_URL=ws://beresheet1.edgewa.re:9944
Environment=BLOCKSCOUT_HOST=edgscan.com
Environment=DATABASE_URL=postgresql://blockscout:blockscout@localhost:5432/blockscout
Environment=NETWORK=EDG
Environment=SUBNETWORK="Beresheet"
Environment=CHAIN_ID=2022
Environment=COIN=EDG
Environment=COINGECKO_COIN_ID=edgeware
Environment=RELEASE_LINK=https://github.com/edgeware-builders/edgscan
Environment=LOGO=/images/edgeware_logo.svg
Environment=LOGO_FOOTER=/images/edgeware_logo.svg
Environment=SHOW_PRICE_CHART=false
Environment=SHOW_TXS_CHART=true
Environment=ENABLE_TXS_STATS=true
Environment=SECRET_KEY_BASE=`mix phx.gen.secret`
Environment=SUPPORTED_CHAINS='[ {\"title\": \"Edgeware\", \"url\": \"https://mainnet.edgscan.com\"}, {\"title\": \"Beresheet\", \"url\": \"https://beresheet.edgscan.com\", \"test_net?\": true }]'
Environment=PATH=/root/.cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WorkingDirectory=`pwd`
ExecStart=mix phx.server
[Install]
WantedBy=multi-user.target" > /etc/systemd/system/blockscout.service
```

Start the service:
```
systemctl daemon-reload
systemctl start blockscout.service
```

To check the status of the service:
```
systemctl status blockscout.service
```

Blockscout output will go to syslog. Use `tail -f /var/log/syslog` to
see the latest trailing output.

If the service isn't running check that the WorkingDirectory is set
correctly and mix is configured in the correct location.

### Configuring SSL using Certbot

Set the intended hostname of your server. This will be used in later steps:
```
export NAME=beresheet.edgscan.com
```

Install nginx:
```
apt install -y certbot python3-certbot-nginx
```

Configure nginx:
```
echo "server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_bind \$server_addr;
        proxy_buffering off;
    }
}" > /etc/nginx/sites-enabled/default
```

Restart nginx with the new config:
```
nginx -t && nginx -s reload
```

Start the certbot process for getting a new certificate:
```
certbot --nginx -d $NAME
```

Set up a crontab to renew your Certbot certificate:
```
crontab -e
```

Add this line to the crontab:
```
0 12 * * * /usr/bin/certbot renew --quiet
```

### Known Issues

* Internal transactions are not indexed because the API call is not
  provided by the current version of the Frontier EVM runtime used
  by Edgeware.
* The libsecp256k1 package does not compile correctly on Linux, which
  causes "Failed to load NIF library" errors. This is a known issue:
  https://github.com/exthereum/exth_crypto/issues/8