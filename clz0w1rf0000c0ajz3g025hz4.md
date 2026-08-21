---
title: "Unlocking Kubernetes Efficiency: Mastering Custom Column Formatting with Kubectl"
datePublished: 2024-07-25T06:24:17.676Z
cuid: clz0w1rf0000c0ajz3g025hz4
slug: unlocking-kubernetes-efficiency-mastering-custom-column-formatting-with-kubectl
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721818418306/263dc559-06c7-4fc6-9113-414770a40856.png
tags: kubernetes, devops, containerization

---

Kubernetes is a powerful system for managing containerized applications in a clustered environment. `kubectl` is the command-line tool for interacting with the Kubernetes API, helping developers and administrators manage their Kubernetes resources efficiently. One of the lesser known but very useful features of `kubectl` is its ability to format output into custom columns. This feature can greatly enhance productivity by allowing users to extract specific information directly from the command line without manually parsing the full output.

## **Understanding Custom Column Formatting**

Custom column formatting with `kubectl` involves using the `-o custom-columns` option. This option lets you define a set of columns and the corresponding JSONPath expressions to extract values from the JSON output of Kubernetes resources.

The basic syntax for custom column formatting is as follows:

```bash
kubectl get [resource] -o custom-columns=[COLUMN_NAME]:[JSON_PATH_EXPRESSION]
```

You can specify multiple columns by separating them with commas. This feature is especially useful when you need to extract specific pieces of information from a list of resources.

## **Practical Examples**

Let's explore some practical examples to see how custom column formatting can be used in real-world scenarios.

### **Example 1: Listing Pods with Custom Columns**

Suppose you want to list all pods in the current namespace, showing only their names and the nodes they are running on. You can do this with the following command:

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

This command defines two columns: `NAME` and `NODE`, which display the pod's name and the node's name, respectively.

### **Example 2: Extracting Container Image Information**

If you want to list the names of the pods along with the images of the containers they are running, you can use:

```sh
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image
```

This command is especially useful for auditing or tracking the container images deployed in your cluster.

### **Example 3: Displaying Resource Requests and Limits**

To monitor the resource efficiency of your cluster, you might want to list the CPU and memory requests and limits for each pod:

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,CPU_REQUEST:.spec.containers[*].resources.requests.cpu,CPU_LIMIT:.spec.containers[*].resources.limits.cpu,MEMORY_REQUEST:.spec.containers[*].resources.requests.memory,MEMORY_LIMIT:.spec.containers[*].resources.limits.memory
```

This command helps you quickly identify pods that might be over or under-allocated in terms of resources.

## **Tips for Advanced Usage**

* **JSONPath Expressions**: Learn JSONPath syntax because `kubectl` uses it to define the data shown in each column.
    
* **Scripting and Automation**: Custom column formatting is very useful in scripts and automation tools. It lets you extract specific data points for monitoring, reporting, or further processing.
    
* **Aliases**: If you often use a specific custom column format, add an alias to your shell configuration to save time.
    

## **Conclusion**

Custom column formatting is a powerful feature of `kubectl` that can make managing Kubernetes resources much easier. By letting you specify exactly what information you want to see, it can save time, reduce complexity, and boost your productivity. Whether you're a developer debugging an application or an administrator managing a cluster, mastering custom column formatting will be a valuable skill in your Kubernetes toolkit.