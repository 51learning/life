
Let's look at the effect first, and then complete the configuration step by step.

### Configurate APIG + CCE workloads

Create the APIG step by step like this:

![buyapig](./images/buyapig.png)

click `Gateway Information--> Basic Information --> Routes`  , Edit and add the container CIDR.

import CCE workload 

![apig import workload](./images/apig-import-workload.png)

The backend configuration of APIG must be consistent with that of CCE workload.

![apig backend](./images/apig-backend-configuration.png)

If we want to access the publish APIG through the public network, this is need bind `Inbond Access` EIP. 

If need CCE workload access the outside, this is need open the outbound access permission. 

![apig backend](./images/apig-network.png)

### APIG policy

You can select the corresponding plug-in to implement different functions. for example: Rate limit, IP block

![apig policies](./images/apig-policy.png)

**configurate authenticate**

create a function and integrate with APIG 

![apig policies](./images/apig-function-authentication.png)

the python demo code like this：

```python
# -*- coding:utf-8 -*-
import json


def handler(event, context):
    if event["headers"].get("authorization") == 'Basic dXNlcjE6cGFzc3dvcmQ=':
        return {
            'statusCode': 200,
            'body': json.dumps({
                "status": "allow",
                "context": {
                    "user_name": "user1"
                }
            })
        }
    else:
        return {
            'statusCode': 200,
            'body': json.dumps({
                "status": "deny",
                "context": {
                    "code": "1001",
                    "message": "incorrect username or password"
                }
            })
        }
```

Follow this procedure to create `API Policies --> Custom Authorizers` 

![apig custom authorizser](./images/apig-create-custom-authorizer.png)

let's debug the apis, if didn use authorize, you will recive this response:


```bash
HTTP/1.1 401 Unauthorized
Transfer-Encoding: chunked
Connection: keep-alive
Content-Type: application/json
Date: Mon, 04 Sep 2023 11:57:37 GMT
Server: api-gateway
X-Apig-Latency: 557
X-Request-Id: 822cf60eb35d5280fc83614b31e7b507

{"error_msg":"Incorrect authentication information: frontend authorizer","error_code":"APIG.0305","request_id":"822cf60eb35d5280fc83614b31e7b507"}
```

Add the authorzation header and requeste again

![apig authorization](./images/apig-authorisation.png)

