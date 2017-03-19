# CatLight Protocol 1.0

CatLight protocol is an open standard for modern continuous delivery servers. 

[CatLight status notifier](https://catlight.io) can connect to servers that support this protocol and show desktop build notifications.

Continuous delivery servers can implement the basic part of this protocol by just placing a static file on their server that is updated on each build.

This protocol supports authentication, caching and dynamic loading for optimal performance of large servers.

# Data model

Most build server use the similar concepts, but they name them differently. CatLight terms are closely based on TFS.

CatLight protocol uses the following build server model:

* Server
  * Spaces
    * BuildDefinitions
      * Branches
        * Builds

#### Mapping between different build servers

| CatLight | TFS  | Jenkins | Travis CI | 
| --- |---|--- | ---
| Server | Project Collection | Server | Service (OS/Pro) |
| Space | Project | - | Github Account |
| Build Definition | Build Definition | Job | Repository |
| Build | Build | Build | Job |


## Entities

### Server

Server is a top-level object

| Property | Type | Description |
| --- | --- | --- |
| protocol | string | Constant string that defines the version and mode of CatLight protocol.
| id | string | Globally unique server id. 
| name | string | Display name of the server that is shown to the user.
| webUrl | string | (Optional) Url of the main server page.
| serverVersion | string | (Optional)
| currentUser | User | (Optional) Currently logged-in user. If it is absent, build investigation feature will not be available.
| spaces | Space[] | Array of spaces available to current user on this server.


Server id must be globally unique and should not change.

How to choose server id?
* For online services, you can use your domain name, e.g. "myserver.com"
* For installed servers, use your domain or product name + unique installation id. For example, "myserver.com/3A5126E7-EE74-48E4-9997-1C1DE52BA711". If your server does not have unique id, you can generate a GUID during installation and save it to configuration file. 


Try to keep the id shorter than 100 symbols. If your server has a built-in unique id that is very long, you can use a hash sum of it that should be shorter.

Protocol value can be one of the following:
* https://catlight.io/protocol/v1.0/basic
* https://catlight.io/protocol/v1.0/dynamic

Read more about basic and dynamic modes below.

### Space

Space is a logical container of build definitions. For most servers, space will map to a project or source control repository. 

CatLight apps exchange information about build investigations on space level. If your server has thousands of build definitions, put them into separate spaces to avoid performance problems.


| Property | Type | Description |
| --- | --- | --- |
| id | string | Space id should be unique on the server. 
| name | string | Display name
| webUrl | string | (Optional)
| buildDefinitions | BuildDefinition[] |



### BuildDefinition

| Property | Type | Description |
| --- | --- | --- |
| id | string | Id should be unique in parent space. 
| name | string | Display name
| webUrl | string | (Optional) Webpage that provides user with detailed information about this build definition.
| folder | string | (Optional) path to the build definition, e.g. "my folder/sub-folder".
| branches | BuildBranch[] | 


Branches array should include:
  * Primary branches (master, develop, etc.)
  * Active feature branches that had builds in the past 30 days.

If your system cannot separate builds into branches, add a single branch to this array with id = "~all"


### BuildBranch

| Property | Type | Description |
| --- | --- | --- |
| id | string | Id should be unique within the parent build definition.
| webUrl | string | (Optional)
| builds | Build[] | Sorted array of builds. Last build in the array should be the newest. Return 5-10 builds for each branch. 

Branch id can be taken from source control. Examples: 
* refs/heads/features/test-feature-branch
* refs/heads/master
* refs/pull/2/merge
* refs/remotes/origin/features/test-feature-branch

CatLight will cleanup branch id and use it as a display name for the branch. For example, "refs/heads/master" will be transformed into "master".

Note: if you don't see the branch in CatLight app UI, make sure that it has recent builds.

### Build

| Property | Type | Description |
| --- | --- | --- |
| id | string | Id should be unique within the parent build definition. 
| name | string | (Optional)
| webUrl | string | (Optional) Web page with build details 
| status | string | Build status, see below for the list of possible values
| startTime | DateTime | Date and time when this build was started.
| finishTime | DateTime | Should be present for completed builds, but can be absent for running builds.
| triggeredByUser | User | (Optional) User that requested the build. If the build started automatically after new commit, this should be the user that committed the code.
| contributors | User[] | When the build contains changes from multiple user, additional contributors can be placed here.

CatLight will use "triggeredByUser" and "contributors" field to determine if this build is related to current user.

Build status can be one of the following:
* Queued
* Running
* Succeeded
* PartiallySucceeded
* Failed
* Canceled

### User

| Property | Type | Description |
| --- | --- | --- |
| id | string | Id should be unique within the server 
| name | string | Display name

### DateTime

DateTime is represented in [ISO date format](https://docs.microsoft.com/en-us/scripting/javascript/date-and-time-strings-javascript#ISO). 

`YYYY-MM-DDTHH:mm:ss.sssZ`

Example: `2017-01-25T17:30:10.000Z`

Z at the end means that the time is in UTC time zone.




# Implementation

CatLight protocol support two modes - basic and dynamic. 
Your server should implement one of them.

In basic mode, server returns all the build information on each polling cycle. It is easy to implement, but the amount of generated and returned data can be quite large.

In dynamic mode, CatLight requests the status of the specific build definitions from the server, and server returns only them.

Both modes use the same data model.


## Basic mode


In basic mode, CatLight makes the same HTTP GET request on each polling cycle (1 minute by default).

Server should return all build information available to the current user.

You can create an endpoint like http://myserver.com/catlight . It should return server object from the data model in JSON format.

Basic mode can be implemented by creating a static file on a build server, and updating this file on each change.

Example:
```
{
    "protocol": "https://catlight.io/protocol/v1.0/basic",
    "id": "myAwesomeServer/12345678-1234-4567-abcd-123456789abc",
    "webUrl": "http://myserver.example/dashboard",
    "name": "My Server",
    "currentUser": {
        "id": "tim95",
        "name": "Tim Drake"
    },
    "spaces": [
        {
            "id": "super-project",
            "name": "Super Project",
            "webUrl": "http://myserver.example/super-project",

            "buildDefinitions": [
                {
                    "id": "nightly-build",
                    "name": "Nightly Integration Build",
                    "webUrl": "http://myserver.example/super-project/nightly-build/view",
                    "folder": "build folder/subfolder",
                    "branches": [
                        {
                            "id": "develop",
                            "builds": [
                                {
                                    "id": "100",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/100",
                                    "status": "Succeeded",
                                    "startTime": "2017-01-25T17:30:10.000Z",
                                    "finishTime": "2017-01-25T17:30:20.000Z",
                                    "triggeredByUser": {
                                        "id": "tim95",
                                        "name": "Tim Drake"
                                    },
                                    "contributors": [
                                        {
                                            "id": "jgordon",
                                            "name": "James Gordon"
                                        }
                                    ]
                                }, {
                                    "id": "101",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/101",
                                    "status": "Running",
                                    "startTime": "2017-01-25T17:40:10.000Z",
                                    "triggeredByUser": {
                                        "id": "jgordon",
                                        "name": "James Gordon"
                                    }
                                }
                            ]

                        }, {
                            "id": "features/new-searchlight",
                            "builds": [
                                {
                                    "id": "300",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/300",
                                    "status": "Succeeded",
                                    "startTime": "2017-01-25T16:30:10.000Z",
                                    "finishTime": "2017-01-25T16:30:20.000Z",
                                    "triggeredByUser": {
                                        "id": "tim95",
                                        "name": "Tim Drake"
                                    }
                                }
                            ]

                        }
                    ]
                }
            ]
        }
    ]
}
```

You can try the following URLs with the CatLight app to see how they are processed:

* https://catlight.io/catlight-protocol-examples/basic/multi-space.json
* https://catlight.io/catlight-protocol-examples/basic/basic.json

## Dynamic mode

Dynamic mode can reduce the load on the server in situations when user is interested in only a small subset of available build definitions.

Workflow in dynamic mode:

1. Metadata request: CatLight gets metadata from the server by making HTTP GET request to the endpoint like http://myserver.com/catlight
2. User selects build definitions that he wants to monitor. 
3. State request: on each polling cycle, CatLight makes HTTP POST request to the same endpoint, and provides build definitions in request body. Server should return details only about requested build definitions.

### Metadata request

Server can return less fields for metadata request. Metadata response should contain the following:

| Entity | Required fields | Optional fields |
| --- | --- | --- |
| Server | protocol, id, name, spaces | webUrl, currentUser, serverVersion  |
| Space |  id, name, buildDefinitions | webUrl |
| BuildDefinition | id, name | webUrl, folder |
| User | id, name |  |


### State request

CatLight will send HTTP POST request to the same endpoint. This request will have a JSON body containing server entity with the following fields:

| Entity | fields |
| --- | --- |
| Server | id, spaces | 
| Space |  id,  buildDefinitions | 
| BuildDefinition | id |  

Example:
```
{
    "id": "myAwesomeServer/12345678-1234-4567-abcd-123456789abc",
    "spaces": [
        {
            "id": "super-project",
            "buildDefinitions": [
                {
                    "id": "nightly-build"
                }
            ]
        }
    ]
}
```

State response should include only requested build definitions and it should contain the following fields:

| Entity | Required fields | Optional fields |
| --- | --- | --- |
| Server | protocol, id, spaces | |
| Space |  id,  buildDefinitions | |
| BuildDefinition | id, branches |  |
| BuildBranch | id, builds | webUrl |
| Build | id, status | name, webUrl, startTime, finishTime, triggeredByUser, contributors |
| User | id, name |  |

Example:

```
{
    "protocol": "https://catlight.io/protocol/v1.0/dynamic",
    "id": "myAwesomeServer/12345678-1234-4567-abcd-123456789abc",
    "spaces": [
        {
            "id": "super-project",
            "buildDefinitions": [
                {
                    "id": "nightly-build",
                    "branches": [
                        {
                            "id": "develop",
                            "builds": [
                                {
                                    "id": "100",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/100",
                                    "status": "Succeeded",
                                    "startTime": "2017-01-25T17:30:10.000Z",
                                    "finishTime": "2017-01-25T17:30:20.000Z",
                                    "triggeredByUser": {
                                        "id": "tim95",
                                        "name": "Tim Drake"
                                    },
                                    "contributors": [
                                        {
                                            "id": "jgordon",
                                            "name": "James Gordon"
                                        }
                                    ]
                                }, {
                                    "id": "101",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/101",
                                    "status": "Running",
                                    "startTime": "2017-01-25T17:40:10.000Z",
                                    "triggeredByUser": {
                                        "id": "jgordon",
                                        "name": "James Gordon"
                                    }
                                }
                            ]

                        }, {
                            "id": "features/new-searchlight",
                            "builds": [
                                {
                                    "id": "300",
                                    "webUrl": "http://myserver.example/super-project/nightly-build/300",
                                    "status": "Succeeded",
                                    "startTime": "2017-01-25T16:30:10.000Z",
                                    "finishTime": "2017-01-25T16:30:20.000Z",
                                    "triggeredByUser": {
                                        "id": "tim95",
                                        "name": "Tim Drake"
                                    }
                                }
                            ]

                        }
                    ]
                }
            ]
        }
    ]
}
```

## Caching


Since builds usually do not change every minute, most of the requests will return the same data. Server can save processing power and bandwidth by implementing HTTP response caching.

Response caching is supported in basic and dynamic modes. This is an optional feature, and servers don't have to implement it.


Workflow:
1. Server returns ETag HTTP header with the response.
2. CatLight sends If-None-Match header with the next request, providing the value from ETag header.
3. If state has not changed, server returns 304 HTTP status code, and does not return the response body.

Read more about [HTTP response caching and headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag)

## Authentication

If your server requires user authentication, CatLight can authenticate on HTTP level using one of the following methods:
* Username and password (HTTP Basic authentication)
* Token (HTTP Bearer authentication)


## Support

If you have questions or comments about this protocol, please use our [support site](http://catlight.helprace.com/) or contact us directly at [support@catlight.io](mailto:support@catlight.io)
