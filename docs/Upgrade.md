#Virtual Machine upgrade instructions for ripple-rest versions 1.24 and below 

##Step by Step Instructions:
1.  sudo -s
1.  /etc/init.d/ripple-rest stop
1.  cd /opt
1.  rm -rf ripple-rest
1.  git clone https://github.com/ripple/ripple-rest
1.  cd ripple-rest
1.  npm install
1.  touch ripple-rest.db
1.  cp config-example.js config.js
1. Edit new config.js with the following changes
  * ` "host" : "0.0.0.0",`
  * ` "ssl_enabled": true,`
  *   ` "ssl": {
    "key_path": "/etc/ssl/private/server.key",
    "cert_path": "/etc/ssl/certs/server.crt"
  },`
  * `  "rippled_servers": [
    "wss://fs1.ripple.com:443"
  ]`
1. chown restful:restful ripple-rest.db
1.  **REPLACE CURRENT** /etc/init.d/ripple-rest file with this [initscript](https://gist.github.com/n0rmz/a0075ab7a07a33ffdcae).
1. /etc/init.d/ripple-rest start`
