## nginx 400 error (prestashop)
可以先刪除cookie看看

可能request header過大
也許是prestashop某些動作重複寫入cookie

可以先用無痕視窗試試看是否正常

參考: https://blog.csdn.net/zhuyiquan/article/details/78707577


## virtual url
```
location /group1-shop1/ {
    rewrite ^/group1-shop1/(.*)$ /$1;
}
```


## error.log
錯誤不會寫到指定的error.log path 與 access.log path
```
rm -f /var/log/nginx/*
nginx -s reload
```
可參閱:https://stackoverflow.com/questions/32299582/nginx-access-log-and-error-log-file-empty  

開啟 rewrite_log
```
error_log  /var/log/nginx/error.log notice; # rewrite_log 僅在notice level有效
rewrite_log on;
```
參閱:https://stackoverflow.com/questions/9900443/where-does-nginx-store-the-rewrite-log



## location debug
除錯可以將 `error_log`寫在 location裡，用以確定使否有跑進此location
```
location /moon2/ {
    error_log /etc/nginx/sites-enabled/moon2.log debug;
    rewrite_log on; 
    rewrite ^/moon2/(.*)$ /$1 last;
}
```
若用debug不用加`rewrite_log on`也可以印出 rewritten
用notice或info則必須加 `rewrite_log on` 才會出現 rewritten


### priority
一次狀況不管怎麼改都無動於衷，後來是利用上面的 error_log 才發現吃到別的 location  
location會依照以下的符號決定優先順序
1. `=` (exactly)
2. `^~`
3. `~` (regular expression case insensitive)
4. `~*` (regular expression case insensitive)
5. `/`

參考: https://stackoverflow.com/questions/5238377/nginx-location-priority

## localhost 403 forbidden
`try_files $uri $uri/ /index.php?$args;`
這段的意思是在根目錄找檔案$uri，若找不到則找尋目錄，nginx訪問目錄會列出目錄中的所有檔案,
但預設目錄的索引是笨禁止的，所以回傳403 forbidden
可以拿掉`$uri/` 或者 開啟索引目錄功能
```
location / {
    try_files $uri /index.php?$args;
}
```
or
```
location / {
    autoindex on;
    try_files $uri $uri/ /index.php?$args;
}
```

參考: https://stackoverflow.com/questions/19285355/nginx-403-error-directory-index-of-folder-is-forbidden


## localhost/index.php 500 Internal Server Error
搜尋過後發現是多了一行導致
```
location ~ [^/]\.php(/|$) {
    try_files $fastcgi_script_name /index.php$uri&$args;
}
```
官網有類似這一行，訪問localhost/index.php會導致其不會送到fpm而在此location無限循環，故拿掉

$fastcgi_script_name: 
> This variable is equal to the URI request or, if if the URI concludes with a forward slash, then the URI request plus the 
> name of the index file given by fastcgi_index. It is possible to use this variable in place of both SCRIPT_FILENAME and 
> PATH_TRANSLATED, utilized, in particular, for determining the name of the script in PHP
$fastcgi_script_name => 指request的路徑或者 request的路徑加上 `index.php`
ex:
訪問 `http://localhost:8080/admin214zaerzf/index.php?controller=AdminOrders&token=2fdcdfe48c44491c244c65ec1ef05665`
=> $fastcgi_script_name 為 /admin214zaerzf/index.php

從官網可看出 `$fastcgi_script_name`  
一般搭配 fastcgi_split_path_info, $fastcgi_path_info 使用
```
location ~ ^(.+\.php)(.*)$ {
    fastcgi_split_path_info       ^(.+\.php)(.*)$;
    fastcgi_param SCRIPT_FILENAME /path/to/php$fastcgi_script_name;
    fastcgi_param PATH_INFO       $fastcgi_path_info;
```
官網範例:
當request為 `/show.php/article/0001` 時 
SCRIPT_FILENAME = /path/to/php/show.php
PATH_INFO       = /article/0001
