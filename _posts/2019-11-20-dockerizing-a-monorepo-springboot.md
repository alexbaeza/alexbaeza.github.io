---
layout: post
title:  "Dockerizing Multiple Spring Boot Services in a Monolithic Repo"
date:   2019-11-20 21:00:00 +0000
---

Last night I was writing multiple spring boot services for a programming challenge;
I decided to complicate things a little bit and created these services as a monolithic repository 
and springboot nano-services where each service could only do one thing, because why not?

So in this post I'm going to show you how I built the docker images for all of the spring boot services and then used `docker-compose` to encapsulate the entire build and deployment process using a single file.

#### Example codebase
I will use a small example codebase to explain how to use Docker for a monolithic repository.
For this example I built 3 services:
 - app
 - data-retriever
 - classifier
 
 Structure is as follow:
```
.
├── README.md
├── app-service
│   └── src
│       ├── main
│       │   └── resources
│       └── test
│           └── resources
├── data-retriever-service
│   └── src
│       ├── main
│       │   └── resources
│       └── test
│           └── resources
└── classifier-service
    └── src
        ├── main
        │   └── resources
        └── test
            └── resources
```

### 1. Writing the `Dockerfile`

Firstly we will need to write a Dockerfile for each of the spring boot nano-services.
they will all be very similar so let's write the first Dockerfile for the `app-service`:
```dockerfile
FROM gradle:jdk8-alpine as builder
COPY --chown=gradle:gradle . /home/gradle/src
WORKDIR /home/gradle/src
RUN gradle build

FROM openjdk:8-jdk-alpine
EXPOSE 8091
COPY --from=builder /home/gradle/src/build/libs/app-service-0.0.1-SNAPSHOT.jar /app/app-service.jar
WORKDIR /app
CMD ["java","-XX:+UseG1GC", "-jar","app-service.jar"]

```
I'll explain line by line this is a multi-stage Dockerfile that does two things:
 - builds and packs the fat jar
 - runs the jar

#### First Stage - The build (line by line)

`FROM gradle:jdk8 as builder`

The keyword _FROM_ tells Docker to use a given base image as a build base. We have used 'gradle' with tag 'jdk8-alpine'. Think of a tag as a version.
The image is labelled as `builder` and it will be is used to run gradlew and build the fat jar,

`COPY --chown=gradle:gradle . /home/gradle/src`

_COPY_ instruction tells Docker to copy files from the local file-system to a specific folder inside the build image.

`WORKDIR /home/gradle/src`

_WORKDIR_ instruction sets the working directory

`RUN gradle build`

_RUN_ instruction tells Docker to execute a shell command-line within the target system. Here we executed a command to run gradle and build us the `.jar` file  

#### Second Stage - The run (line by line)

`FROM openjdk:8-jdk-alpine`

We select a new image. We have used 'openjdk' with tag '8-jdk-alpine'.

`EXPOSE 8090`

The _EXPOSE_ instruction informs Docker that the container listens on the specified network ports at runtime. Here we set our port to `8090`

`COPY --from=builder /home/gradle/src/build/libs/app-service-0.0.1-SNAPSHOT.jar /app/app-service.jar`

We copy the jar previously built as part of the "build image" to the new "run image".

`WORKDIR /app`

sets the working directory

`CMD ["java","-XX:+UseG1GC", "-jar","app-service.jar"]`

_CMD_ instruction sets the command to be executed when running the image. Here we will run the fat .jar using G1GC garbage collector:

Note: There can only be one CMD instruction in a Dockerfile. If you list more than one CMD then only the last CMD will take effect.

#### Voila! 
Now that we have our first Dockerfile we will need to create 2 more (1 for each of the remaining services)
To do this just copy and paste the same docker file and replace `app-service` with the target service (`data-retriever` or `classifier-service`) and change the port to your desired port
and place it on the root of each service.

You'll should end up with the following structure:
```
.
├── README.md
├── app-service
│   ├── Dockerfile
│   └── src
├── data-retriever-service
│   ├── Dockerfile
│   └── src
└── classifier-service
    ├── Dockerfile
    └── src
```

### 2. Writing the `docker-compose.yml`
We are going to use `docker-compose` to encapsulate the entire build and deployment process in a single file. 
This allows us to build and deploy the images with a single invocation of `docker-compose up`.

```yaml
version: "3"

services:
  app-service:
    build:
      context: app-service
      dockerfile: ./Dockerfile
    ports:
      - 8090:8080
    networks:
      - challenge-network
    depends_on:
      - data-retriever-service
      - classifier-service
  data-retriever-service:
    build:
      context: data-retriever-service
      dockerfile: ./Dockerfile
    ports:
      - 8091:8080
    networks:
      - challenge-network
  classifier-service:
    build:
      context: classifier-service
      dockerfile: ./Dockerfile
    ports:
      - 8092:8080
    networks:
      - challenge-network
networks:
  challenge-network:
    driver: bridge
```

This `docker-compose.yml` defines the services that we need to run together in an isolated environment.
Also creates a network called `challenge-network` and we will run our containers in the network so they can immediately communicate with other containers in the network.
We will place the `docker-compose.yml` at the root of our project. so your final structure will looks like this:
```
.
├── README.md
├── docker-compose.yml
├── app-service
│   ├── Dockerfile
│   └── src
├── data-retriever-service
│   ├── Dockerfile
│   └── src
└── classifier-service
    ├── Dockerfile
    └── src
```

### 3. Running it
 
Now we can simply run `docker-compose up` it will create our 3 docker containers and start them up.
and they will be available on the ports that we defined in our `.yml` previously.
