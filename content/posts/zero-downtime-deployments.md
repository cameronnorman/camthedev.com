+++
title = "Zero downtime deployments with Docker and Nginx"
date = 2018-03-03T13:23:10+01:00
draft = false
tags = []
categories = []
+++
During my software development career I have had times where the infrastructure had services running that could not afford downtime. I managed to achieve zero downtime using Docker and Nginx. Here is how I did it.


Obviously this is achievable through many technologies like Kubernetes and Docker swarm but due to time restrictions I didn’t want to invest in understanding how these complex solutions work and therefore decided to use what I had.

## Hosting a web service using Docker and Nginx
Let's start with a simple scenario, I want to run a single web application in a docker container and make that web application accessible through nginx.

[Installing Nginx](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04)

[Installing Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04)

Once Docker and Nginx are installed lets try running the commands below.

#### Create a simple webpage
```html
<html>
  <head></head>
  <body>
    <h1>Simple website 1</h1>
  </body>
</html>
```
#### Create a Sinatra App to host the website using Ruby
```ruby
require 'sinatra'

get '/' do
  erb :index
end
```
#### Create your dockerfile
```Dockerfile
FROM ruby:alpine3.10

RUN gem install sinatra

COPY . .

CMD ruby server.rb -p 3000 -o 0.0.0.0
```
#### Build a docker image
```bash
docker build . -t simple_site_image:0.0.1
```
#### Startup a your container
```bash
docker run -d --name simple_website -p 3000:3000 simple_site_image:0.0.1
```
#### Create Nginx config file
```bash
touch /etc/nginx/conf.d/default.conf
```
#### Update Nginx config to point incoming request to docker container:
```bash
vim /etc/nginx/conf.d/default.conf
```

```bash
server {
  listen 443 ssl;
  listen [::]:443;
  server_name simplesite.com;

  ssl_certificate /etc/nginx/certs/your_special.pem;

  ssl_certificate_key /etc/nginx/certs/your_special_key.pem;

  client_max_body_size 0;
  location / {
    proxy_pass http://simple_website:3000;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
  }
}
```

#### Validate that it works
Once you have setup the above you should proceed to simplesite.com to confirm that your server has been setup correctly
![Screenshot-2020-04-29-at-09.11.50](https://camthedev.com/images/Screenshot-2020-04-29-at-09.11.50.png)

Yay it works!

## Using Nginx Upstream

So great! We understand on a very high level how to get traffic to our container through nginx. Now how do we deploy new changes to this service without having downtime? It starts at the Nginx config. Nginx has this cool feature called upstream. Upstream allows you to decide how traffic should be sent to one or more services.

Where could this feature be useful? Well what if we had two different versions of a Docker image running on two containers on our server, we could then send the traffic we receive from our domain to these two containers. What is even better is that we can decide what percentage of traffic we would like to do this with. So we could send only ten percent of traffic to the Docker container running the new Docker image if we wanted to. Cool!

Can you perhaps think of another use case for this feature? This is a great way to do AB testing for a website because then you are able to send portions of your traffic to two different services running the two things you would like to be AB Testing. Well thats cool but how can you show me some code now please…. Sure!

#### Changing Nginx config to have support for Upstream
```bash
upstream simple_website_upstream {
    server simple_website:3000       weight=1;
}

server {
  listen 443 ssl;
  listen [::]:443;
  server_name simple.yourdomain.com;

  ssl_certificate /etc/nginx/certs/your_special.pem;

  ssl_certificate_key /etc/nginx/certs/your_special_key.pem;

  client_max_body_size 0;
  location / {
    proxy_pass http://simple_website_upstream;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
  }
}
```

#### Reload Nginx
```bash
sudo service nginx reload
```


## Switching containers with no downtime

If you go to https://simplesite.com now you should still see your simple website. Technically nothing should have changed. Awesome! So now that we know how to use Nginx upstream. How do we achieve zero downtime using this feature? Lets start by making a change to our html

```html
<html>
  <head></head>
  <body>
    <h1>Simple website 2</h1>
  </body>
</html>
```

#### Start a container running the new image
```bash
docker run -d --name simple_website_2 -p 3000:3000 simple_site_image:v0.0.2
```

#### Update the Nginx upstream config to send all new traffic to this new service running the new image
```bash
upstream simple_website_upstream {
    server simple_website_1:3000       weight=1;
    server simple_website_2:3000       weight=1;
}

server {
  listen 443 ssl;
  listen [::]:443;
  server_name simple.yourdomain.com;

  ssl_certificate /etc/nginx/certs/your_special.pem;

  ssl_certificate_key /etc/nginx/certs/your_special_key.pem;

  client_max_body_size 0;
  location / {
    proxy_pass http://simple_website_upstream;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
  }
}
```

#### Reload nginx
```bash
sudo service nginx reload
```

You should see below while I am refreshing my browser that once I reload my nginx the traffic is now being sent between the two docker containers.
![Screen-Recording-2020-04-29-at-09.18.51-2](https://camthedev.com/content/images/2020/04/Screen-Recording-2020-04-29-at-09.18.51-2.gif)

Now I simply remove the first service from my nginx config because I am happy that the new docker image is running successfully
```bash
upstream simple_website_upstream {
    server simple_website_2:3000       weight=1;
}

server {
  listen 443 ssl;
  listen [::]:443;
  server_name simple.yourdomain.com;

  ssl_certificate /etc/nginx/certs/your_special.pem;

  ssl_certificate_key /etc/nginx/certs/your_special_key.pem;

  client_max_body_size 0;
  location / {
    proxy_pass http://simple_website_upstream;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
  }
}
```

I reload my nginx

```bash
sudo service nginx reload
```

At this stage my new docker image is running in production.What is nice about this approach is that you can also pick up potential bugs in the new image early if you send a smaller portion of traffic to the new service. To do this you would adjust the weights from the 1:1 that you see in the example above to somehting like 3:1.
```bash
server simple_website_1:3000       weight=1;
server simple_website_2:3000       weight=1;
```

```bash
server simple_website_1:3000       weight=3;
server simple_website_2:3000       weight=1;
```
This means that 1 in every 4 requests will be sent to the new server. Once you identify that there is something wrong with the new image all you need to do is update your Nginx upstream config to point all traffic to the old service and you can safely stop the old container.

Using this approach allowed me to achieve the zero downtime that I wanted but also didn’t require me to make massive investments into learning and implementing complex solutions. As much as I love making these changes in a console I decided that it would be easier to make these kinds of changes using an API. That's why I built coxswain(link to coxswain here). It's an open source project that allows you to do these cool things using rest
