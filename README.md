# Proof of concept: Running Vue.js on Google Cloud Run

## Description

I wanted to test running [Vue.js](https://vuejs.org) on Cloud Run and run some benchmarks.

[![Run on Google Cloud](https://deploy.cloud.run/button.svg)](https://deploy.cloud.run)

## Creation of the project

```bash
$ npm i -g @vue/cli
$ vue create cloud-run
```

Vue CLI creates a Single Page Application type of project, so I needed to install `serve` to serve the project after the build. 

To install `serve`, run:

```bash
$ npm install --save-dev serve
```

I had to update the `package.json` file with the "start" script, like so: 

```javascript
"scripts": {
    ...,
    "start": "serve -p $PORT dist/",
}
```

## Creating the Dockerfile

Now I can build the app and run the `start` command.

```dockerfile
# Use the official lightweight Node.js 12 image.
# https://hub.docker.com/_/node
FROM node:12-alpine

ENV PORT=8080

# Create and change to the app directory.
WORKDIR /usr/src/app

RUN set -ex && \
    adduser node root && \
    chmod g+w /app && \
    apk add --update --no-cache \
      g++ make python \
      openjdk8-jre

# Copy application dependency manifests to the container image.
# A wildcard is used to ensure both package.json AND package-lock.json are copied.
# Copying this separately prevents re-running npm install on every code change.
COPY package*.json ./

# Install production dependencies.
RUN npm ci

# Copy local code to the container image.
COPY . ./

RUN npm run build

# Run the web service on container startup.
CMD [ "npm", "run", "start" ]

```

## Deployment

I used Cloud Build to build the docker image

```bash
$ gcloud builds submit --tag gcr.io/${PROJECT_ID}/helloworld
```

Then deployed it:

```bash
$ gcloud run deploy --image gcr.io/${PROJECT_ID}/helloworld --platform managed
```

## Benchmark

Ran a small benchmark (to avoid crazy costs) with Apache Benchmark:

```bash
$ ab -n 1000 -c 80 https://cloud-run-url/
```

Here are the results: 

```
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking helloworld-2fjcn3qpbq-uw.a.run.app (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        Google
Server Hostname:        helloworld-2fjcn3qpbq-uw.a.run.app
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-CHACHA20-POLY1305,2048,256
Server Temp Key:        ECDH X25519 253 bits
TLS Server Name:        helloworld-2fjcn3qpbq-uw.a.run.app

Document Path:          /
Document Length:        746 bytes

Concurrency Level:      80
Time taken for tests:   8.671 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      1272004 bytes
HTML transferred:       746000 bytes
Requests per second:    115.33 [#/sec] (mean)
Time per request:       693.656 [ms] (mean)
Time per request:       8.671 [ms] (mean, across all concurrent requests)
Transfer rate:          143.26 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       49  391 156.9    370     888
Processing:    32  252 106.9    285     878
Waiting:       28  198 101.0    185     876
Total:        313  643 204.8    620    1398

Percentage of the requests served within a certain time (ms)
  50%    620
  66%    703
  75%    734
  80%    790
  90%    898
  95%   1054
  98%   1136
  99%   1194
 100%   1398 (longest request)
```

## Conclusion

Really easy to deploy VueJS on Cloud Run and the performances aren't bad, even with a cold start.
