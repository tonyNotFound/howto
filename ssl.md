# Setting up SSL

Setting up SSL can be a pain. But it will save you headaches down the road, so do it.

To start, make a temp folder ~/ssl:

    mkdir ~/ssl
    cd ~/ssl

Next, generate the RSA key you will use for the request:

    openssl genrsa -out YOURSITE.com.key 2048


Then generate the certificate signing request (CSR):

    openssl req -new -key YOURSITE.com.key -out YOURSITE.com.csr

Answer all of the questions. Common name is the name of the domain you are generating a request for. If it is for your root domain use "www.example.com" not just "example.com". If the request is for a wildcard cert then use "*.example.com"

    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:US
    State or Province Name (full name) [Some-State]:YOUR STATE
    Locality Name (eg, city) []:YOUR CITY
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:YOUR COMPANY NAME
    Organizational Unit Name (eg, section) []:Engineering
    Common Name (e.g. server FQDN or YOUR name) []:www.YOURSITE.com
    Email Address []:GENERIC EMAIL FROM YOUR COMPANY

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    
I usually leave the password fields blank.

Now you have the Certificate request files. Go through a service (like [https://startssl.com](https://startssl.com) and use this request to get the actual certificate. Eventually you will save the result as `ssl.crt`. 

## Startssl

Startssl is a good option to use because the generated certs are approved and you can get a free cert for a single domain. The website can be sluggish if it is overloaded, but that is only a problem when generating your request. They are also a great option for Wildcard and Verified certs if you need them.

**Note**: StartSSL will work for the root domain and 1 subdomain only for the free option.

Go to:

    http://www.startssl.com/
    
You should see a link for "Free" certificates. Click this, and it will take you to

    http://www.startssl.com/?app=1
   
At the bottom of the text block you'll see a link to the [Certificate Control Panel](http://www.startssl.com/?app=12). Once there, proceed through the "Express Lane" and create a new account.

You'll get a confirmation code you must copy and paste. Once complete, the site will generate a "Client key" to install in your browser. Once created you'll need to click "Install".

Once you get through all of this you need to validate that you are the owner of the domain. Note, you need to validate that you are the owner of the __root__ domain (i.e, example.com), even if you plan on generating a certificate for a subdomain. This will use the registered email on the DNS (which you might have setup, or you might have a catch-all). An email is sent with a token which you need to enter to continue.

> Note: you probably shouldn't use abuse@ or postmaster@ if you are using Google Apps. Google reserves these addresses. Use webmaster@ instead.

Once you download `ssl.crt` into the folder, you will need to combine it with a couple other root certificates:

    wget http://www.startssl.com/certs/sub.class1.server.ca.pem
    wget http://www.startssl.com/certs/ca.pem
    cat ssl.crt sub.class1.server.ca.pem ca.pem > YOURSITE.crt

Now you have everything that you need, but you need to get it in chef. These files should be kept private and safe and you can do this by encrypting them. In your `chef` folder create a new encrypted data bag:

    knife solo data bag create certs YOURSITE

Here you can just use the domain name (without the www, or .com) or the subdomain name. You can also use a simpler scheme and just call it "production".

The data bag is JSON, which doesn't allow for multi-line strings. You'll need to replace the line ends with "\n" (the literal string, not a new line character).

In your site cookbook default attributes:

    certs = Chef::EncryptedDataBagItem.load("certs", "production")
    default['sample']['certs'] = {
      :key => certs['key'],
      :crt => certs['crt']
    }

In your site cookbook default recipe:

    ## SSL
    #
    directory "#{shared_dir}/certs" do
      action :create
      owner node['sample']['user']
      group node['sample']['group']
      mode '0755'
      recursive true
    end

    file "#{shared_dir}/certs/YOURSITE.key" do
      action :create
      owner node['sample']['user']
      group node['sample']['group']
      mode '0600'
      content node['sample']['certs']['key']
    end

    file "#{shared_dir}/certs/YOURSITE.crt" do
      action :create
      owner node['sample']['user']
      group node['sample']['group']
      mode '0644'
      content node['sample']['certs']['crt']
    end
   
# Unformatted 


## Let's Encrypt!

Let's Encrypt is all the new new.

_On Ubuntu 14.04 for Nginx_ 

https://www.nginx.com/blog/free-certificates-lets-encrypt-and-nginx/

.[https://certbot.eff.org/#ubuntutrusty-nginx](https://certbot.eff.org/#ubuntutrusty-nginx)

    sudo su deploy
    cd /home/deploy
    mkdir ssl
    cd ssl
    wget https://dl.eff.org/certbot-auto
    chmod a+x certbot-auto
    sudo service nginx stop # takes down the server
    ./certbot-auto certonly
    sudo service nginx start
    
    
Choose temporary webserver
https://rpl.cat/mWC85ufneufIqsbyEisgcwVKI8F4Pqy4I5SCbozz7g0

Enter your email
https://rpl.cat/OEkAKmJgS4-ZeqzlwRoFsc8fFoKfa11TuXwlHHCf9cU


Enter domain names:
https://rpl.cat/RySYznG66K65T1SUXxbvvXPACqyPlkDEn9ldUEUgDdE


## Certificate pinning / HSTS / etc

* http://stackoverflow.com/questions/16613081/ssl-pinning-with-afnetworking
* https://www.owasp.org/index.php/Pinning_Cheat_Sheet
* https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning
* http://initwithfunk.com/blog/2014/03/12/afnetworking-ssl-pinning-with-self-signed-certificates/
* http://stackoverflow.com/questions/15728636/how-to-pin-the-public-key-of-a-certificate-on-ios
* https://github.com/AFNetworking/AFNetworking/issues/1852   

Certificate pinning has downsides (key rotation, key expiration and revocation). In such cases it is better to use public key pinning.





```
server {
  listen      [::]:80;
  listen      80;
  server_name click.rpl.cat;

  location /.well-known/acme-challenge {
    root /var/www/letsencrypt;
  }

  return 301 https://click.rpl.cat$request_uri;
}

server {
  listen      [::]:80;
  listen      80;
  server_name api.rpl.cat;
  
  location /.well-known/acme-challenge {
    root /var/www/letsencrypt;
   }
   
   return 301 https://api.rpl.cat$request_uri;
}  

server {
  listen      [::]:443 ssl spdy;
  listen      443 ssl spdy;
  server_name api.rpl.cat;
  server_name click.rpl.cat;
  # ssl_certificate     /home/dokku/api.rpl.cat/tls/server.crt;
  # ssl_certificate_key /home/dokku/api.rpl.cat/tls/server.key;
  
   ssl_certificate /etc/letsencrypt/live/api.rpl.cat/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.rpl.cat/privkey.pem;
  
  
  keepalive_timeout   70;
  add_header          Alternate-Protocol  443:npn-spdy/2;
  location    / {     
    proxy_pass  http://api.rpl.cat;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection upgrade;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header X-Request-Start $msec;
  } 
  include /home/dokku/api.rpl.cat/nginx.conf.d/*.conf;
} 
upstream api.rpl.cat { server 172.17.0.175:5000; }
```



