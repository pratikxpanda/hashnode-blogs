---
title: "Streamlining Container Image Promotion Across Environments: Two Effective Strategies"
datePublished: 2024-07-24T10:18:42.202Z
cuid: clyzozd0a001p08l4auo523kl
slug: streamlining-container-image-promotion-across-environments-two-effective-strategies
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721747824700/331e7cf9-1106-41c4-bc12-8c6d406a2312.jpeg
tags: kubernetes, devops, containerization, ci-cd

---

When managing containerized applications, a crucial aspect to consider is the promotion of container images across various environments.

In this article, I will discuss two strategies to achieve this. It's important to note that I will only cover these approaches at a high level. I won't go into the implementation details or syntax of any DevOps platform. However, both strategies can be implemented using any modern DevOps platform.

# Strategies

## Two Proven Approaches for Container Image Promotion

Here are the two container image promotion strategies you can use:

* Using an additional environment-neutral container registry (called **base container registry**).
    
* Using build artifacts
    

# Using an additional environment neutral container registry

## Simplify Image Management with a Base Container Registry

In a system with containerized components, we typically have container registries for each environment. In this approach, besides having registries for each environment, we will also have an additional environment-neutral container registry. This registry is called the base container registry. By environment neutral, I mean the registry should be accessible to all environments and should not belong to any specific environment.

During the build process, we will containerize the application and publish the container image to the base container registry.

Then, during the deployment process for each environment, we will pull the container image from the base container registry, retag it for the target environment-specific container registry, and publish it there. These deployment steps will need to be repeated for each environment.

Here is how the stages in the DevOps pipeline would look using this approach:

![](https://media.licdn.com/dms/image/D5612AQHd4azvAaamjA/article-inline_image-shrink_1500_2232/0/1716009932556?e=1727308800&v=beta&t=fjxvbZr2l3uqzYkKmzAiUcnPpG7OSUST7UiPzW9TKFQ align="center")

This process is widely adopted by many organizations due to its simplicity and ease of use. However, there are some downsides to this approach:

1. The additional container registry adds extra cost and maintenance overhead.
    
2. Some organizations require completely isolated environments and do not align with the idea of an additional environment-neutral container registry.
    
3. Since the environment-neutral container registry is not part of any specific environment, there might be security gaps. A malicious attacker could gain access to the registry and tamper with the published images, which would then spread to other environments.
    

This is where the other approach becomes useful.

# Using a build artifact

## Secure and Cost-Effective Image Promotion with Build Artifacts

A build artifact is the package created during the build stage. It contains items useful for testing or deployment. Most modern DevOps platforms support jobs that let you publish build artifacts. These artifacts are managed by the DevOps platform in a secure, read-only location.

In the build artifact approach, during the build stage, we will publish our container images as part of the build artifact. These can then be used by the subsequent environment-specific deployment stages. However, in typical scenarios, we cannot directly publish a container image as a build artifact.

To do this, we will use the `docker save` and `docker load` commands. These commands are not very well-known in the Docker community but are very useful.

* **docker save**: This command lets us save the container image as a tar file.
    
* **docker load**: This command lets us load the container image from a tar file (created using `docker save`).
    

During the build process, we will containerize the application and save the container image as a tar file. This tar file will then be published as a build artifact.

During the deployment process for each environment, we will load the container image from the tar file in the published build artifact, retag it for the specific environment's container registry, and publish the container image to that registry. These deployment steps will be repeated for each environment.

Here is what the stages in the DevOps pipeline would look like using this approach:

![](https://media.licdn.com/dms/image/D5612AQEUsL4RL6rsOg/article-inline_image-shrink_1500_2232/0/1716010961122?e=1727308800&v=beta&t=SRMruNnjLeUBmPebPllT5KCu2g6rfAOyLcTZ7qkvObw align="center")

This process, though less common, has the following benefits:

1. There is no cost or maintenance overhead from maintaining an additional container registry.
    
2. Organizations can use completely isolated environments.
    
3. The build artifacts are immutable and cannot be tampered with.
    

In conclusion, promoting container images across different environments is a crucial aspect of managing containerized applications. The two strategies discussed—using an additional environment-neutral container registry and leveraging build artifacts—each have their own advantages and drawbacks. The choice between these methods depends on factors such as cost, maintenance overhead, security requirements, and the need for isolated environments. By carefully considering these factors, organizations can select the most suitable approach to streamline their container image promotion process, ensuring efficient and secure deployments across all environments.