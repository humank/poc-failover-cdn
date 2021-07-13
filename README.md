# POC for Failover Cloudfront distribution 

    

## Summary

The POC result is - Currently (2021) , AWS Cloudfront doesn't support for **multiple origin distribution for the same domain name**, due to it is global unique constraint. So to have 2 cloudfront distribution to serve as a failover CDN for specific domain is not workable.



## Context

It's all about redundant issue for the managed services, customers may worry about if there is any infrastructure level outage, how to minimize the impact on production workloads. However, to build up **higher availability systems with single regional managed services is to build DR site in another region.** The cost would be high and need to have clear RTO/RPO goal set to move on the progess. 

Most of the time, people would like to seek for regional redundant mechanism to achieve the desire in a cost-effective way, although it doesn't get too much robustness as expected.



### Architecture

![Screen Shot 2021-07-13 at 10.43.23 AM](/Users/yikaikao/git/poc-failover-cdn/images/architecture.png)



Let's take a look on the service discovery chain and how the traffic will be.

The theme is - 

**when client user send a request to https://cdnfailover.kimkao.io, then the Route 53 service resolve the domain name as it has been registered with failover routing policy in Route 53. So the dns revolver will be get cdn1.kimkao.io back since it's the primary domain name.** The traffic will go over cdn1 --> api --> private integration with vpc resources.



## Problem

When trying to propose the above architecture, there is a classical TLS matching problem within. The origin request intention is to visit **cdnfailover.kimkao.io**, although the DNS resolve the **cdn1.kimkao.io** as the target domain name. API Gateway is responsible for receiving all of the incoming requests, so the API Gateway is needed to registered as **cdn1.kimkao.io** as well.

Based on this dns resolving result, all of the traffic will not go through CDN but directly send to API Gateway, then lead to

> CURL error 60: SSL: no alternative certificate subject name matches target host name



## Force

There are some limitation as in existing legacy workloads.

* There are 2 API gateway HTTP API  exposed to serve incoming requests, they are also private integrated with VPC private subnet resources.
* API Gateway serve as the target host, so the domain name, alternate domain name should be registered with the failvoered 2 domains (cdn1.kimkao.io, and cdn2.kimkao.io)

## Solutions

To make the DNS resolving works well, we need to modify all of the DNS records, Cloudfront distributions as below.

### Lambda@Edge

The entire architecture is leveraging route53 to deal with servcie lookup for healthy CDN endpoint, and pass  all of the requests through multiple network components.

> Client --> Ask route53 --> visit cdn primary endpoint --> go Origin to API Gateway --> through VPCLink to ALB --> proxy request to backend EC2.

When working on these network traffic, do need to make sure the request header matches destination host name. However, when request ask to visit https://cdnfailover.kimkao.io, this domain name is recorded on Route 53, which is not a real distribution on Cloudfront, so the cloudfront received the request will reject the request due to TLS host name doesn't match to the requested one.

How can we resolve this issue? A traditional approach is to add the "host" header in runtime, or on CloudFront. But CloudFront service doesn't allow to add host header directly. We could only leverage Lambda@Edge to override the request header in runtime. Here are the instruction guidances.





### CloudFront distribution setting

#### CDN distribution -1 

```
Distubrition domain name : https://d3e36777gjas6.cloudfront.net (auto-generated)
ARN : xxxx ( auto-generated)
Description : CFN1 to API 1 
Supported HTTP versions : HTTP/2, HTTP/1.x
alternate domain names : cdn1.kimkao.io, cdnfailover.kimkao.io ( to make sure the origin request header match the CDN destination)
origin domain : (set to api gateway domain name) d-q6v52yluri.execute-api.us-east-1.amazonaws.com
```

#### Behaviors

```
Viewer protocol policy : HTTP and HTTPS
Allowed HTTP methods : GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
Restrict viewer access: No
```

#### Cache key and origin requests

```
Cache policy and origin request policy (recommend)
Cache policy :  CachingDisabled
Origin request policy : AllViewers
```



#### Function associations

```
Everything leave default but add one function for Origin request.

Rogin request : Function type as Lambda@Edge, Function ARN/ Name as you deployed lambda function ARN with version number
```







#### CDN distribution -2

You could create the second distribution as first distribution steps, but don't add the alternate domain name **cdnfailover.kimkao.io** at the time.

### Route 53 setting

* Create a DNS record with failover routing policy , domain name  - cdnfailover.kimkao.io

```
Domain name : cdnfailover.kimkao.io
Routing policy : failover
Primary : Cloudfront distribution1 - d3e36777gjas6.cloudfront.net
Secondary : Cloudfront distribution2 - d2dufh55qr16mi.cloudfront.net
```



