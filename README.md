# podman-nginx-reverse-proxy
Simple configuration of a reverse proxy with nginx, httpd and podman

## Original tutorial
https://www.redhat.com/sysadmin/podman-nginx-multidomain-applications

## Actual steps

Pull the httpd and nginx images. Httpd will be used to serve the content and nginx as reverse proxy.

```bash

> podman pull docker.io/library/httpd
> podman pull docker.io/library/nginx

```

Create the main folder, three subfolders, two for the content and one for the proxy configuration.

```bash

> cd $HOME
> mkdir tutorial-podman
> cd tutorial-podman
> mkdir syscom syslog nginx 

```

Open your hosts file:

```bash
> sudo nano /etc/hosts
```

And add the following rules:

```
127.0.0.1	sysadmin.org
127.0.0.1	sysadmin.com
```

In /syscom add and index.html with the following content:

```html

<html>
  <header>
    <title>SysAdmin.com</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .com domain</p>
  </body>
</html>

```

In /sysorg add and index.html with the following content:

```html

<html>
  <header>
    <title>SysAdmin.org</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .org domain</p>
  </body>
</html>

```

Run two containers, one for each content serving folder:

```bash

> podman run --name=syscom -p 8080:80 -v $HOME/tutorial-podman/syscom:/usr/local/apache2/htdocs:Z -d docker.io/library/httpd

> podman run --name=syscom -p 8080:80 -v $HOME/tutorial-podman/syscom:/usr/local/apache2/htdocs:Z -d docker.io/library/httpd

> curl http://sysadmin.com
curl: (7) Failed to connect to sysadmin.com port 80 after 0 ms: Connection refused
> curl http://sysadmin.org
curl: (7) Failed to connect to sysadmin.org port 80 after 0 ms: Connection refused

> curl http://sysadmin.org:8081
<html>
  <header>
    <title>SysAdmin.org</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .org domain</p>
  </body>
</html>

> curl http://sysadmin.com:8080
<html>
  <header>
    <title>SysAdmin.com</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .com domain</p>
  </body>
</html>

```

Configure the reverse proxy:

```bash

> cd ./nginx
> touch default.conf syscom.conf sysorg.conf

```

Below is available the configuration for the three config files.
To retrieve the value for the "proxy_pass" I had to check 
the available IPs using ```ip a``` command. 
The ```podman inspect <container>``` was returning an empty ip.
I have to improve on this point in the future.

```txt

//default.conf

server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}

//syscom.conf

server {
  listen 80;
  server_name sysadmin.com;

  location / {
    proxy_pass http://192.168.1.30:8080;
  }
}

//sysorg.conf

server {
  listen 80;
  server_name sysadmin.org;

  location / {
    proxy_pass http://192.168.1.30:8081;
  }
}

```

Before running the nginx container, we must allow it on the port 80 ( after running the container return to port 1024):

```bash

//enable nginx accesso to port 80
> sudo sysctl net.ipv4.ip_unprivileged_port_start=80 

//return to default
> sudo sysctl net.ipv4.ip_unprivileged_port_start=1024

```

Running nginx:

```bash

> podman run --name=nginx -p 80:80 -v $HOME/podman-tutorial/nginx:/etc/nginx/conf.d:Z -d docker.io/library/nginx

```

Test the proxy:

```bash

curl http://sysadmin.org
<html>
  <header>
    <title>SysAdmin.org</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .org domain</p>
  </body>
</html>

> curl http://sysadmin.com
<html>
  <header>
    <title>SysAdmin.com</title>
  </header>
  <body>
    <p>This is the SysAdmin website hosted on the .com domain</p>
  </body>
</html>

```

FIN
