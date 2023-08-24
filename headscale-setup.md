# Headscale
#### An open source, self-hosted implementation of the Tailscale control server.

https://github.com/juanfont/headscale

## Setup Control Server
1. Install docker - this is the easiest way to install with less configurations
https://docs.docker.com/engine/install/
2. Configure and run https://headscale.net/running-headscale-container/#configure-and-run-headscale
```bash
mkdir -p ./headscale/config
cd ./headscale
touch ./config/db.sqlite

# Download sample config file
curl https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml -o ./config/config.yaml

# Modify the config file to your preferences before launching Docker container. Here are some settings that you likely want:

server_url: https://your-host-name.com
metrics_listen_addr: 0.0.0.0:9090
private_key_path: /etc/headscale/private.key
noise:
  private_key_path: /etc/headscale/noise_private.key
db_type: sqlite3
db_path: /etc/headscale/db.sqlite
```

Run
```
docker run \
  --name headscale \
  --detach \
  --restart unless-stopped \
  --volume /home/ubuntu/headscale/config:/etc/headscale/ \
  --publish 0.0.0.0:8080:8080 \
  --publish 0.0.0.0:9090:9090 \
  headscale/headscale:latest \
  headscale serve
```

3. Headscale UI
```
docker run --restart unless-stopped -d --name headscale-ui -p 9443:443 ghcr.io/gurucomputing/headscale-ui:latest
```

4. Caddy - for reverse proxy
https://caddyserver.com/docs/install
- create file named 'Caddyfile' and paste the config below
- `caddy stop` - to stop caddy
- `sudo caddy start` - to start caddy
- `sudo caddy reload` - to reload Caddyfile config
```
# Caddyfile config
your-host-name.com {
  reverse_proxy /web* https://127.0.0.1:9443 {
    transport http {
      tls_insecure_skip_verify
    }
  }
  reverse_proxy 127.0.0.1:8080
}
```

## Generate API key
this will be used to access data on Headscale UI
```
docker exec headscale headscale apikeys create
```
- Navigate to https://your-host-name.com/web/settings.html
- Paste the generated apikey 
![Alt Text](https://s11.gifyu.com/images/SgEwU.gif)


## Add User
- Navigate to User View https://your-host-name.com/web/users.html
- Click on + New User and save


## Generate Preauth Keys - will be used by the client to add machine
- Click on the user to expand additional menus
- Click on the **plus sign** beside 'Preauth Keys'
- Update the **Expiry date** to your desired date
- Check the **Reusable** checkbox
- Click **Create Preauth Key**
- Key will be generated

## Client Setup
1. Download Tailscale client
https://tailscale.com/download/
2. Open command line and execute command
```
tailscale up \
  --login-server https://your-host-name.com \
  --authkey <generated_preauth_key> \
  --reset --accept-routes 
```
To advertise routes
```
tailscale up \
  --login-server https://your-host-name.com \
  --authkey <generated_preauth_key> \
  --advertise-routes=<network_address> \
  --reset --accept-routes 
  
# example:
tailscale up \
  --login-server https://headscale.magisinov.com \
  --authkey qwertyuiop123456789 \
  --advertise-routes=192.168.30.0/24,192.168.80.0/24  \
  --reset --accept-routes 
```


## Enable Route advertised by a client
- Navigate to https://your-host-name.com/web/devices.html
- Click on a device to expand more options
- Beside the Device Routes will be the address advertised
- Click on the button to Enable Route
![Alt Text](https://s11.gifyu.com/images/SgE3L.gif)

## Solution for overlapping ipv4 addresses
https://tailscale.com/kb/1201/4via6-subnets/

```
# generate IPv6 subnet route
# tailscale debug via <site_id_1_to_255> <network_cidr_address>
tailscale debug via 1 192.168.55.0/24

# advertise route
tailscale up --advertise-routes=<generated_ipv6> --reset --accept-routes --login-server https://your-host-name.com --authkey <preauth_key>
```

Test by ping
```
# ping -6 <ipv6_address> 
ping -6 fd7a:115c:a1e0:b1a:0:1:c0a8:140 

# ping -6 <magicdns_format> 
ping -6 192.168.1.64.via-2 
```
