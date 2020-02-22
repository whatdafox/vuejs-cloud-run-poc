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

```

## Conclusion

Really easy to deploy VueJS on Cloud Run.
