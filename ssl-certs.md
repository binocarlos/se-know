ssl-certs
---------

steps for generating and plugging in ssl certificates.

you will need a linux server with openssl installed

## step 1 - generate private key

the private key is how we encode the SSL cert using SSH

to generate one first change to the folder you keep the keys for your project:

```
$ cd /my/project/keys
```

then generate the key:

```
$ openssl genrsa -des3 -out www.myapp.com.key 2048
```

enter a password

check the key:

```
$ cat www.myapp.com.key
```

## step 2 - generate csr

csr stands for 'certificate signing request'

you need to send this to the people you buy the SSL cert from - it tells them who you are

to make one from the private key:

```
$ openssl req -new -key www.myapp.com.key -out www.myapp.com.csr
```

## step 3 - get the signed certs back

send the csr to the ssl provider - they will return 3 files:

 * www.myapp.com.crt - the signed certificate
 * PositiveSSLCA2.crt (providername.crt) - the provider certs
 * AddTrustExternalCARoot.crt - the root certs

there are other files that are combinations:

 * *-bundle.ca-bundle - the provider cert and root certs combined


## step 4 - combine certs

we now need to create a certificate chain from the 3 files listed above:

```
$ cat www.myapp.com.crt > www.myapp.com.deploy.crt
$ cat PositiveSSLCA2.crt >> www.myapp.com.deploy.crt
$ cat AddTrustExternalCARoot.crt >> www.myapp.com.deploy.crt
```

## step 5 - remove key password

now we have the app ready to deploy we want to remove the password from the key for auto-restarts

```
$ openssl rsa -in www.myapp.com.key -out www.myapp.com.deploy.key
```

## step 6 - deploy the keys

now the keys are made you can use them in any context - here are some examples:

### node

```js
var options = {
  key: fs.readFileSync(ssldir + '/www.myapp.com.deploy.key').toString(),
  cert: fs.readFileSync(ssldir + '/www.myapp.com.deploy.crt').toString()
}

https.createServer(options,app).listen(sslport, function(){
  console.log('funkybods HTTPS website listening: ' + sslport);
});

```

### nginx

```
server {
  listen 80;
  listen      443 ssl;
  server_name www.myapp.com;
  root /home/myapp/www;
  index index.html index.htm;
  ssl on;
  ssl_certificate /home/myapp/keys/www.myapp.com.deploy.crt;
  ssl_certificate_key /home/myapp/keys/www.myapp.com.deploy.key;
}
```

### quarry

```yaml
website:
  type: worker
  dockerfile: |
    FROM quarry/monnode
    ADD . /srv/app
    RUN cd /srv/app && npm install
    WORKDIR /srv/app
    ENTRYPOINT NODE_ENV=production mon "node ./src/index.js --port 80"
  ssl_key: ./keys/www.myapp.com.nopass.key
  ssl_cert: ./keys/www.myapp.com.nopass.crt
  domains:
    - "www.myapp.com"
```

# test certs

you can bypass the full loop above for testing - you will still want to remove the password from the key:


```
$ openssl x509 -req -days 365 -in www.myapp.com.csr -signkey www.myapp.deploy.key -out www.myapp.deploy.crt
```