# Lab 7: Troubleshooting, what is a Pod?

This Lab helps you to troubleshoot your application and shows you some tools to make troubleshooting easier.

## Login to a Container

Running container should be treated as immutable infrastructure and should therefore not be modified. Although, there are some use-cases in which you have to login into your running container. Debugging and analyze is one example for this.


## Task: LAB7.1 Shell into POD


With Kubernetes you can open a remote Shell into a pod without installing SSH by Using the command `kubectl exec`. The command is used to executed anything in a pod. With the parameter `-it` you can leave open an connection. We can use `winpty` for this.

Choose a pod with `kubectl get pods --namespace [USER]` and execute the following command:

```bash
$ kubectl exec -it [POD] --namespace [USER] -- /bin/bash
```

With this, you can work inside the pod, e.g.:

```bash
bash-4.2$ ls -la
total 40
drwxr-xr-x 1 default root 4096 Jun 21  2016 .
drwxr-xr-x 1 default root 4096 Jun 21  2016 ..
drwxr-xr-x 6 default root 4096 Jun 21  2016 .gradle
drwxr-xr-x 3 default root 4096 Jun 21  2016 .pki
drwxr-xr-x 9 default root 4096 Jun 21  2016 build
-rw-r--r-- 1 root    root 1145 Jun 21  2016 build.gradle
drwxr-xr-x 3 root    root 4096 Jun 21  2016 gradle
-rwxr-xr-x 1 root    root 4971 Jun 21  2016 gradlew
drwxr-xr-x 4 root    root 4096 Jun 21  2016 src
```

With `exit` you can leave the pod and close the connection

```bash
bash-4.2$ exit
```

## Task: LAB7.2 Single Command

Single commands inside a container can be executed with `kubectl exec`:


```bash
$ kubectl exec [POD] --namespace [USER] env
```

```bash
$ kubectl exec example-spring-boot-69b658f647-xnm94 --namespace [USER] env
PATH=/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=example-spring-boot-4-8mbwe
KUBERNETES_SERVICE_PORT_DNS_TCP=53
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=172.30.0.1
KUBERNETES_PORT_53_UDP_PROTO=udp
KUBERNETES_PORT_53_TCP=tcp://172.30.0.1:53
...
```

## Watch Logfiles

Logfiles of a pod can with shown with the following command:


```bash
$ kubectl logs [POD] --namespace [USER]
```

The parameter `-f` allows you to follow the logfile (same as `tail -f`). With this, logfiles are streamed and new entries are shown directly

When a pod is in State **CrashLoopBackOff** it means, that even after some restarts, the pod could not be started successfully. Even if the pod is not running, logfiles can be viewed with the following command:


 ```bash
$ kubectl logs -p [POD] --namespace [USER]
```


## Task: LAB7.3 Port Forwarding

Kubernetes allows you to forward arbitrary ports to your development workstation. This allows you to access admin consoles, databases etc, even when they are not exposed externaly. Port forwarding are handled by the Kubernetes master and therefore tunneled from the client via HTTPS. This allows you to access the Kubernetes platform even when there are restrictive firewalls and/or proxies between your workstation and Kubernetes.

Exercise: Access the Spring Boot Metrics from [Lab 4](04_deploy_dockerimage.md).


```bash
$ kubectl get pod --namespace [USER]
$ kubectl port-forward example-spring-boot-1-xj1df 9000:9000 --namespace [USER]
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
```

Don't forget to change the pod Name to your own Installation. If configured, you can use Auto-Completion.

The metrics are now available with the following Link: [http://localhost:9000/metrics/](http://localhost:9000/metrics/).
Those metrics are shown in JSON. With the same concept you can access databases from your local client or connect your local development environment via remote debugging to your application in the pod.
 
With the following link you find more information about port forwarding: <https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/>

**Note:** The `kubectl port-forward`-process runs as long as it is not terminated by the user. So when done, stop it with CTRL-C.

---

**End Lab 7**

<p width="100px" align="right"><a href="08_database.md">deploy datebases and bind to it →</a></p>

[← back to Overview](../README.md)