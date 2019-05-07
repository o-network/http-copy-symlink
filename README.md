# Open Network HTTP COPY Symlink Module for FS Store

The intention of this module is to provide support for the `COPY` command to help achieve compatibility to the [Solid](https://solid.mit.edu/) project. 

To allow for clean user experience, we sometimes require very quick operations to allow for instant user interaction, one type of application implementation that will be seen built on top of Solid is to define state machines by utilising the powerful graphing ability of [RDF](https://github.com/solid/solid-spec/blob/master/content-representation.md) and [FaaS](https://www.openfaas.com/). 

To achieve a quick response time we will need a way to offload `COPY` requests to a backing service, but we also need to be able to make the resources available from our service. 

There are different levels of acceptable outcomes for a `COPY` request, here's a quick list of use cases we will cover:

- Agent makes a `COPY` request with header `Source` a resource URI, agent requires the resource to be cloned immediately, and for a shallow copy to be made (this means they do not want us to follow linked resources at all), service is expected to return status code `201`, with the `Location` header the URI of the cloned resource.
- Agent makes a `COPY` request with header `Source` a resource URI, agent requires the resource to be available immediately, and for a full copy to be made for the origin (this means they do want us to follow linked resources from the same origin), service is expected to return status code `201`, with the `Location` header the URI of the symbolic resource. Service must follow an agent accessible pre-defined cloning scheme.

Above, we have defined an entry point for a new set of rules, what a _cloning scheme_ is. We want to seperate this concern as agents that have different requirements will require different ways to clone information. 

An example of a cloning scheme is:

`COPY`:

1. Service will make a `HEAD` request to `Source`
2. If the `HEAD` status code is not between `200` and `299`, and `301` to `308`, then return `400` with header `Warning: Resource not available - ${headResponse.status}`
3. Service will make a `COPY` request to backing service for `Source`
	1. Backing service will create new random `Pointer URI`
	2. Backing service will return status `201` with header `Location: ${Pointer URI}`
	3. Backing service will set resource at `${Pointer URI}.meta` to `Downloading`
	4. Backing service will make a `GET` request to `Source`
	5. If `GET` return status is not between `200` and `299`, and `301` to `308`, then set resource at `${Pointer URI}.meta` to `Warning: ${getResponse.text()}` and abort.
	6. Backing service will store download resource at `Pointer URI` once downloaded
	7. Backing service will set resource at `${Pointer URI}.meta` to `Downloaded`
4. Create resource at `${Target URI}.meta` describing the location of the new resource at `Pointer URI`
5. Return `201` with header `Location: ${Target URI}`.

`GET`:

1. Service will check if `${Target URI}.meta` describes the location of a resource, if not continue as normal
2. Service will make a `GET` request to `${Pointer URI}.meta` to see the status of the resource
3. If `GET` response is `Downloading` or `Downloaded`, then return status `302` with `Location: ${Pointer URI}`
4. If `GET` response is `Warning`, then return status `404` with header `Warning: ${getResponse.header.get("Warning")`
	1. Backing service will schedule `${Pointer URI}.meta` to be deleted in `n` days 
5. If `GET` response is not `Downloaded`, return
6. Service will make a `COPY` request to `${Target URI}` with header `Source: ${Pointer URI}`, service requires the resource to be cloned immediately
7. Service will delete the location described in `${Target URI}.meta`
 

