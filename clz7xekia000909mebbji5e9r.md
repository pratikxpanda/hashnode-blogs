---
title: "Securely Deploying React Apps on Kubernetes with Non-Root NGINX Containers"
datePublished: 2024-07-30T04:36:38.098Z
cuid: clz7xekia000909mebbji5e9r
slug: securely-deploying-react-apps-on-kubernetes-with-non-root-nginx-containers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1722256052249/52ee9030-4f95-4db2-ad08-7aa556a9f525.png
tags: nginx, kubernetes, reactjs, devops

---

In this article, we will explore how to securely deploy your React front-end applications on Kubernetes by serving them from a non-root NGINX container. This approach enhances security by minimizing the risk of privilege escalation attacks.

# Non Root Dockerfile

As a first step, ensure that your React app is served from a non-root container. To learn how to create a container where the React app is served by NGINX, please refer to my previous article: [Enhancing React App Security: Serving with Non-Root NGINX Containers (](https://pratikpanda.hashnode.dev/enhancing-react-app-security-serving-with-non-root-nginx-containers)[hashnode.dev](http://hashnode.dev)[)](https://pratikpanda.hashnode.dev/enhancing-react-app-security-serving-with-non-root-nginx-containers)

# [runAsNonRoot](https://pratikpanda.hashnode.dev/enhancing-react-app-security-serving-with-non-root-nginx-containers)

[In this section, we will discuss the SecurityContext section of a Ku](https://pratikpanda.hashnode.dev/enhancing-react-app-security-serving-with-non-root-nginx-containers)bernetes manifest. It contains security settings that Kubernetes applies.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: my-app
      image: myregistry.azurecr.io/my-app:latest
      ports:
        - containerPort: 8080
```

This `securityContext` object ensures that the container in the pod runs with a non-root user.

`runAsNonRoot` checks that the user (via UID) is a non-root user (&gt; 0); otherwise, pod creation will fail. Kubernetes only reads container image metadata for this check.

# runAsUser

`runAsUser` is a related setting that specifies the user ID to run the container. It should be used if the `USER` directive in the container image is unset, set by name instead of UID, or needs to be changed for any specific reason.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
  containers:
    - name: my-app
      image: myregistry.azurecr.io/my-app:latest
      ports:
        - containerPort: 8080
```

Use 101 for the value of runAsUser because it is the user ID of the NGINX user. To learn how to find this value, please check the UID in the NGINX unprivileged base image.: [Image Layer Details - nginxinc/nginx-unprivileged:1.25 | Docker Hub](https://hub.docker.com/layers/nginxinc/nginx-unprivileged/1.25/images/sha256-a79c2d0ab503f161b43b42cda70d752f037ec48dd811d175b31c8d4efc34f35f?context=explore)

# Summary

By making the two configuration changes (runAsNonRoot and runAsUser) in your Kubernetes manifest as shown above, you can switch to non-root hosting for your React apps in Kubernetes. Your app will be more secure and resilient to attacks. This approach also aligns your app with Kubernetes Pod hardening best practices.