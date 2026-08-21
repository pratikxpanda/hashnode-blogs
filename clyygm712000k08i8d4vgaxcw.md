---
title: "Enhancing React App Security: Serving with Non-Root NGINX Containers"
datePublished: 2024-07-23T13:36:44.822Z
cuid: clyygm712000k08i8d4vgaxcw
slug: enhancing-react-app-security-serving-with-non-root-nginx-containers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721739736522/0ca0ad3f-6674-486a-bf2e-4188f1bf8fca.png
tags: docker, nginx, reactjs, devops, containerization, websecurity

---

This post demonstrates how to enhance the security of your React front-end applications by serving them from a non-root NGINX container. NGINX, an open-source web server, is widely used for serving web applications and can be installed locally or as a Docker container.

# Creating a React + NGINX container

## Step-by-Step Guide to Containerizing Your React App with NGINX

Here is a Dockerfile you can use to containerize a React application and serve it with a NGINX container:

```dockerfile
# Use a specific version of node as the base image
FROM node:20.12.0-alpine as build

# Set the working directory to /app
WORKDIR /app

# Copy package.json to the working directory
COPY package.json /app/package.json

# Install npm packages quietly
RUN npm install --quiet

# Copy the rest of the application to the working directory
COPY . /app

# Build the application
RUN npm run build

# Use the nginx image version 1.25 as the base image
FROM nginx:1.25-alpine

# Copy the nginx configuration file to the appropriate location
COPY nginx.conf /etc/nginx/nginx.conf

# Copy the built application from the first build stage to the nginx directory
COPY --from=build /app/build /var/www

# Expose port 80 for the application
EXPOSE 80
```

To create the Docker image from this Dockerfile, use the following command:

```bash
docker build -t my-app -f Dockerfile . 
```

To run the Docker container based on the image created above, use the following command:

```bash
docker run -d --name my-app -p 7241:80 my-app
```

In the above command, we are exposing the app on port 80, which is mapped to port 7241 on our local machine.

Now, try accessing the application using [http://localhost:7241](http://localhost:7241/), and you will see the page below:

![React App Running](https://media.licdn.com/dms/image/D5612AQFCyVU59p3Qbg/article-inline_image-shrink_400_744/0/1714392144586?e=1727308800&v=beta&t=IryKblMnHxJgyoSejo5ix5_ftJSvpWVHu4MuYVh1vHQ align="center")

Now that the application is running successfully and can be accessed, let's find out which user is running the app inside the container.

Run the following command on the running container:

```bash
docker exec -it my-app /bin/sh
```

The above command will open a shell. In the shell, run the following command (as shown in the screenshot):

```bash
# whoami
root
```

As you can see, the application is running as the root user in the container. Now, let's understand why this is a security risk.

# Root vs Non root container

## Understanding the Security Risks: Root vs Non-Root Containers

If you run your app as root, the app process can do anything in the container, such as modify files, install packages, or run arbitrary executables. This is a concern if your app is ever attacked. Hosting containers as non-root aligns with the principle of least privilege. It is free security provided by the operating system. If you run your app as non-root, the app process cannot do much, greatly limiting what a bad actor could accomplish.

# Securing the app using non root React + NGINX container

## Enhancing Security: Serving Your React App with a Non-Root NGINX Container

Now that you understand the difference between root and non-root containers, let's modify the Dockerfile to switch to a non-root container.

There isn't much to change. The main change is switching from **nginx** to **nginxinc/nginx-unprivileged** in the Dockerfile.

Here is a Dockerfile that can be used to containerize a React app and serve it using a NGINX container:

```dockerfile
# Use a specific version of node as the base image
FROM node:20.12.0-alpine as build

# Set the working directory to /app
WORKDIR /app

# Copy package.json to the working directory
COPY package.json /app/package.json

# Install npm packages quietly
RUN npm install --quiet

# Copy the rest of the application to the working directory
COPY . /app

# Build the application
RUN npm run build

# Use the nginxinc/nginx-unprivileged image version 1.25-alpine as the base image for the second build stage
FROM nginxinc/nginx-unprivileged:1.25-alpine

# Copy the nginx configuration file to the appropriate location
COPY nginx.conf /etc/nginx/nginx.conf

# Copy the built application from the first build stage to the nginx directory
COPY --from=build /app/build /var/www

# Expose port 8080 for the application
EXPOSE 8080
```

In addition to the above, please make sure you include the following line in your nginx.conf.

```nginx
pid        /tmp/nginx.pid;
```

This is a well-known configuration requirement for the nginxinc/nginx-unprivileged image.

Also, ensure that NGINX is listening on port 8080, as port 80 cannot be used by a non-root user since it is a privileged port. Check for the following:

```nginx
listen 8080;
```

To create the Docker image from this Dockerfile, use the following command:

```bash
docker run -d --name my-app-non-root -p 7242:8080 my-app-non-root
```

In the above command, we are exposing the app on port 8080, which is mapped to port 7242 on our local machine.

Now, try accessing the application using [http://localhost:7242](http://localhost:7242/), and you should see the page below:

![React App Running](https://media.licdn.com/dms/image/D5612AQFCyVU59p3Qbg/article-inline_image-shrink_400_744/0/1714392144586?e=1727308800&v=beta&t=IryKblMnHxJgyoSejo5ix5_ftJSvpWVHu4MuYVh1vHQ align="center")

Now that the application is running successfully and can be accessed, let's find out which user the app is running as inside the container.

Run the following command on the running container:

```bash
# whoami
nginx
```

As you can see, the application is running as a non-root user (nginx), adding an extra layer of security.

By following the steps outlined in this post, you can significantly enhance the security of your React front-end applications by serving them from a non-root NGINX container. This approach aligns with the principle of least privilege, reducing the potential impact of any security vulnerabilities. By using the `nginxinc/nginx-unprivileged` image and configuring NGINX to run on a non-privileged port, you ensure that your application runs with minimal permissions, thereby mitigating risks associated with running processes as the root user. This simple yet effective change can provide an additional layer of security for your web applications.