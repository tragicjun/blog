---
layout: post
title: JFlynn Demo
published: true
---

###Build HTTP Server Image

    FROM centos
    VOLUMN /root/files
    WORKDIR /root/files
    EXPOSE 8000
    ENTRYPOINT python -m SimpleHTTPServer 8000

###Build&Start HTTP Server
    
    docker build -t tegdsf/httpserver .
    docker run -d -v /root/flynn/files:/root/files -p 8000:8000 tegdsf/httpserver

###Upload below files to HTTP server

    # ls /root/flynn/files/
    jdk/openjdk1.6.0_27.tar.gz
    maven-3.2.1.tar.gz
    
###Copy java-buildpack

    # ls /root/flynn/buildpacks/
    heroku-buildpack-java-master
    
###Start slugbuilder

    cat slug-java-example.tar | docker run -i -v /root/flynn/buildpacks:/tmp/buildpacks -e HTTP_SERVER_URL=http://localhost:8000 -a stdin -a stdout -a stderr flynn/slugbuilder
    
