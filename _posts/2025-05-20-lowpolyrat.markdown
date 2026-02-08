---
layout: post
title:  "lowpolyrat.xyz"
date:   2025-05-20 14:16:48 -0500
categories: projects
---

A while ago I was partaking in one of my favorite passtimes of messing around with the Google Cloud Pricing Calculator, and I noticed an interesting product called Cloud Run. Google was promising that I could take any old container and `gcloud run deploy` it to the web. As someone with a bunch of silly side projects practically begging for their own domain name, I decided to try putting one of my silly projects on the web for everyone (mostly just web scrapers and bots in hindsight) to see. This was also a good chance to overcomplicate things further and buy a domain and create a full website for fun, something which I hadn't done before.

# The Idea

A website where you can customize and download a 3D model of a rat with a low polygon count. I want the static assets to be embedded in the binary.

# The implementation

### Tech Stack

The server is nothing unusual. Static files are stored using Go's `embed.FS`. There is one endpoint to generate and return the 3D model. The frontend renders the 3D model using three.js and vanilla JavaScript. Having only one build step for the entire application is nice.

### Building the rat

This was probably the most involved part of the project. I didn't want to just read a 3D model and apply some scaling, since the proportions could get out of whack. It seemed more flexible to compute the vertices of the model. I took inspiration from the [Loft](https://en.wikipedia.org/wiki/Loft_(3D)) operation in CAD programs. Building the model worked by defining a number of cross-sections at different locations, and wrapping faces around these cross sections.

First, the cross sections are generated.
![](/assets/images/loft1.png)

For each pair of cross sections, the loft is generated.
![](/assets/images/loft2.png)

This repeats for every pair of cross sections.
![](/assets/images/loft3.png)

This process leaves two holes in the model: one at the nose and one at the tip of the tail. The first and last cross-sections are turned into planes in the model to close these holes.

At this point, the model is just a list of planes. To render this model on the UI, it is first converted to a binary STL file. From [Wikipedia](https://en.wikipedia.org/wiki/STL_(file_format)#Binary), the STL format is as follows:
```
UINT8[80]    – Header                 - 80 bytes
UINT32       – Number of triangles    - 04 bytes
foreach triangle                      - 50 bytes
    REAL32[3] – Normal vector         - 12 bytes
    REAL32[3] – Vertex 1              - 12 bytes
    REAL32[3] – Vertex 2              - 12 bytes
    REAL32[3] – Vertex 3              - 12 bytes
    UINT16    – Attribute byte count  - 02 bytes
end
```

Since the header is ignored, this entire step consists of taking each plane and turning it into two triangles, then writing those triangles to the stl. Our API endpoint now receives a request with parameters, builds the custom model, then returns a STL file.

On the frontend, three.js has a `STLLoader` addon that can fetch and load from a URL. The query params are put into the url, and passed to the `STLLoader`
```
return new Promise((resolve, reject) => {
    url = `/api/rat?s=${size}&w=${width}&f=${flatness}`;
    const loader = new STLLoader();
    loader.load(url, (geometry) => {...});
});
```

# Deploying to Cloud Run

With my million dollar idea up and running locally, now I just need to deploy it. Creating a container was also very easy, basically just running go build and copying the binary to /usr/local/bin. I figured I would go full gcloud and push my image to Google Cloud Artifact Registry.
```
docker build . -t lowpoly-rat
docker tag lowpoly-rat:latest $GCLOUD_REGION-docker.pkg.dev/$GCLOUD_PROJECT/$GCLOUD_REPO/lowpoly-rat:latest
```

I found that the easiest way to deploy the app was to first do so using the Google Cloud web console. Once the Cloud Run instance is up and running, I downloaded the `service.yaml` from the web console. I replaced a couple of the values with bash variable statements, similar to the above `docker tag ...` statement.
```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: lowpoly-rat
  labels:
    cloud.googleapis.com/location: "$GCLOUD_REGION"
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      name: "lowpoly-rat-$REVISION"
      labels:
        run.googleapis.com/startupProbeType: Default
      annotations:
        autoscaling.knative.dev/maxScale: '1'
        run.googleapis.com/startup-cpu-boost: 'false'
    spec:
      containerConcurrency: 80
      timeoutSeconds: 1
      containers:
      - name: lowpoly-rat
        image: $GCLOUD_REGION-docker.pkg.dev/$GCLOUD_PROJECT/$GCLOUD_REPO/lowpoly-rat:latest
        ports:
        - name: http1
          containerPort: 80
        resources:
          limits:
            cpu: 1000m
            memory: 128Mi
        startupProbe:
          timeoutSeconds: 240
          periodSeconds: 240
          failureThreshold: 1
          tcpSocket:
            port: 80
```

Replacing these values with variables is a quick and dirty way of deploying a new revision without having to use the web console again. In a `.env` file I set in `$GCLOUD_REGION`, `$GCLOUD_PROJECT`, and `$GCLOUD_REPO`. Deploying the latest revision is as simple as build
```
# build and push
docker build . -t lowpoly-rat
docker tag lowpoly-rat:latest $GCLOUD_REGION-docker.pkg.dev/$GCLOUD_PROJECT/$GCLOUD_REPO/lowpoly-rat:latest
docker push $GCLOUD_REGION-docker.pkg.dev/$GCLOUD_PROJECT/$GCLOUD_REPO/lowpoly-rat:latest

# deploy
export REVISION=`docker image list --format "{{.ID}}" --filter reference=$GCLOUD_REGION-docker.pkg.dev/$GCLOUD_PROJECT/$GCLOUD_REPO/lowpoly-rat:latest`
envsubst < deploy/service.template.yaml > artifacts/service.yaml
gcloud run services replace artifacts/service.yaml
```

Putting this all into a Taskfile, deploying is as easy as running `task deploy`.

# Conclusion

It works! My first website with a real domain name is live.

![](/assets/images/rat.png)

Is it useful? No. Was it fun? Yes.
