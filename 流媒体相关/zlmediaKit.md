# quickStart

官方文档已经写的很好了，我这里只是做记录，采用官方的docker镜像进行测试入门

1. 拉取镜像

   ```shell
   docker run -id 
   	-p 1935:1935 
   	-p 8080:80 
   	-p 8554:554 
   	-p 10000:10000 
   	-p 10000:10000/udp 
   	-p 8000:8000/udp 
   	zlmediakit/zlmediakit:Release.last
   
   ```

   

2. 推流测试

   1. 采用OBS、RTMP推流

      `rtmp://159.75.241.64/live/test` 

3. vlc拉流

   1. rtmp
      1. rtmp://159.75.241.64/live/test
   2. hls
      1. http://159.75.241.64:8080/live/test/hls.m3u8   注意这里的端口在上方的容器映射关系