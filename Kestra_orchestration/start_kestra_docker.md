# Using Docker to start kestra

```
docker run --pull=always --rm -it -p 8080:8080 --user=root \
  --name kestra \
  -v kestra_data:/app/storage \
  -v kestra_db:/app/data \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp:/tmp \
  kestra/kestra:latest server local
```
After install when you to stop or start kestra use  this command

```
docker start kestra
docker stop kestra

```

Url to connect : http://localhost:8080