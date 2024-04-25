run locally with docker 

```
sudo systemctl start docker
docker run -p 4000:4000 -v $(pwd):/site bretfisher/jekyll-serve
```