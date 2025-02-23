+++
title = "Minimising Docker Images"
date = "2025-01-26T09:56:21Z"
author = "Samson"
cover = ""
coverCaption = ""
tags = []
keywords = []
description = ""
showFullContent = false
readingTime = true
hideComments = false
Toc = true
draft = true
+++

> [!INFO] What is Docker?
> See [Docker for Beginners by Prakhar Srivastav](https://docker-curriculum.com/)

## Background

At work no-one seems much concerned with having small docker images. Most of our images are ~250MB, some are ~700MB, few are >1GB. 

When I'm at home with my 1Gbps connection, I don't much care, but when I go an work somewhere else and the internet is slow, it's a real pain. What takes a matter of seconds at home takes minutes elsewhere. Plus at work I have to connect to [ECR](https://aws.amazon.com/ecr/) over a VPN where the endpoint is in the US. So my slow traffic goes from UK/Europe to the US to an EU datacenter back to me. A long way at 5Mbps. 

| _Download times_ | 5 Mbps  | 1 Gbps |
| ------------- | ------- | ------ |
| **250 MB**    | 6m 40s  | 2s     |
| **700 MB**    | 18m 40s | 5.6s   |
| **1.5 GB**    | 40m     | 12s    |

If I take this [FastAPI Starter Kit](https://github.com/MahmudJewel/fastapi-starter-boilerplate), and follow the instructions in the [README](https://github.com/MahmudJewel/fastapi-starter-boilerplate?tab=readme-ov-file#setup) to run it, I get this Dockerfile:

```dockerfile
FROM python:3.11-slim

COPY fastapi-starter-boilerplate /app

WORKDIR /app

RUN pip install -r requirements.txt

RUN alembic upgrade head

ENTRYPOINT ["uvicorn", "app.main:app"]

CMD ["--reload"]
```

And we can find the size of this image by building it an inspecting it. 

```bash
❯ docker build -f Dockerfile.v1 . -t fastapi-starter:v1
❯ docker inspect fastapi-starter:instructions -f '{{ .Size }}' | xargs echo '0.000001 *' | bc
279.321653
```

{{< details summary="_We can wrap this up in a little script..._" >}}

```shell
#!/usr/bin/env bash

dockerfile="Dockerfile.${1}"

if [ -f ${dockerfile} ]; then
  tag="fastapi-starter:${1}"
  docker build -f ${dockerfile} -t ${tag} . 2> /dev/null
else
  tag=${1}
  docker pull ${tag} > /dev/null
fi

docker inspect ${tag} -f '{{ .Size }}' | xargs echo '0.000001 *' | bc
```

Then we can use it to get the size of either the dockerfiles we make, or existing images. 

```shell
❯ ./get-size.sh v1
279.321653
```

{{< /details >}}

279 MB!

But the size of the starter kit is much smaller:

```shell
❯ du -h fastapi-starter-boilerplate | tail -1
144K	fastapi-starter-boilerplate
```

**So how can we get the size of the image to be closer to 144 KB than 279 MB?**


## Iterating on size...

### Choose the base image

In the Dockerfile above we choose the base image as `python:3.11-slim`. This `slim` refers to "Debian Slim Bookworm". We can see the size of this by inspecting the size directly. 

```shell
❯ ./get-size.sh python:3.11-slim
155.400060
```

That's a little over half the size. What if we use a different base image?

Fortunately there is a probject to have a small base image called "Alpine Linux". And we can get it just by replacing `slim` with `alpine`. We'll call this docker image `v2`.

```shell
❯ ./get-size.sh python:3.11-alpine
58.054110

❯ ./get-size.sh v2
182.761998

# still 124 MB of extra stuff
❯ echo "$(./get-size.sh v2) - $(./get-size.sh python:3.11-alpine)" | bc
124.707888
```






