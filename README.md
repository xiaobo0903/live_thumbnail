Use nginx-rtmp create livestream thumbnail;

1、安装 Nginx-rtmp moudle：

   下载 nginx-rtmp source code: 
       git clone https://github.com/arut/nginx-rtmp-module.git;
   下载 nginx source Code: 
       wget http://nginx.org/download/nginx-1.9.3.tar.gz
   编译和安装完成；

2、配置nginx-rtmp的收录模块：
   直播截图就是利用收录模快来进行处理；
   首先启动收录模块的keyframes功能，并通过exec_record_down的消息处理截图事件，在nginx.conf中配置如下：
   
   rtmp {
    server {
        listen 1935;

        application myapp {
            live on;
            record keyframes;
            record_path /home/thumbnail;   #设置保存路径
            record_max_frames 5;           #文件中保存的最大frame数为5帧；
            record_interval 30s;           #每次收录文件生成的文件间隔为30秒；
            record_suffix _myapp.flv;      #生成的文件名后缀；
            exec_record_done ffmpeg -y -i $path -vf select='eq(pict_type\,I)' -vsync 2 -f image2 $dirname/$basename.jpg;  #每次收录完成后截收图片；$basename是依据record_suffix项内容生成的
        }
      }
   }
   
   
3、通过nginx的缓存来提供截图的访问：

    http {
       
       proxy_cache_path /tmp/cache levels=1:2 keys_zone=thumbnail_cache:10m max_size=4g inactive=60m use_temp_path=off;  #nginx 设置缓存的配置

       server {

           listen   80;          
       
           location /thumbnail {          #这样配置可通过rtmp://domain.com/appcode/channel来生成截图的访地址； http://domain.com/thumbnail/$basename.jpg

			            proxy_cache thumbnail_cache;
			            proxy_cache_methods GET HEAD POST;
			            proxy_cache_min_uses 1;
			            proxy_cache_valid 200 302 304 60s;   #对于 200、302、304的返回码都要做缓存处理，有效期为60秒；
			            proxy_cache_valid 404 60s;
			            proxy_cache_key "$host:$server_port$uri$is_args$args";
			
			            proxy_redirect off;
			            proxy_set_header Host $host;
			            proxy_set_header X-Real-IP $remote_addr;
			            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			            add_header  Nginx-Cache "$upstream_cache_status";  #增加此项可以查看命中效果
			            proxy_pass http://127.0.0.1:11080/;                #跳转服务,注间后面带"/"和不带的区别；
        		} 
        }
        
        server {

            listen 11080;

            location / {

                  root /home/thumbnail;
            }
        }            
      }
      
配置和处理完成！

根据onsite的直播架构，实际环境中使用要进行内外地址的转换，才能够该问到最终的截图资源，所以需要在nginx中增加对于内外地址路由表的查询处理；
