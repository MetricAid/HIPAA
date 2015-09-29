---
title: Building a Secure, Multi-Tenant Docker-based Platform as a Service - Part 1 - Design Considerations
date: 2015-09-04
author: mohan
author_full: Mohan Balachandran
author_alt: Co-Authors - Adam Leko, Nate Radtke, Vince Kenney
tags: PaaS, Platform as a service, Docker, Multi-tenancy, Orchestration, Docker Networking
---

<blockquote>
    <p>Detailed explanations are just apologies in long form.</p>
    <p>- Adam Leko</p>
</blockquote>

This article is intended to provide a top down introduction to the design of the Catalyze Platform as a Service (PaaS). It will additionally delve into the design of the components needed to make our vision a reality and what that means for you as a customer. This will include the design choices, trade-offs, and a quick view into the next generation of these components. Or in the more succinct words of the author of the above quote - "all of the major orchestration components, how and why they were designed, and a brief history of how they managed to come into existence."

*Credits: Our lead engineer, Adam Leko wrote up chunks of this around July of 2014 (yep, that long ago). Other sections have been contributed by Nate Radtke and Vince Kenney.*

# Motivation

<blockquote>
    <p>People often say that motivation doesn't last. Well, neither does bathing - that's why we recommend it daily.</p>
    <p>- Zig Ziglar</p>
</blockquote>

