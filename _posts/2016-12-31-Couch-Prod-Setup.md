---
layout: post
title: Convoluted CouchDB setup
---

So over the past few weeks I've been trying to use CouchDB as the backend for an app I've been working on. CouchDB brings native speed, convenient sync and automatic conflict resolving. However, as CouchDB opens up to the world, how to secure the connections and how to have coordinated access control remains to be a challange. I think I've cracked it and here's how: 

### General Concept:
 - A Python Flask based server acts as a CouchDB front to receive all sync requests, authenticate them via token and username provided in the HTTP Headers, then use requests to query the actual CouchDB. 
 - The actual CouchDB is hosted on an EC2 instance, behind an NGINX proxy. NGINX is there to provide client side SSL Authentication so only the Python Flask server will have access. 
 - The client CouchDB can set arbitrary headers, making it perfect for Flask's token based authentication. 
 
### Setup Details:
#### Python Flask:
        url_options = request.url.replace(request.base_url, "")
        r = requests.get(COUCHDB_URL + request.path + url_options,
            headers = request.headers, 
            cert = ('./client.crt', './client.key'), verify = False)
            
 - .cert and .key are there for client side SSL Authentication
 - we are basically relaying the CouchDB sync request to the central CouchDB on EC2. Before requesting, we have the luxury of authenticating and throttling and even device number limiting our clients. 
 
#### CouchDB:
Set it up on another server, where NGINX will live.

Instruct CouchDB to use proxy authentication, as this is exactly what we do here. 

    authentication_handlers = {couch_httpd_oauth, oauth_authentication_handler}, {couch_httpd_auth, cookie_authentication_handler}, {couch_httpd_auth, proxy_authentication_handler}, {couch_httpd_auth, default_authentication_handler}

#### NGINX Proxy:
    # HTTPS server

    server {
        listen       443 ssl;
        ssl on;
        server_name  localhost;

        ssl_certificate      /usr/local/etc/nginx/certs/server.crt;
        ssl_certificate_key  /usr/local/etc/nginx/certs/server.key2;

        ssl_client_certificate /usr/local/etc/nginx/certs/ca.crt;
        ssl_verify_client on;
        # ssl_verify_depth 0;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        # auth_basic "Restricted Content";
        # auth_basic_user_file /usr/local/etc/nginx/certs/htpasswd;

        location / {
           proxy_pass http://127.0.0.1:5986;
           proxy_pass_request_headers      on;
        }
    }

- ssl_client_certificate and ssl_verify_client does client side SSL Authentication.
- location / proxies to CouchDB. 

#### Generating NGINX client SSL Certificates:
- Create the CA Key and Certificate for signing Client Certs

      openssl genrsa -des3 -out ca.key 4096
      openssl rsa -in ca.key -out ca.key2
      openssl req -new -x509 -days 365 -key ca.key2 -out ca.crt

- Create the Server Key, CSR, and Certificate

      openssl genrsa -des3 -out server.key 1024
      openssl rsa -in ca.key -out server.key2
      openssl req -new -key server.key2 -out server.csr

- We're self signing our own server cert here.  This is a no-no in production.

      openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt

- Create the Client Key and CSR

      openssl genrsa -des3 -out client.key 1024
      openssl rsa -in ca.key -out client.key2
      openssl req -new -key client.key2 -out client.csr

- Sign the client certificate with our CA cert.  Unlike signing our own server cert, this is what we want to do.

      openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key2 -set_serial 01 -out client.crt

About how to do crl: [https://arcweb.co/securing-websites-nginx-and-client-side-certificate-authentication-linux/]


### How to instruct client CouchDB to use our authentication system.

The key is to set the headers for sync requests:

- CouchDB http:
   
   {"source":"testdb","target":{"url":"http://127.0.0.1:5000/couchfront/testdb","headers":{"X-Auth-CouchDB-UserName":"angke","X-Auth-CouchDB-Roles":"_admin","X-Auth-CouchDB-Token":"random_token"}}}
   
- PouchDB:

    db = new PouchDB('my_database');
    var remoteCouch = 'http://127.0.0.1:5000/couchfront/testdb'

    function syncError(error) {
     console.log("Sync Error: " + error);
    }
    function sync() {
      console.log("syncing");
      var opts = {
       live: false, 
       ajax: { headers: {
        "X-Auth-CouchDB-UserName": "angke",
        "X-Auth-CouchDB-Roles": "_admin",
        "X-Auth-CouchDB-Token": "random_token"
       }}};
      db.replicate.to(remoteCouch, opts, syncError);
      // db.replicate.from(remoteCouch, opts, syncError);
      console.log("done");
    }
   
- CouchBase Lite (iOS, Android):

    NSURL *url = [NSURL URLWithString: @"http://127.0.0.1:5000/couchfront/testdb"];
    CBLReplication *push = [database createPushReplication: url];
    CBLReplication *pull = [database createPullReplication: url];
    // push.continuous = YES;
    // pull.continuous = YES;
    
    // Add authentication.
    pull.headers = [[NSDictionary alloc] initWithObjectsAndKeys:@"angke", @"X-Auth-CouchDB-UserName",
                    @"_admin", @"X-Auth-CouchDB-Roles", @"random_token", @"X-Auth-CouchDB-Token", nil];
    push.headers = pull.headers;
    
    // Start replicating. 
    [push start];
    [pull start];
