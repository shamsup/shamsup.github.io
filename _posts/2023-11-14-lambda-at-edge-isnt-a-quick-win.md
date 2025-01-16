---
title: "PSA: Lambda@Edge isn't a quick win"
# author: shamsup
date: 2025-01-16 12:38:00 -0800
description: Do the benefits outweigh the complexity for you?
# tags: [aws, edge, cloud]
---

_Originally published on November 14, 2023 to [dev.to](https://dev.to/shamsup/psa-lambdaedge-isnt-a-quick-win-3252)._

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> This is based on a post I originally wrote in the Remix Discord server in October of 2022 titled "Lambda@Edge isn't the same 'Edge'" to weigh Lambda@Edge against something like Cloudflare, but I decided to turn it into more of a blog post around the pitfalls of edge computing itself with the recent news around the Partial Pre-render feature from Vercel and [them _refining_ their stance on edge-rendering](https://x.com/acdlite/status/1724151383136330233){:target="_blank" rel="noopener"}.
{: .prompt-info}
<!-- markdownlint-restore -->

---

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> **The short of it**: Just stick to CloudFront + Lambda and **avoid** Lambda@Edge unless you need to manage multi-region deployments anyway.
{: .prompt-tip}
<!-- markdownlint-restore -->

---

## What is Lambda@Edge?

Lambda@Edge is AWS's serverless function service that automatically distributes your code across regions to execute near your users. It uses CloudFront, AWS's CDN service, to detect the closest AWS region to the requesting user and starts a Lambda function in that region, the idea being to reduce the latency between the user and the executing code. It has a few limitations compared to the normal Lambda service, which I'll cover, but it's essentially the same thing.

## How is it a different from Cloudflare's "Edge"?

AWS Lambda@Edge is not the same "edge" that something like Cloudflare Workers runs on. Lambda@Edge is essentially just your Lambda being automatically deployed to other regions dynamically based on where requests are coming in from. This is to say that a request coming from Washington (state) might hit a Lambda in `us-west-2` region, while you deployed to `us-east-1`. **It will still access your resources wherever they are deployed**, so a DynamoDB table or RDS cluster will stay in whatever region you deployed it in, but the Lambda will use AWS's region-to-region optimized network to access those resources from whatever region the Lambda is executed in.

Compared to Cloudflare's pages or workers, where the code actually runs on the CDN servers themselves, there will be latency involved between the CloudFront edge server calling the nearest region for your Lambda, and the region potentially having to cold-start an instance of your function. In Cloudflare, there is virtually 0 cold-start because [they are using V8-isolates](https://developers.cloudflare.com/workers/learning/how-workers-works/#isolates){:target="_blank" rel="noopener"}, which has a much lower overhead to start compared to Lambda's Node runtime.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> CloudFront also has [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html){:target="_blank" rel="noopener"}, but they are severely limited in scope, like not being able to execute any asynchronous code
{: .prompt-warning}
<!-- markdownlint-restore -->

## Why is Lambda@Edge bad?

Well there are 2 categories I want to address here: edge computing on its own, and Lambda@Edge vs AWS Lambda. It's also not _bad_, but it may take more effort than you expect to realize a win by using it.

## Edge Computing's Biggest Hurdle: Distance to the Origin

Edge computing has a major pitfall for executing code with data dependencies: latency to the origin. If the code is querying a database or hitting an API in a different region, the latency from requests to the data source can easily end up taking longer than the latency you save for your user by moving the code closer to them, especially when doing any requests in series (or request waterfalls).

As an example, let's use something I tested on my own: DynamoDB deployed to `us-east-1` with Lambda@Edge executing in `us-west-2`, where I live. I have an application that uses database-backed sessions, where a cookie in the user's browser is just an ID for an item in the database that contains the actual session contents. Most requests begin by retrieving the user's session before proceeding with any other functionality so that we can do things like check the user's permissions to access the resource they are requesting, account-level rate-limiting, read user preferences, read feature flags, and so on.

A typical request from my computer in Vancouver, WA to `us-east-1` through CloudFront takes about 60ms round-trip. This latency will be very similar to AWS's region-to-region latency for `us-west-2` because CloudFront routes through the same network. You can check the latency between other regions on [CloudPing](https://www.cloudping.co/grid){:target="_blank" rel="noopener"}. This cross-region latency "tax" is paid once between CloudFront and the origin. Once the code is executing in `us-east-1`, the same region as the database, the requests to the database have 1-2ms of latency each, which is pretty miniscule compared to the overall latency between the user and the origin. Say reading the session record takes ~10ms, 1-2ms of that is latency, then we request the resource from the database with another 10ms total with 1-2ms of latency. We then return a response from Lambda, which gets routed back through the region-to-region network to CloudFront's edge, then back to the user. Overall, this takes about 200ms, with about 75ms of unavoidable latency just due to physical distance. There's other overhead by using Lambda in the first place, but we can ignore it for this purpose.

We can add up the total cost of just the latency from the physical distances of the requests:

<figure>
<table class="text-right">
  <thead>
    <tr>
      <th>Hop</th>
      <th>us-east-1</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>User to CF</td>
      <td>10ms</td>
    </tr>
    <tr>
      <td>CF to origin</td>
      <td>60ms</td>
    </tr>
    <tr>
      <td>DB request 1</td>
      <td>2ms</td>
    </tr>
    <tr>
      <td>DB request 2</td>
      <td>2ms</td>
    </tr>
    <tr>
      <td><strong>Total</strong></td>
      <td><strong>74ms</strong></td>
    </tr>
  </tbody>
</table>

<!-- 
| Hop          | us-east-1 |
| ------------ | --------- |
| User to CF   | 10ms      |
| CF to origin | 60ms      |
| DB request 1 | 2ms       |
| DB request 2 | 2ms       |
| **Total**    | **74ms**  |
{: .text-right} -->

<figcaption>

Latency due to physical distance to a single Lambda in us-east-1 from the us-west-2 area

</figcaption>

</figure>

Now let's compare this to the same application, but the code is deployed to Lambda@Edge instead of Lambda. We now have 10ms latency to the code instead of only to CloudFront (we'll give the benefit of the doubt and say that there is 0 additional latency between CloudFront and the nearest AWS Region). However, remember our code is executing in `us-west-2` but our database is in `us-east-1`, so each database request pays the 60ms region-to-region latency because we have to wait for the first query to complete before requesting the second resource. We then return the response to the user, the full request taking about 260ms, with roughly 130ms of that being unavoidable due to physical distance.

<figure><table class="text-right">
  <thead>
    <tr>
      <th>Hop</th>
      <th>us-east-1</th>
      <th>us-west-2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>User to CF</td>
      <td>10ms</td>
      <td>10ms</td>
    </tr>
    <tr>
      <td>CF to origin</td>
      <td><span class="danger">60ms</span></td>
      <td><span class="success">0ms</span></td>
    </tr>
    <tr>
      <td>DB request 1</td>
      <td><span class="success">2ms</span></td>
      <td><span class="danger">60ms</span></td>
    </tr>
    <tr>
      <td>DB request 2</td>
      <td><span class="success">2ms</span></td>
      <td><span class="danger">60ms</span></td>
    </tr>
    <tr>
      <td><strong>Total</strong></td>
      <td><span class="success"><strong>74ms</strong></span></td>
      <td><span class="danger"><strong>130ms</strong></span></td>
    </tr>
  </tbody>
</table>

<!-- 
| Hop          | us-east-1                             | us-west-2                             |
| ------------ | ------------------------------------- | ------------------------------------- |
| User to CF   | 10ms                                  | 10ms                                  |
| CF to origin | <span class="danger">60ms</span>      | <span class="success">0ms</span>      |
| DB request 1 | <span class="success">2ms</span>      | <span class="danger">60ms</span>      |
| DB request 2 | <span class="success">2ms</span>      | <span class="danger">60ms</span>      |
| **Total**    | <span class="success">**74ms**</span> | <span class="danger">**130ms**</span> |
{: .text-right} -->

<figcaption>

Comparing latency between a single Lambda in us-east-1 and a Lambda@Edge deployment when requested from the us-west-2 area

</figcaption>

</figure>


<!-- ```txt
         User to CF: 10ms
CF to origin Lambda:  0ms
       DB request 1: 60ms
       DB request 2: 60ms

      Total Latency: 130ms
``` -->

The solution to the data locality problem is to do multi-region deployments of your data as well, which is much harder to do correctly and comes with its own set of tradeoffs.

## Lambda@Edge vs Lambda

The next few points will focus on differences between Lambda@Edge and Lambda.

1. **No unified logging**. Lambda@Edge's logs will be in Cloudwatch, but they will be in whichever region the function executed in, making it **your** job to aggregate the logs into a single place. Knowing which regions your function has been executed in is just the first part of this problem.
2. **No function versioning**. There are no function versions or aliases for Lambda@Edge, so canary deployments or other partial rollout solutions are a significant undertaking. I honestly don't have a solution in mind for this challenge.
3. **No environment variables**. You will either need a solution to inject values into the executable at build-time, or use a service such as SSM to store runtime configuration for Lambda@Edge.
4. **Deploys take significantly longer**. Lambda@Edge deployments have to propagate out to the CloudFront distribution, just like any other change to the distribution configuration, so the major bottleneck for deployment time is the time it takes to propagate a change out to CloudFront, which regularly takes 15 minutes or more. This can significantly reduce velocity, especially when iterating on new features or attempting to debug problems in a deployment.
5. **Increase in cold starts**. We could ignore this point, since every request in an app with 0 users will be a cold start anyway. However, if you want to explore a hypothetical with me, let's say we have more than 0 users. Lambda@Edge fragments the pool of warm Lambda instances with your code across regions, which could lead to an increase in the cold-start ratio for your application depending on the geographic distribution of your userbase. If all users are routed to the same region as they are without Lambda@Edge or multi-region deployments, they share a pool of Lambda instances, which increases the likelihood that one is available as the Lambda service juggles concurrent requests.

## Conclusion

Unless you are working on something big and have a lot of developer hours to put into monitoring, management tooling, and cross-region deploys/maintenance for your dependencies, **I would highly advise you not to use Lambda@Edge** and just trust AWS's internal network to route from CloudFront to your static AWS regions automatically.

---

There are many valid use-cases for edge computing, but it's hard to do correctly. Generally speaking, your compute resources need to live close to the data that they depend on. There are a ton of projects and companies working on bringing your data closer to the edge, which is an awesome problem space. Cloudflare, Fly.io, and Turso, to name a few. The purpose of this article, again, was to highlight the challenges of moving to an edge compute model specifically with Lambda@Edge and should not be taken to discount the viability of edge computing as a whole. Vercel's Partial pre-render feature is incredibly cool for taking no extra effort on the developer's part and getting a ridiculously fast TTFB. I'm excited about the future of this space.
