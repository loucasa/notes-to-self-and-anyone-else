---
# notes-to-self (and anyone else)

A place to collect my knowledge journey writing code and developing applications

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

    - Curl allows you to quckly check if headers are present as a good first step but does not fully similate a browser where preflight check is involved and also the origin header set by the browser:

       `curl -i -X OPTIONS -H "Origin: http://foo.com" -H 'Access-Control-Request-Method: GET' https://my.api/path`
    - Online tools like this [cors-tester](https://cors-error.dev/cors-tester/) really helped too
### Lessons
- I found it necessary to include the `Origin` and `Access-Control-Request-Method` to get the AWS Gateway to return the headers
- I had to enable Cors on the API Gateway with '*' for each header value except for `Access-Control-Allow-Methods` which only seemed to work if I explicitly select all the methods

## Python Requirements
A useful command to capture versions of packages when you have not specified version in the requirements.txt:
`pip freeze -q -r requirements.txt | sed '/freeze/,$ d'`
