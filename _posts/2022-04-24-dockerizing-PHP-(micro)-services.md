---
title: Dockerizing PHP (Micro-) Services
author: Oliver
date: 2022-04-23 16:22:00 +0010
categories: [DevOps]
tags: [php,symfony,docker]
---
When writing the backend for a web application, you might at one point come to the conclusion that you want 
to run dedicated (micro-)services for parts of the backend's functionality. 
For example you might have an API for CRUD operations in your online shop but handle billing through a separate
service. It would be possible to host these services on different servers with different domains or use subdomains. 
But for me the best solution was to serve different APIs at different paths on the same server and dockerize the 
services. They are then served by the same webserver under the same IP and domain, which makes using cookies easier
and does not require CORS configuration.  
Java or JavaScript were the languages of choice for me so far and here the web server is part of the application 
itself. Running the application will allow you to use a reverse proxy on your webserver to serve the application 
at the desired path. With Python the process is already a little more complicated, as you will have to run a separate 
WSGI or ASGI server that executes your code when it receives an HTTP request. But the way this server is 
executed makes it quite clear what happens because everything needs to be stated explicitly.
```
uvicorn --port 8080 --app-dir src main:app
```
This lets uvicorn listen for HTTP requests on port 8080 and executes what is to be found inside `app` in the `src/main.py` file.
PHP has a similar service, that allows executing PHP code in response to an HTTP request, named FPM (FastCGI Process Manager).
But FPM is way less explicit in what it will do after you run it. To start the process it is enough to run `php-fpm`. This leaves
a lot to your fantasy. What will be executed when? How do HTTP parameters and payload relate to the PHP arguments? 
Most of this is defined by the web server you are using. Nginx in my case. So I did what every reasonable person 
would do and copied code from a previous project without reading any documentation. 
```
  location ~ \.php$ {
    fastcgi_pass php_frontend;
    include fastcgi_params;
    fastcgi_index index.php;
    fastcgi_buffers 16 16k;
    fastcgi_buffer_size 32k;
    fastcgi_read_timeout 600;
  }
}
```
I replaced the regex to match my path, changed the php_frontend upstream to the name of my FPM container and rewrote the URL to cut 
out the part of the path that distinguished the different APIs (e.g. `/api/crud/v1.0/products` would be `/products` so the application 
doesn't have to be aware of how it is served on the internet). This gave me the following config: 
```
  location ~ api\/crud {
    rewrite \/api/crud\/[0-9]\.[0-9](.*) $1 break;
    fastcgi_pass php_api;
    include fastcgi_params;
    fastcgi_index index.php;
    fastcgi_buffers 16 16k;
    fastcgi_buffer_size 32k;
    fastcgi_read_timeout 600;
  }
}
```
Then I ran FPM in one container with the PHP files and nginx in another one. To nobody's surprise but my own it didn't work. 
My simple assumption that everything would just work for some reason was shattered.  
Did the webserver require access to the files to read them and execute them through FPM? Or did FPM just need the right instructions 
to pick the right file to execute? I had to resort to desperate measures and actually read the docs. 
The server does indeed not need to have access to the files. The `fastcgi_param SCRIPT_FILENAME` param defined in the 
included `fastcgi_params` lets FPM know which file to pick. 
I used Symfony so I only ever want to execute one file; the `index.php` file in the projects root. But the `SCRIPT_FILENAME` sets it tos
`$document_root$fastcgi_script_name`. The webserver's `$document_root` is not the same as the project root in the FPM container.
This already rules out any of my setup would work. So with this in mind and knowing I always ever want to execute a single file, I set 
`SCRIPT_FILENAME` to the hardcoded value of `fastcgi_param SCRIPT_FILENAME /var/www/public/index.php;` and mapped my Symfony projects root 
to `/var/www` as a docker volume. I held my breath and fired the request and it went through... With the wrong URI being requested from 
my application. Even though I rewrote the URI and the URI was logged correctly by Symfony, the 404 response returned by my app showed 
the original unaltered path (`/api/crud/v1.0/products` stayed `/api/crud/v1.0/products` instead of being just `/products`). 
So, some more docs 🥲  
The URI passed to FPM is defined by the `REQUEST_URI` parameter, again defined in the include. And this parameter is set to `$request_uri`. This 
is the request's original URI before rewriting. A small change, setting it to `$uri?$args` instead and finally everything works. 
My final configuration is this:
```
  location ~ \.php$ {
    rewrite \/api/crud\/[0-9]\.[0-9](.*) $1 break;
    fastcgi_pass php_api;
    include fastcgi_params;
    fastcgi_param REQUEST_URI $uri?$args;
    fastcgi_param SCRIPT_FILENAME /var/www/public/index.php;
    }
  }
```
I use it for local development only and wouldn't recommend using it in production without some optimization. But you do you 👌