---
title: "Transforming Retry-After Headers in Azure APIM: A Step-by-Step Guide"
datePublished: 2024-07-29T06:17:22.397Z
cuid: clz6lk9nh000q09jz10dy3j4g
slug: transforming-retry-after-headers-in-azure-apim-a-step-by-step-guide
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1722233156969/cc342e9d-252c-489f-9aab-fbd8e4f94744.jpeg
tags: azure, apim, api-management, ratelimting

---

In this article, you'll learn how to customize the Retry-After response header in Azure API Management (APIM) rate-limiting policies, enhancing your API's flexibility and user experience. While it does not delve into the specifics of the **rate-limit** or **rate-limit-by-key** policies, it provides a practical guide for altering the Retry-After header. For detailed information on the rate-limit policy, please visit [Azure API Management policy reference - rate-limit | Microsoft Learn](https://learn.microsoft.com/en-us/azure/api-management/rate-limit-policy).

# Understanding Rate Limiting: Protecting Your API from Overuse

Rate limiting is a technique used to control how often requests are made to a resource. It helps prevent excessive or abusive use and ensures the resource is available to all users.

Rate limiting is often used to protect against denial-of-service (DoS) attacks, which aim to overwhelm a network or server with too many requests, making it unavailable to legitimate users. It can also limit the number of requests from individual users to prevent a single user or group from monopolizing the resource.

# Azure API Management Rate Limit Policies

In Azure, access to APIs is controlled using the following API Management policies:

* rate-limit
    
* rate-limit-by-key
    

The implementation of these policies is straightforward but somewhat limited and less flexible, in my opinion.

# The Default Retry-After Header: What You Need to Know

In the Azure APIM rate-limit policy documentation, it is mentioned that once the client's requests are throttled, the service starts returning a response header containing the time interval (in seconds) after which the client should retry the request. The default name of the header is **Retry-After**, and this name can be customized.

For example: **Retry-After**: 60

However, in one use case for a customer, there was a requirement to provide a timestamp instead of a time interval as a header value.

For example: **Retry-After**: 2020-05-04T12:23:41.6181792Z

To implement this, the header value needs to change, but this is something that the rate-limit policy does not support.

# Customizing the Retry-After Header

The basis for changing the response header value lies in the on-error scope. You can implement a policy like the following:

```xml
<inbound>
  <base />
  <rate-limit-by-key calls="1000" renewal-period="60" counter-key="@(context.Request.IpAddress)" increment-condition="@(context.Response.StatusCode == 200)" remaining-calls-variable-name="remainingCallsPerIP" retry-after-header-name="Retry-After" remaining-calls-header-name="Requests-Remaining" retry-after-variable-name="retryAfter" />
</inbound>
```

```xml
 <on-error>
   <choose>
     <when condition="@(context.LastError.Reason == "RateLimitExceeded")">
       <set-header name="Retry-After" exists-action="override">
         <value>@(DateTime.UtcNow.AddSeconds(context.Variables.GetValueOrDefault<int>("retryAfter")).ToString("o"))</value>
       </set-header>
     </when>
  </choose>
  <base />
</on-error>
```

Please refer to APIM predefined errors for policies here: [Error handling in Azure API Management policies | Microsoft Learn](https://learn.microsoft.com/en-us/azure/api-management/api-management-error-handling-policies#predefined-errors-for-policies)

Here, the key point is that whenever the APIM rate limit is reached, an error occurs, which is then captured in the on-error scope. To set or override the response header only in rate-limiting scenarios, you need to filter using the RateLimitExceeded error reason. After that, the exact error value is determined by adding the current UTC timestamp with the value of the retryAfter variable in seconds.

With this, you have now customized the Retry-After header with a timestamp instead of a time interval (in seconds).

# Conclusion

In conclusion, customizing the Retry-After response header in Azure API Management can significantly enhance the flexibility and user experience of your API services. By leveraging the on-error scope and handling the RateLimitExceeded error, you can provide a more informative and user-friendly response to clients when rate limits are exceeded. This approach not only meets specific customer requirements but also demonstrates the adaptability of Azure APIM in handling various scenarios. With these steps, you can ensure that your API remains robust, efficient, and user centric.