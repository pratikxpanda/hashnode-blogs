---
title: "Streamlining Azure API Management: A Layered Strategy with App, Process, and Core APIs"
datePublished: 2024-07-26T11:26:35.599Z
cuid: clz2made7000a09mj3lacbvis
slug: streamlining-azure-api-management-a-layered-strategy-with-app-process-and-core-apis
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721994130306/c41de80d-f6df-42c1-aac6-83d847f48a57.jpeg
tags: azure, cloud-computing, devops, api-design, api-management

---

In today's interconnected digital landscape, designing and composing APIs is a complex yet crucial task. A well-structured API strategy can significantly enhance system integration and performance. One effective approach is the layered API method described by [MuleSoft](https://blogs.mulesoft.com/dev-guides/how-to-tutorials/api-templates-reusable-system-process-apis/), which categorizes APIs into three distinct layers:

* **Experience APIs**: These are APIs created for specific apps, also known as BFFs (Backends for Frontends).
    
* **Process APIs**: These APIs orchestrate business functions based on specific domains and are presented as a consistent set of APIs.
    
* **System APIs**: These are the standard API endpoints of your backend systems or third-party backend systems.
    

This post describes important aspects of how you can achieve this layering within Azure API Management.

# Simplifying the Terminology

**Redefining API Layers: App, Process, and Core APIs**

For some reason, I am not completely comfortable with the terms **Experience** and **System** APIs. Instead, I prefer to use the terms **App** and **Core** APIs. Therefore, our implementation of the layered API Management design will consist of the following layers:

* **App APIs**: These are APIs created for specific apps, also known as BFFs (Backends for Frontends). They don't have to be frontend apps and can be used by backend services too. They can either connect to Process APIs or directly to Core APIs, depending on the needs.
    
* **Process APIs**: These APIs manage the orchestration between multiple Core APIs.
    
* **Core APIs**: These are the APIs that connect to the actual backend systems. By defining a set of Core APIs, we create a single point of reference for the APIs that need to be exposed from our backend systems, avoiding duplication in multiple places.
    

Here is how the calls to the backend systems will be directed from the clients:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1721992757008/1afbd985-bd44-4c23-8bed-babfe188e3ae.png align="center")

# Examples and Use Cases

**Real-World Applications: Layered API Strategy in Action**

For instance, consider an e-commerce platform. The App APIs could handle user-facing functionalities like product search and order placement, the Process APIs could manage order processing and inventory updates, and the Core APIs could interact with the underlying database and third-party payment gateways.

# **Defining the API Layer**

**Establishing Clear API Layers in Azure API Management**

Azure API Management does not have built-in support for layers. If you want to implement layering, it should be clear to everyone which layer an API belongs to. To make this clear, you need to start with an analysis of the API's scope and purpose.

A clear way to indicate the API layer is by using a suitable naming convention for your APIs. For example, if you have a set of customer-oriented APIs to be exposed from your backend systems as part of Core APIs, you can name the corresponding API **Core - Customer API** in API Management. Similarly, if you have a set of APIs to be used by a frontend app called Portal, you can name the corresponding API **App - Portal API**.

In addition, it would be helpful to indicate the API layer by using the API Tags available in API Management. However, this step is optional. You can define the following tags and assign them to APIs based on their category:

* **App**
    
* **Process**
    
* **Core**
    

Once the API layers are clearly defined, the next crucial step is to establish efficient routing mechanisms to ensure seamless communication between these layers.

# **Routing**

**Optimizing API Routing for Efficiency and Security**

If you want to route an API request from one API layer to another, it's best not to call the external DNS of Azure APIM again, as this would require an extra network hop. Additionally, firewall rules might block this if your API Management instance is protected by a Virtual Network. The simplest and fastest way to call another API exposed on the same API Management instance is by using its address via [***localhost***](http://localhost). This way, you stay within the same gateway. If you want to do this, it is required to set the **Host HTTP header**, as described in this detailed [blog post](https://www.codit.eu/blog/configure-loopback-services-azure-api-management/).

```xml
<!--Set hostname for interal routing to Core-APIs-->
<set-header name="Host" exists-action="override">
   <value>your-apim-hostname</value>
</set-header>
<!--Point to Customer-API-->
<set-backend-service base-url="https://localhost/customer-api/v1.0" />
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">A more suitable way to define the backend for Core APIs is to use the <strong>Backends</strong> capability of Azure API Management. This way, any suitable parameters, headers to call the backend can be defined in the API Management backend directly.</div>
</div>

# **Security**

**Securing Your APIs: Best Practices for Layered API Management**

Even when you perform such routing across different layers of API Management, the APIs have to be secured. I suggest implementing the design that **Core** APIs can only be consumed by **App** or **Process** APIs in API Management. To make that happen, you can leverage **Products** capability of API Management.

You can define a product named **Core** which includes all the Core APIs and requires subscription key which is only available to internal APIM users. Any request from App or Process APIs needs to include the subscription key as part of the request. I suggest using a custom header name for the name (preferably something like **Ocp-Apim-Core-Subscription-Key**). A similar approach can be taken for Process APIs as well.

# **Monitoring**

**Effective Monitoring: Keeping an Eye on Layered API Interactions**

When you implement a layered design, monitoring is crucial. If your API Management is not set up for monitoring, it will become a difficult-to-manage black box, especially for inter-API communications. However, by configuring Application Insights, you can use native API monitoring to get a clear view of these API-to-API interactions.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">One downside to note is that the tracing feature of API Management does not work end to end across the different layers.</div>
</div>

# **Conclusion**

**The Benefits of a Layered API Approach in Azure API Management**

In summary, a layered API design within Azure API Management offers a robust framework for organizing, securing, and maintaining your APIs. By clearly defining App, Process, and Core APIs, you streamline development and integration processes. Efficient internal routing and secure communication through subscription keys further enhance system reliability. While monitoring with Application Insights provides valuable insights, be mindful of its limitations in end-to-end tracing. Embracing this layered approach can lead to a more scalable and efficient API ecosystem, benefiting both developers and end-users alike.