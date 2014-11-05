---
layout: post
title: Kubernetes REST API
published: true
---

```json
{ 
    "kind": "PodList", 
    "creationTimestamp": null, 
    "selfLink": "/api/v1beta1/pods", 
    "resourceVersion": 7, 
    "apiVersion": "v1beta1", 
    "items": [ 
        { 
            "id": "dse-demo", 
            "creationTimestamp": "2014-11-05T09:34:20+08:00", 
            "resourceVersion": 6, 
            "namespace": "default", 
            "labels": { 
                "name": "dse-demo"
            }, 
            "desiredState": { 
                "manifest": { 
                    "version": "v1beta1", 
                    "id": "dse-demo", 
                    "uuid": "dd6f282d-648b-11e4-95ba-001f290cf88c", 
                    "volumes": null, 
                    "containers": [ 
                        { 
                            "name": "dse-demo", 
                            "image": "tegdsf/dse-demo", 
                            "ports": [ 
                                { 
                                    "hostPort": 19800, 
                                    "containerPort": 19800, 
                                    "protocol": "TCP"
                                }
                            ], 
                            "imagePullPolicy": "PullIfNotPresent"
                        }
                    ], 
                    "restartPolicy": { 
                        "always": { }
                    }
                }, 
                "status": "Running", 
                "host": "10.6.207.17"
            }, 
            "currentState": { 
                "manifest": { 
                    "version": "", 
                    "id": "", 
                    "volumes": null, 
                    "containers": null, 
                    "restartPolicy": { }
                }, 
                "status": "Running", 
                "host": "10.6.207.17", 
                "podIP": "192.168.1.230", 
                "info": { 
                    "dse-demo": { 
                        "state": { 
                            "running": { 
                                "startedAt": "2014-11-05T01:34:23.303504122Z"
                            }
                        }, 
                        "restartCount": 1, 
                        "image": "tegdsf/dse-demo"
                    }, 
                    "net": { 
                        "state": { 
                            "running": { 
                                "startedAt": "2014-11-05T01:34:22.18712419Z"
                            }
                        }, 
                        "restartCount": 1, 
                        "podIP": "192.168.1.230", 
                        "image": "kubernetes/pause:latest"
                    }
                }
            }
        }
    ]
}
```
