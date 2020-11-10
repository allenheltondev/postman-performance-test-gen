# Postman Performance Test Generator
An often overlooked part of API development is performance testing. However, getting a gauge of how quickly your API responds is a great measure for KPIs as well as an indicator of reliability to consumers. 

This repository was built to enable consumers to get out-of-the-box performance testing for APIs they have created in Postman. With minimal setup, the generator is designed to run as a [Postman monitor](https://learning.postman.com/docs/designing-and-developing-your-api/monitoring-your-api/intro-monitors/) that will run dynamic tests as you update your API definition document.

The test generator is a Postman [collection](https://learning.postman.com/docs/sending-requests/intro-to-collections/) and accompanying [environment](https://learning.postman.com/docs/sending-requests/managing-environments/) to enable quick configuration and a hands-off approach to performance testing.

# Objective
To build an automated way to provide performance testing of a set of APIs without having to create and maintain bulky test setups and configurations. 

It is also meant to encourage full use of an [Open API Specification](https://www.openapis.org/) to help establish best practices with API development. 

The generator tests the **implementation** of a given API -- not the definition. It uses the definition document to determine what the endpoints are and what data to send to each API.

# What It Does

## Execution
Every run, the generator will perform the following actions:

* Load the OAS for the provided APIs/workspace
* Validate the definition is in the proper format
    * All parameters and schemas have an `example` provided
    * `Servers` are defined    
* Build performance test array
* Loop through every test in the array a user-defined amount of times
* Collect response times and status codes for each API execution
* Evaluate response times for mean execution time, p99, and success rate.

## Performance Test Definition
For every path defined in the Open API Spec, a `json` object will be created in the following format to define the performance test:

```
{
  "path": "", // Combines url from server and path
  "parameters": [ ], // All parameters defined at the path
  "method": "", // Path method
  "allowedRole": "", // If using role based apps, grabs the first allowable role  
  "body": { }, // Generated body in JSON  
}
```

## Test Generation
The generator will build request bodies for each endpoint using the **example** provided for every schema property. If there is no example provided for an individual property, it will not be included in the request. Examples will be used for all parameters and schema properties.

For accurate tests, it is advised *to use real values from your test environment in your examples* in order to get valid responses back during test execution. This means using known, hardcoded or seeded values in the system to define your API in the OAS.

The request body will be generated with all fields that have an example provided. If your schema allows for multiple options, like the use of `oneOf` or `anyOf`, an example will be generated for each permutation and will be executed during the performance tests.


# What It Does NOT Do
The generator will **not** build a collection to be run at a later date. It builds and executes the tests at runtime, allowing for the test to dynamically update as changes are made to the Open API Spec.

# Requirements
The Open API Spec tested by the generator is required to be in a specific format. Below are the minimum requirements for the tests to execute:

* Each API must be created as a [Postman API](https://learning.postman.com/docs/designing-and-developing-your-api/the-api-workflow/) and defined in Open API Spec 3.0 format in either json or yaml
* The `Servers` object is defined and has at least one server with a description
* Every schema property must have an example defined
* The following fields in the provided environment are configured
    * `env-apiKey` - Integration API Key for Postman (string)
    * `env-server` - Matches the description of the server to be tested in the `Servers` object (string)
    * `env-performanceIterations` - How many times each path defined in the OAS should be executed (integer)
    * `env-performanceAverageMs` - The non-inclusive threshold in milliseconds for how fast each endpoint should execute (integer)
    * `env-performanceP99Ms` - The non-inclusive threshold in milliseconds for how fast the slowest execution allowed is per endpoint (integer)
* Request bodies are defined in the `#/components/schemas` section of the OAS and are referred to by using `$ref`
* Either `env-workspaceId` or `env-apiIds` should be defined.
    * If `env-apiIds` is defined, the generator will test all APIs with the provided ids (array)
    * If `env-workspaceId` is defined, the generator will test all APIs in the workspace provided (string)

# Optional Fields
There are a couple of optional fields in the provided environment that you can configure if your application uses role based authentication. Refer to the [security test generator](https://github.com/allenheltondev/postman-security-test-generator) for more details.

* `env-securityExtensionName` - If you define which role(s) are allowed per endpoint in your OAS, this is the name of the extension you use to define them
* `env-roleHeaderName` - If you pass in a role as a header in your requests, this is the name of the header

# Authentication
Authentication is the only piece of the generator that needs to be handled by the consumer. It is set up assuming all requests that execute against your API use the same authentication method.

## Configuration
If your API uses authentication, you will be required to configure it on the collection itself.

1. Right click the collection in your Postman workspace and select **Edit**.
2. Click on the **Authorization** tab and configure the type of Authentication your API requires.
3. If using OAuth2.0, you may [reference my blog post](https://www.readysetcloud.io/blog/allen.helton/how-to-automate-oauth2-token-renewal-in-postman-864420d381a0/) on how to automate the token renewal.
4. Click **Update** to save your changes

**NOTE-** If your API uses a standard api key header like `x-api-key` this step is unnecessary. You may just add it in the `#/components/parameters` section and it will be included automatically in each request.
```yaml
components:
  parameters:
    ApiKey:
      name: x-api-key
      in: header
      example: 982345jsdw0971ls09812354
      schema:
        type: string
```

# Setup
In this repo there are two files you need to import into your [Postman](https://postman.com/) workspace:
* Performance Test Generator.postman_collection.json - *Collection*
* Performance Generator Environment.postman_environment.json - *Environment*

If you are unsure how to import these into Postman, please [refer to this guide](https://kb.datamotion.com/?ht_kb=postman-instructions-for-exporting-and-importing).

# Running The Generator
You have the ability to run the generator collection manually or automatically as a monitor (recommended).

## Running Manually
1. Before you attempt to run the collection, make sure you fill out the require environment variables listed in the **Requirements** section above. 
2. From within the Postman app, click on the **Runner** button.
3. Select `Performance Test Generator` as the collection to run
4. Select `Performance Generator Environment` as the environment to run in
5. Leave Iterations at 1, delay at 0, and all checkboxes at their default state
6. Hit the `Run Performance Test Generator` button to begin execution
7. The generator will automatically test every endpoint defined in the Open API Specs provided in the environment
8. When execution is complete, it will run a series of assertions verifying:
    * The average execution time for each endpoint is below the defined threshold in `env-performanceAverageMs`
    * The slowest execution time for each endpoint is below the definede threshold in `env-performanceP99Ms`
    * At least half of the executions were successful (status code 2XX)

## Running Automatically
If you wish to *set and forget* your performance tests, then you will want to configure a [Postman monitor](https://learning.postman.com/docs/designing-and-developing-your-api/monitoring-your-api/intro-monitors/).

The monitor will run at a specified schedule and test all endpoints in the provided APIs, updating dynamically as the definition is updated.

# Contact
You may contact me by any of the social media channels below:

[![Twitter][1.1]][1] [![GitHub][2.1]][2] [![LinkedIn][3.1]][3] [![Ready, Set, Cloud!][4.1]][4]

[1.1]: http://i.imgur.com/tXSoThF.png
[2.1]: http://i.imgur.com/0o48UoR.png
[3.1]: http://i.imgur.com/lGwB1Hk.png
[4.1]: https://readysetcloud.s3.amazonaws.com/logo.png

[1]: http://www.twitter.com/allenheltondev
[2]: http://www.github.com/allenheltondev
[3]: https://www.linkedin.com/in/allen-helton-85aa9650/
[4]: https://readysetcloud.io
