# pavelvizir_platform
pavelvizir Platform repository  

[![Build Status](https://travis-ci.com/otus-kuber-2019-06/pavelvizir_platform.svg?branch=master)](https://travis-ci.com/otus-kuber-2019-06/pavelvizir_platform)

[Link to docker hub](https://hub.docker.com/r/pavelvizir)

# Table of contents:  
- [Homework-01 aka 'kubernetes-intro'](#homework-01-aka-kubernetes-intro)  

## Homework-01 aka 'kubernetes-intro'  
[.history-01](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-intro/.history-01)  
### Task \#1:  
#### How pods get recreated in kube-system

`kubectl delete pod --all -n kube-system`

> kube-apiserver = static pod, contolled by kubelet directly (/etc/kubernetes/manifests/kube-apiserver.yaml)  
> core-dns = normal pod in deployment, deployment controller, replicas > 0  

### Task \#2:  
#### Make everything in homework

[.history-01](https://github.com/otus-kuber-2019-06/pavelvizir_platform/blob/kubernetes-intro/.history-01)
