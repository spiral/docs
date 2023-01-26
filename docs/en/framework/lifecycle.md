# Framework — Application Lifecycle

The Spiral Framework can be used with a traditional Nginx/PHP-FPM setup, where Nginx acts as the web server and PHP-FPM
processes incoming requests and initializes the application. However, this can be resource-intensive as it requires CPU
and memory resources to complete with every incoming request.

![Nginx HTTP bootstraping](https://user-images.githubusercontent.com/773481/211190445-06c17d86-58d6-43d8-995f-36cf448714ae.jpg)

To improve the performance of the Spiral Framework, an alternative solution is to use application
server [RoadRunner](https://roadrunner.dev/).

RoadRunner is a high-performance application server designed to handle a wide range of request types, including HTTP,
gRPC, TCP, Queue Job consuming, and Temporal. It operates by running workers only once when it is initiated and then
directing requests to a [dispatcher](../framework/dispatcher.md) based on their type. This means that each worker is
isolated and works independently, following a "share nothing" approach where resources are not shared between workers.

> **Note**
> The approach of running application is similar to other languages like **Java**, **C#**, etc.

This allows developers to write code in a way that is familiar to them and allows workers to be horizontally scaled,
improving resource utilization and increasing the capacity to handle incoming requests.

![RoadRunner HTTP bootstraping](https://user-images.githubusercontent.com/773481/211197998-96b09ff1-4ede-4db0-9b1d-902e996920be.jpg)

Using RoadRunner can significantly improve the speed and efficiency by eliminating the need for
the application to go through the bootstrapping process repeatedly. This can save on CPU and memory resources and reduce
response time.

Another cool thing is that it allows developers to be a bit more flexible with how they bootstrap the application. For
example, they can load configs or routes from various sources like attributes or files without having to worry about
caching them. This can make it easier to modify and update the application as needed.

> **See more**
> Read more about Framework and application server symbiosis n the [Framework — Design Approach](../framework/design.md)
> section. Read about PSR-7 request flow in the [HTTP — Request Lifecycle](../http/lifecycle.md) section.