HIPAA compliance is painful. It requires huge effort for our customers if they were to do it themselves. This is not only because of the technical requirements but also because of the [administrative](https://hipaa.catalyze.io/#administrative-safeguards-see-a-hrefhttpwww-hhs-govocrprivacyhipaaadministrativesecurityruleadminsafeguards-pdf164-308a) and [policy requirements](https://hipaa.catalyze.io/#policies-and-procedures-and-documentation-requirements-see-a-hrefhttpwww-hhs-govocrprivacyhipaaadministrativesecurityrulepprequirements-pdf164-316a) imposed by HIPAA. The Catalyze Backend-as-a-Service (BaaS, or sometimes just "the API") provides a nice, easy-to-use method for spinning up new mobile or web applications, but what about customers that have already invested in an application which uses their own code or backend? We can't expect customers to completely rewrite their applications.

For these customers, we offer a standard platform on which to host applications. The platform takes care of all the prerequisites for HIPAA compliance while adding useful features like standardized logging and monitoring. This offering has generated a lot of interest, which is not surprising given that these features are offered in combination with our core competencies.

The first few customers that signed on to the PaaS platform were migrated, deployed, and managed completely manually. This was a good way to prove out the utility of the platform but is incredibly difficult to scale.  Rather than using a brute-force approach to growing the platform (adding hordes of staff), we started working on ways to automate deploying and managing customers' applications. The end result draws heavy inspiration from [Heroku](http://www.heroku.com) with some unique twists to satisfy the additions needed for HIPAA compliance.

# Core Design Considerations

<blockquote>
    <p>Don't think about the faster way to do it or the cheapest way to do it, think about the most amazing way to do it.</p>
    <p>- Richard Branson</p>
</blockquote>

As an obvious starting point, commoditization of software components used to build web applications has made it much easier to support varied customer environments. Web frameworks such as [node.js](http://nodejs.org) and [Ruby on Rails](http://rubyonrails.org/) along with database and storage technologies like [MySQL/Percona](https://www.percona.com/), [PostgreSQL](http://www.postgresql.org/), and [MongoDB](http://www.mongodb.com/) (among others) on top of Linux have become so common that adding support for these services enables support for a *wide* variety of user applications.

In addition, virtualization technologies make it cost-effective to run customer applications and services on a relatively small amount of hardware. Virtual machines provide a nice way of segregating customer environments but tend to be a bit heavyweight with resource usage. Containers and jails provide a lighter-weight solution but can be tricky to manage at larger scales without comprising security guarantees. [Docker](http://www.docker.com) is an interesting take on the container approach; on the outset it appears to provide little advantage over [Linux Containers](https://linuxcontainers.org/) until you take a look at the additional management utilities provided by it and the ecosystem surrounding it. As it turns out, Docker solves a lot of the operational issues you would have to solve if you attempted to build a platform for managing deployments using containers. It is a relatively young project but has improved very quickly, particularly over the last year.

We made the decision to go with Docker (and all its attendant challenges) in the Summer of 2014. Docker was at Release 0.6 then.

Once that decision was made, it led to a whole series of design considerations for which solutions did not -- and still don't -- exist. So we had to develop a lot more than we initially anticipated. In retrospect, we regret the decision not to open source it all as that would, we believe, have benefited the broader Docker community. But hindsight is 20-20. We chose to prioritize our specific company needs and revenue targets over an open source model.

In any case, there are multiple design considerations that we had to take into account and develop towards, which we discuss below.

## Multi-Tenancy

<blockquote>
    <p>The key to performance is elegance, not battalions of special cases.</p>
    <p>— Jon Bentley and Doug McIlroy</p>
</blockquote>

The goal of Catalyze is to simplify compliance for its customers, which also implies that hosts will have containers belonging to different customers. This leads to the design requirement that each customer's application, data and traffic must be isolated and segregated. Isolating containers implies configuration rules around AppArmor and SELinux. Data segregation implies encryption with customer-specific keys and associated block stores. Traffic isolation implies customer-defined networks (what connects and talks to what) with encryption of traffic using customer-specific TLS keys.

Simply using HTTPS to the application is not sufficient as connections between the application and the database or other services must be encrypted as well. Also, some services have missing, incomplete, or poorly-performing options for encrypting traffic, and we wanted to avoid pushing the complexity of certificate management and secure key distribution down to our customers.

## Compliance & Security

<blockquote>
    <p>I want security, yeah. Without it I had a great loss, oh now.</p>
    <p>- Otis Redding, "Security" from the album "Dreams To Remember"</p>
</blockquote>

These are tightly coupled. Compliance defines specific security requirements. These are often not sufficient because regulatory requirements lag technology advances but they are excellent for defining the baseline capabilities. Regulatory requirements vary depending of not only industry (HIPAA, PCI...) but also geography (NIST, Safe Harbor...). Part of the reason we focused on HIPAA is because from a healthcare perspective, HIPAA (and its more prescriptive cousin, HITRUST) is the only healthcare specific standard and is often held as the gold standard as other countries realize the value and importance of protecting healthcare data. HIPAA compliance implies administrative and technical protocols that need to be followed. From a technical perspective, it implies encryption at rest and in transit (see above section), data integrity, prevention of data loss and service (backups, disaster recovery), industry best practices around system hardening (vulnerability scans, IDS), logging, and system monitoring.

## Building Images

<blockquote>
    <p>Start where you are. Use what you have. Do what you can.</p>
    <p>- Arthur Ashe</p>
</blockquote>

This is where things got a little philosophical: Should we allow any custom image derived from a customer-supplied `Dockerfile` or should we be prescriptive? Prescriptiveness implies lack of flexibility which most developers aren't very comfortable with, but that lack of flexibility allows us to build in much tighter security capabilities.

The customer interface to our build system currently uses a `git-push` model using [progrium's](http://progrium.com/blog/) [buildstep](https://github.com/progrium/buildstep), which in turn leverages [Heroku's buildpacks](https://devcenter.heroku.com/articles/buildpacks). This more prescriptive approach lets us do some interesting things automatically during deploy (particularly with log management and service monitoring) and has worked well so far. We are also working towards allowing customers to leverage raw `Dockerfile`s under constraints based on our security requirements and experience with using large numbers of containers in production.

## Docker Registry

<blockquote>
    <p>These words are mine, those notes are mine, this song is mine...</p>
    <p>- Merle Haggard</p>
</blockquote>

Given that we need to store customer build artifacts so they can be pulled down on other hosts during deployment, a private Docker registry was automatically a requirement. We value our customer's privacy and intellectual property; we feel it is absolutely necessary to treat build artifacts the same way we treat the rest of our customer's data. This led us to learn a lot about the challenges of managing and maintaining a private registry which we intend to share soon in a separate blog post.

## Service Registry & Discovery

Service discovery is a key component of most distributed systems and service oriented architectures. The problem seems simple at first: How do clients determine the IP and port for a service that exist on multiple hosts? There are two sides to the problem of locating services, *viz.*, Service Registration and Service Discovery.

- **Service Registration**: The process of a service registering its location in a central registry. Services typically register details about location (host and port), protocols, version numbers, and environment details.
- **Service Discovery**: The process of a client application querying the central registry to learn of the location of services.

Any service registration and discovery solution also has other development and operational aspects to consider, such as monitoring (what happens when a registered service fails?), load balancing, availability concerns, etc. Our goals was to keep this component as simple as possible to maximize uptime and minimize maintenance. We looked at [several open-source options](http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/); their main shortcomings were limited or nonexistent support for multi-tenancy.

Multi-tenancy is a very important feature for our platform. Given the additional protections we have in place between tenants it isn't strictly necessary, but leaking service information may give potential attackers a foothold into launching targeted attacks on our customers (even if those are conducted externally to the platform). The [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) is one of the guiding tenants of our platform design that we didn't want to violate in our service discovery layer. In the end, we ended up reluctantly building our own in-house service discovery solution.

## HA Considerations

<blockquote>
    <p>You go - Up... Up... Up... Up... Up... Up... Up... Up.</p>
    <p>- Wiz Khalifa, "Up"</p>
</blockquote>

It is of course, easy to spin and wire up a container. But it is also critical to offer highly available options to our customers to maximize their uptime. Healthcare services can be mission and life critical as well. We decided to approach the problem of HA services by providing redundant components and the ability to quickly restore a service component. These HA capabilities would need to extend to the [application languages](https://resources.catalyze.io/paas/getting-started/deploying-your-first-app/supported-languages-frameworks/), the [databases](https://resources.catalyze.io/paas/getting-started/deploying-your-first-app/supported-databases/), and the [cache](https://resources.catalyze.io/paas/getting-started/deploying-your-first-app/supported-add-ons/) offerings. 

## Networking

<blockquote>
    <p>The problem with the Internet is that it is meant for communications among non-friends.</p>
    <p>- Whitfield Diffie</p>
</blockquote>

This is the biggest problem that we had to overcome. Solutions that existed at the time (such as [Skydock](https://github.com/crosbymichael/skydock) and [Pipework](https://github.com/jpetazzo/pipework)) provide some of the features, but lacked things like encryption, access models, and automation. There are other existing ways to solve this problem, even transparently, but they have their own sets of issues. [OpenVSwitch](http://openvswitch.org/) has had a lot of success in OpenStack environments but getting it working reliably in a Docker-based environment is generally tricky and tends to be somewhat error-prone. Also, [Rackspace's experiences with OpenVSwitch](https://www.youtube.com/watch?v=_OdPP_4PYD4&t=256) were a little worrying. Using hardware VLANs in conjunction with IPSec tunneling is arguably the easiest solution to get going if you have hardware to help you along the way, but most cloud providers do not offer environments with dedicated switching hardware, and the ones that do make that quite a capital-intensive proposition. We decided to tackle container-to-container encryption using an in-house solution.

Rather than attempt to shoehorn existing technologies into this environment, we decided to take a radically different software-based approach. We eventually settled on a proxy approach in which all communication between containers is tunneled over TLS-encrypted connections that are set up on demand. This was our initial solution to solve the networking problem. It is also interesting to note that even now, solutions such as Weave still have [performance](http://www.generictestdomain.net/docker/weave/networking/stupidity/2015/04/05/weave-is-kinda-slow/) and [isolation](https://github.com/weaveworks/weave/issues/354) issues when encryption is turned on.

We intend to showcase our next generation of container-to-container networking which can achieve 1Gbps line rates on modest hardware with full encryption turned on soon.

## Orchestration

<blockquote>
    <p>Make it so</p>
    <p>— Jean-Luc Picard, Captain of the USS Enterprise, Star Trek</p>
</blockquote>

Based on all of the above, it was obvious that we needed an orchestration layer that would manage the interactions between all these various components while ensuring the core requirements enforced by the security and compliance needs. So, at the very minimum, the orchestration layer (perhaps made up of several sub-components) would be responsible for:

- **Job distribution**: Which container goes on which host. For example, containers that are part of a specific HA (highly available) configuration cannot go on the same host whereas cache containers would prefer to be on the same host as the code container if there is available capacity. Other considerations include available machine resources, how many containers are already on a host, and the current load on a host. Note that we do not over-provision our hosts.
- **Job scheduling**: Deployment of these containers have to be scheduled. The key driver for scheduling came from our backup jobs. As it turned out, most of the underlying filesystems supported by Docker at the time (such as `btrfs`) handled significant write loads poorly, particularly when Docker was busy pulling down new image layers. Scheduling became a critical need very quickly.
- **Bidding**: Required to answer the question of which host does the container get deployed on. The primary constraint to be taken care of here would be the remaining capacity on the host based on the requirements of the customer container (RAM, CPU, etc.). The approach we landed on was to allow the hosts in the fleet to "bid" on the job by providing a score generated by the host. The job scheduler would then take care of deploying the container to the winning host.
- **Tracking**: A centralized record of what containers are running on which hosts and who (customer and network) they belong to. VMs / hosts die and when they do, rapid recovery is necessary.
- **Garbage collection**: It's a dirty job but someone has to do it. The interaction between Docker and the filesystem (`btrfs`) can lead to some unfortunately edge cases that cause the filesystem to quickly degrade. Combine that with the overhead of the filesystem (up to 50% without aggressive rebalancing), garbage collection became a very critical component to the stability of the platform. This was discovered down the road once we had a significant number of customers and thus containers (and hosts) that we had to manage. 

## Monitoring

<blockquote>
    <p>Доверяй, но проверяй (doveryai, no proveryai). In English, "Trust but verify."</p>
    <p>- Russian Proverb</p>
</blockquote>

HIPAA and HITRUST can also be interpreted to **require** a monitoring service, i.e., something that tracks running applications to ensure that they are in a functional state. That is something that we would be able to utilize as a service provider as well to ensure the customer's service(s) are up and running. The aggregate feeds must be sent to a central monitoring service and customers should be able to get access to their own monitoring instance to customize as they see fit. We initially chose to go with Nagios but ultimately ended up choosing Sensu which provides a much more modern interface but also allows reuse of various Nagios plugins. This service that we provide to our customers goes beyond what a traditional PaaS like Heroku or even an IaaS provider like AWS provides out of the box.

Health checks are a critical feature that supports much of the platform's functionality: monitoring/alerting, seamless redeploys, etc. However, the problem with Sensu is that it does not expose the state of health checks in a convenient manner that can be leveraged by the rest of the platform for specific lower-level capabilities. We're exploring alternatives and better solutions to this specific problem. 

## Logging

<blockquote>
    <p>I carry a log - yes. Is it funny to you? It is not to me. Behind all things are reasons. Reasons can even explain the absurd. .... Watch - and see what life teaches.</p>
    <p>— The Log Lady, from the TV show Twin Peaks</p>
</blockquote>

Similar to monitoring but especially so, logging is a key requirement of any compliance regulation. At the same time, we need to minimize our access to our customers' application. To enable this, we needed to provide customers with their own dedicated logging environment. This is a significant value as other services like AWS CloudTrail, Papertrail, Splunk, etc. can get very expensive very quickly. We chose to go with the ELK (Elasticsearch, Logstash, Kibana) stack. To simplify access and sending of logs to this service, by default, any application log messages directed to `stdout` will be picked up and shipped over. Additionally, all services in a customer's environment are configured to ship their logs to that environment's logging service. This provides our customers with a single, dedicated place where all of their application, database, etc. logs are aggregated and searchable. Logs can get pretty large which is why we retain only 14 days of live logs (i.e. searchable). The remainder of the logs (along with their indexes) are rotated out into longer term encrypted storage like S3. 

In the next post in this series, we'll talk more about how we went about implementing these requirements and the lessons learned over the course of running this PaaS for over a year with many customers. 
