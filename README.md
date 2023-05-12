# wif-invoke-cr-py
Generate token from AWS Lambda fn to invoke a CloudRun service via Workload Identity Federation

I was unable to invoke a CloudRun service via the Google Cloud client libraries using the documentation
mentioned [here](https://cloud.google.com/run/docs/authenticating/service-to-service#run-service-to-service-example-python).

I found a Python example (from step 7 [here](https://cloud.google.com/iam/docs/tutorial-cloud-run-workload-id-federation#get_the_jwt_for_the_authenticated_user_from_keycloak) ) - but it needed a little work (not too much).

# Prerequisites

- Follow the WIF docs to create your workload identity pool
- Create a Service Account with the required permissions - in this case Cloud Run Invoker
- From the Workload Identity pool click "Grant Access" and select the SA
- Download the config json file
- Upload the config.json as part of your Python package and set an environment variable of `GOOGLE_APPLICATION_CREDENTIALS` to point to it

# Create a Lambda Layer
I found it much easier to package up my Python dependencies and deploy them as a Lambda layer. In your function you then just select the layer you created

- From a Linux machine (not Mac / Windows) install dependencies: `python3.10 -m pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib google-cloud-iam  --target ./python`
- Zip up the package directory: `zip -r layer.zip python`
- Push the zip file up to your AWS account as a layer: `aws lambda publish-layer-version --layer-name google-layer --zip-file fileb://layer.zip --compatible-runtimes python3.10 --region us-east-1`


# Create your function using the following code
I used the Python 3.10 runtime. Don't forget to set the environment variable and select your (custom) layer you previously uploaded

Run & enjoy!

```

# This file holds a function for testing service-to-service authentication
from google.auth import credentials
from google.cloud import iam_credentials_v1

import google.auth
import google.oauth2.credentials

from google.auth.transport.requests import AuthorizedSession, Request
def lambda_handler(event, context):
    url = "{YOUR CLOUD RUN URL}"
    aud = "{YOUR CLOUD RUN AUDIENCE URL}"
    service_account = '{YOUR SERVICE ACCOUNT EMAIL}'

    client = iam_credentials_v1.services.iam_credentials.IAMCredentialsClient()
    name = "projects/-/serviceAccounts/{}".format(service_account)
    id_token = client.generate_id_token(name=name,audience=aud, include_email=True)

    print(id_token.token)

    creds = google.oauth2.credentials.Credentials(id_token.token)
    authed_session = AuthorizedSession(creds)
    r = authed_session.get(url)
    print(r.status_code)
    print(r.text)
```

![image](https://github.com/mark-j-chilvers/wif-invoke-cr-py/assets/45714243/0a673014-90e9-40f4-b038-2b64f89f186c)
