| Category | Component      | Purpose         | Technology                |
| :------: | :------------- | :-------------- | :------------------------ |
| App      | Higala         | Frontend        | Vue(x), NPM, Node.js, ... |  
| App      | Zimmerman      | Backend         | Flask                     |
| App      | "the database" | being databasey | sqllite                   |
| App      | Nginx          | webserver       | nginx                     |  
| Hosting  | DNS Registrar  | hosting konishi.club          | some hosting party             |  
| Hosting  | Cloudflare     | websecurity, ssl, flexibility | cloudflare free account        |   
| Hosting  | VM             |  Hosting the apps             | AWS Instance; Ubuntu 18.04 LTS |  

# Current architecture
This document is written to keep track of the current development build, including all the configurations necessary 
to make it work, so that anyone can pick up where I left off / contribute more easily.

## Hosting
The instance is currently hosted on an AWS instance provided by `Michael-Lloyd`. This instance hosts all of the 
"app" components at this time, and is running Ubuntu 18.04 LTS.

The domain name __konishi.club__ is hosted by `LeetCodes`.  
His DNS points towards the Cloudflare registered by `dwrolvink`, where it routes __konishi.club__ to the AWS instance 
for serving the frontend, and __backend.konishi.club__ (to the same instance) for serving zimmerman.

## App architecture
### Zimmerman (backend)
As seen above, zimmerman is a Flask app, and doesn't really require much besides some python modules to be loaded. 
It is responsible for handling all requests, and thus also connects to the sqllite db, which is merely a file in the same
folder as zimmerman.

On the vm, zimmerman is set to `app.run(host='0.0.0.0', port=4000)` so it will accept traffic from all sources on port 4000.
Though, this will be changed in the future when nginx is working properly. This is because nginx actually routes traffic to 
zimmerman from the url `backend.konishi.club:80` and does a `pass_proxy http://127.0.0.1:4000$request_uri`, so port 4000 is not
necessary to be opened on the vm, and poses added security risk. The `host='0.0.0.0'` is currently set so troubleshooting 
can be done while better configuring nginx.
 
### Higala (frontend)
Higala is a VueJS app, and requires a lot of packages to be installed correctly. A pointer on Ubuntu is that the nodejs ppa 
has to be added before installing NPM so that the latest versions are installed (^6.5.0). 

Higala is installed, served, and build for production using NPM. In the current setup, Higala is never run directly from NPM, 
but built using NPM, whereupon the `dist/` folder will be overwritten with the new site. The contents of that folder are copied
over to `var/www/konishi.club` to be served by nginx.

### Nginx (webserver)
The current siteblock setup is:
```
server {
	listen 80;
	listen [::]:80;
	server_name backend.konishi.club;

	location / {
		proxy_pass http://127.0.0.1:4000$request_uri;
		proxy_set_header Host $host;
	}
}

server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name konishi.club www.konishi.club;
	
	location / {
		index index.html;
		try_files $uri $uri/ =404;
		root /var/www/konishi.club;
	}
}
```
As can be clear from above, there is a lot of work left to do on nginx. Research has to be done on adding more security using
headers, general configuration, and increasing performance by having it spawn multiple worker processes.

### DNS and Cloudflare
As can be seen from the siteblock, all traffic should enter at port 80. This will be 443 when SSL has been configured. At the
moment, the before mentioned port 4000 is also still open, but this doesn't concern the DNS or Cloudflare atm.

The current provider for the DNS hosting is unknown, but it does not matter, they all work the same anyways. The one important
thing is that it will have two DNS servers of the Cloudflare account in it's DNS list, and that it doesn't set any records 
itself. That way, all DNS queries for the site will be sent to Cloudflare's DNS servers, instead of being answered by the DNS
Hoster itself.

So, you open your browser to `konishi.club`, and it polls the DNS hosting, which will query Cloudflare. What happens there?

Very little atm. It returns a cloudflare ip which will mediate traffic to and from the VM at our side. This keeps our IP
secure. Cloudflare also has a couple of standard functionalities where it can, for example, cache your website. It also has
auto rewrite rules, which could force all traffic to HTTPS. At the moment, caching is disabled (dev mode), and no rules are
configured.

SSL will also be configured on Cloudflare, as the free Cloudflare account offers free high quality SSL certificates.

### AWS instance
The AWS instance is pretty basic. Ports 80, 443, 22, and 4000 are opened on the security group. No elastic ip is configured
yet, but we might get around to that.

# Installation
I wrote a little package to automate the installation of Konishi front-to-back. Next to that, there is a freshpull.sh file,
which allows you to reset any troubleshooting changes you might have made on the server, whilst keeping all of the untracked
files (like uploaded pictures and the entire database). 

I'll put the package here when I think it's polished enough. ATM, I don't want anyone else editing it as then I'll lose
oversight. The documentation atm is all contained within the config / deployment files, so I'll have to write a full manual 
for that.

The files can be found here https://github.com/dwrolvink/gitautomata (konishi/nginx folders)




