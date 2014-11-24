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

###Start HTTP Server
    
    docker run -d -v /root/files:/root/files -p 8000:8000 tegdsf/httpserver
