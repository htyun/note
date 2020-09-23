# 新建局域网段
> docker network create  --subnet=10.27.0.0/16 mongo-net

# 使用openssl在当前目录生成key_file
openssl rand -base64 756 > key_file
chmod 600 key_file