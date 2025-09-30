---
# notes-to-self (and anyone else)

A place to collect my knowledge journey writing code and developing applications

- [Python Mocks](#python-mocks)
- [SSH Agent Forwarding](#ssh-agent-forwarding)
- [Handling Cors](#handling-cors)
  * [Checks](#checks)
  * [Testing](#testing)
  * [Lessons](#lessons)
- [Python Requirements](#python-requirements)
- [Lambda API Versioning](#aws-api-gateway-stage-lambda-proxy-versioning)

## Python Mocks
When attempting to `@patch` using unittest mocks to define the target for a chain of calls that may also occur across multiple lines e.g.
```
    ...
    response = more.modules.some_function(param='value', other_param='other_value')
    value_object = response.some_list[0].some_value
    outputs = []
    if value_object.value:
        parsed_value = value_object.value.value_dict()
        return parsed_value.get('outputs', [])
    else:
        return value_object.other_value
    ...
```
Define the target call behaviours:
```
@patch(target="some.module.Type")
def test_ml_inference(the_mock):
    the_mock.more.modules.some_function.return_value.some_list[0].some_value.value.value_dict.return_value = {
        'outputs': [{'index': 1, 'real_value': 'the_value'}]
    }
    the_mock.more.modules.some_function.return_value.some_list[0].some_value.other_value = 'mocked value'
```

## SSH Agent Forwarding
When needing to access git repositories on remote hosts, forwarding prevents the need to copy over a private key each time, just `ssh-add ~/.ssh/the_private_key_file` and adding to `.ssh/config`:
```
Host remost_host_ip_address_or_hostname
    ForwardAgent yes
```

## Handling Cors
Check each layer to identify where the problem lies, in this case I had an AWS Lambda, API Gateway and a static html website invoking the lambda via the gateway. Although it may be enough just to check at least the top layer I checked each layer returns the `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods` and `Access-Control-Allow-Methods` headers.

, the API Gateway was also but the browser wasn't getting them (I could easily confirm using )
### Checks
- Invoking the AWS lambda function directly via the AWS console, to simulate the http request I could use the following json:
  ```json
  {"httpMethod": "GET", "queryStringParameters": {"param_name":"param_value"}}
  ```
- Invoking the API Gateway to ensure the API endpoint responds with all headers to OPTIONS requests and at least with `Access-Control-Allow-Origin` to the the actual API request and with a 200 HTTP response status code.

### Testing
- Curl allows you to quckly check if headers are present as a good first step but does not fully similate a browser where preflight check is involved and also the origin header set by the browser:
`curl -i -X OPTIONS -H "Origin: http://foo.com" -H 'Access-Control-Request-Method: GET' https://my.api/path`
- Online tools like this [cors-tester](https://cors-error.dev/cors-tester/) really helped too
- A useful command to start a python local file system web server for testing:
`python -m http.server`
 
### Lessons
- I found it necessary to include the `Origin` and `Access-Control-Request-Method` to get the AWS Gateway to return the headers
- I had to enable Cors on the API Gateway with '*' for each header value except for `Access-Control-Allow-Methods` which only seemed to work if I explicitly select all the methods

## Python Requirements
A useful command to capture versions of packages when you have not specified version in the requirements.txt:
`pip freeze -q -r requirements.txt | sed '/freeze/,$ d'`

## AWS API Gateway Stage Lambda Proxy Versioning
There seems to be a few guides covering this (like [this aws blog](https://aws.amazon.com/blogs/compute/using-api-gateway-stage-variables-to-manage-lambda-functions/)) but I wanted to document the steps here just because it tripped me up when I missed some of the important details. I have a single lambda that is executed via a HTTP API gateway and I wanted to setup independent dev and prod environments so that I can make changes and test them in dev before releasing to prod, here are the steps:

1. Create 2 Lambda aliases (these names will be the value of stage variables used later):
   * dev: Pointing to the $LATEST version.
   * prod: Pointing to a specific, stable version (you can use the current latest version for now).
1. Configure the API Gateway integration on the route:
   * Integration type: `Lambda function`
   * Integration target:
     * select the lambda from the list to get the arn populated if you don't know it
     * change the value of `Lambda function` to the following (replacing `arn_of_the_lambda`): `arn_of_the_lambda:${stageVariables.lambdaAlias}`
     * after saving take note of the `source-arn` in the `Example policy statement` in the integration details (it will be used in the lambda permission)
1. Configure 2 API Gateway stages:
   * dev: with stage variable `lambdaAlias=dev`
   * prod: with stage variable `lambdaAlias=prod`
1. Configure permissions on each lambda alias, you should not need any permissions set on the main funtion, only the aliases
   * Select `AWS Service`
   * Set `Service` to `API Gateway`
   * Set `Statement ID` to anything unique
   * Leave `Principle` as it is
   * Set `Source ARN` to value noted earlier when setting up the gateway integration
     * it should have the following format: "arn:aws:execute-api:`aws_region`:`aws_account`:`api_id`/\*/\*/`api_route`"
     * you can decide how strict to make the trailing path, it defines the `/stage/method/resource` elements so `/dev/*/my-api`, allows any method only on the dev stage on the my-api resource
   * Set `Action` to `lambda:InvokeFunction`
   *  Watch out for 500 status codes from the gateways when the permssions are incorrect. Example policy generated:
```json
{
  "Version": "2012-10-17",
  "Id": "default",
  "Statement": [
    {
      "Sid": "my_api-lambda-alias-policy",
      "Effect": "Allow",
      "Principal": {
        "Service": "apigateway.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789999:function:my-lambda-function:dev",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:execute-api:us-east-1:123456789999:1abcde1a2b/*/*/my-api"
        }
      }
    }
  ]
}
```
