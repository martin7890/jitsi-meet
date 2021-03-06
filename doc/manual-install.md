# Server Installation for Jitsi Meet

This describes configuring a server `jitsi.example.com` running Debian or a Debian Derivative. You will need to
change references to that to match your host, and generate some passwords for
`YOURSECRET1`, `YOURSECRET2`, `YOURSECRET3` and `YOURSECRET4`.

There are also some complete [example config files](https://github.com/jitsi/jitsi-meet/tree/master/doc/example-config-files/) available, mentioned in each section.

## Install prosody
```sh
apt-get install prosody
```

## Configure prosody
Add config file in `/etc/prosody/conf.avail/jitsi.example.com.cfg.lua` :

- add your domain virtual host section:

```
VirtualHost "jitsi.example.com"
    authentication = "anonymous"
    ssl = {
        key = "/var/lib/prosody/jitsi.example.com.key";
        certificate = "/var/lib/prosody/jitsi.example.com.crt";
    }
    modules_enabled = {
        "bosh";
        "pubsub";
    }
```
- add domain with authentication for conference focus user:
```
VirtualHost "auth.jitsi.example.com"
    authentication = "internal_plain"
```
- add focus user to server admins:
```
admins = { "focus@auth.jitsi.example.com" }
```
- and finally configure components:
```
Component "conference.jitsi.example.com" "muc"
Component "jitsi-videobridge.jitsi.example.com"
    component_secret = "YOURSECRET1"
Component "focus.jitsi.example.com"
    component_secret = "YOURSECRET2"
```

Add link for the added configuration
```sh
ln -s /etc/prosody/conf.avail/jitsi.example.com.cfg.lua /etc/prosody/conf.d/jitsi.example.com.cfg.lua
```

Generate certs for the domain:
```sh
prosodyctl cert generate jitsi.example.com
```

Create conference focus user:
```sh
prosodyctl register focus auth.jitsi.example.com YOURSECRET3
```

Restart prosody XMPP server with the new config
```sh
prosodyctl restart
```

## Install nginx
```sh
apt-get install nginx
```

Add a new file `jitsi.example.com` in `/etc/nginx/sites-available` (see also the example config file):
```
server_names_hash_bucket_size 64;

server {
    listen 80;
    server_name jitsi.example.com;
    # set the root
    root /srv/jitsi.example.com;
    index index.html;
    location ~ ^/([a-zA-Z0-9=\?]+)$ {
        rewrite ^/(.*)$ / break;
    }
    location / {
        ssi on;
    }
    # BOSH
    location /http-bind {
        proxy_pass      http://localhost:5280/http-bind;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $http_host;
    }
}
```

Add link for the added configuration
```sh
cd /etc/nginx/sites-enabled
ln -s ../sites-available/jitsi.example.com jitsi.example.com
```

## Install Jitsi Videobridge
```sh
wget https://download.jitsi.org/jitsi-videobridge/linux/jitsi-videobridge-linux-{arch-buildnum}.zip
unzip jitsi-videobridge-linux-{arch-buildnum}.zip
```

Install JRE if missing:
```
apt-get install default-jre
```

_NOTE: When installing on older Debian releases keep in mind that you need JRE >= 1.7._

In the user home that will be starting Jitsi Videobridge create `.sip-communicator` folder and add the file `sip-communicator.properties` with one line in it:
```
org.jitsi.impl.neomedia.transform.srtp.SRTPCryptoContext.checkReplay=false
```

Start the videobridge with:
```sh
./jvb.sh --host=localhost --domain=jitsi.example.com --port=5347 --secret=YOURSECRET1 &
```
Or autostart it by adding the line in `/etc/rc.local`:
```sh
/bin/bash /root/jitsi-videobridge-linux-{arch-buildnum}/jvb.sh --host=localhost --domain=jitsi.example.com --port=5347 --secret=YOURSECRET1 </dev/null >> /var/log/jvb.log 2>&1
```

## Install Jitsi Conference Focus (jicofo)

Install JDK and Ant if missing:
```
apt-get install default-jdk ant
```

_NOTE: When installing on older Debian releases keep in mind that you need JDK >= 1.7._

Clone source from Github repo:
```sh
git clone https://github.com/jitsi/jicofo.git
```
Build distribution package. Replace {os-name} with one of: 'lin', 'lin64', 'macosx', 'win', 'win64'.
```sh
cd jicofo
ant dist.{os-name}
```
Run jicofo:
```sh
cd dist/{os-name}'
./jicofo.sh --domain=jitsi.exmaple.com --secret=YOURSECRET2 --user_domain=auth.jitsi.example.com --user_name=focus --user_password=YOURSECRET3
```

## Deploy Jitsi Meet
Checkout and configure Jitsi Meet:
```sh
cd /srv
git clone https://github.com/jitsi/jitsi-meet.git
mv jitsi-meet/ jitsi.example.com
```

Edit host names in `/srv/jitsi.example.com/config.js` (see also the example config file):
```
var config = {
    hosts: {
        domain: 'jitsi.example.com',
        muc: 'conference.jitsi.example.com',
        bridge: 'jitsi-videobridge.jitsi.example.com'
    },
    useNicks: false,
    bosh: '//jitsi.example.com/http-bind', // FIXME: use xep-0156 for that
    desktopSharing: 'false' // Desktop sharing method. Can be set to 'ext', 'webrtc' or false to disable.
    //chromeExtensionId: 'diibjkoicjeejcmhdnailmkgecihlobk', // Id of desktop streamer Chrome extension
    //minChromeExtVersion: '0.1' // Required version of Chrome extension
};
```

Restart nginx to get the new configuration:
```sh
invoke-rc.d nginx restart
```

## Running behind NAT
In case of videobridge being installed on a machine behind NAT, add the following extra lines to the file `~/.sip-communicator/sip-communicator.properties` (in the home of user running the videobridge):
```
org.jitsi.videobridge.NAT_HARVESTER_LOCAL_ADDRESS=<Local.IP.Address>
org.jitsi.videobridge.NAT_HARVESTER_PUBLIC_ADDRESS=<Public.IP.Address>
```

So the file should look like this at the end:
```
org.jitsi.impl.neomedia.transform.srtp.SRTPCryptoContext.checkReplay=false
org.jitsi.videobridge.NAT_HARVESTER_LOCAL_ADDRESS=<Local.IP.Address>
org.jitsi.videobridge.NAT_HARVESTER_PUBLIC_ADDRESS=<Public.IP.Address>
```

# Hold your first conference
You are now all set and ready to have your first meet by going to http://jitsi.example.com


## Enabling recording
Currently recording is only supported for linux-64 and macos. To enable it, add
the following properties to sip-communicator.properties:
```
org.jitsi.videobridge.ENABLE_MEDIA_RECORDING=true
org.jitsi.videobridge.MEDIA_RECORDING_PATH=/path/to/recordings/dir
org.jitsi.videobridge.MEDIA_RECORDING_TOKEN=secret
```

where /path/to/recordings/dir is the path to a pre-existing directory where recordings
will be stored (needs to be writeable by the user running jitsi-videobridge),
and "secret" is a string which will be used for authentication.

Then, edit the Jitsi-Meet config.js file and set:
```
enableRecording: true
```

Restart jitsi-videobridge and start a new conference (making sure that the page
is reloaded with the new config.js) -- the organizer of the conference should
now have a "recoriding" button in the floating menu, near the "mute" button.
