# wif-invoke-cr
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

# Java example!!
Based on the approach that worked with the Python SDKs, I got the same working with Java:

```
package wifexample;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.LambdaLogger;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.google.cloud.iam.credentials.v1.*;
import com.google.api.client.http.*;
import com.google.api.client.http.javanet.NetHttpTransport;

import java.io.IOException;
import java.util.*;

import java.util.Map;

public class Handler implements RequestHandler<Map<String,String>, String> {

	@Override
	public String handleRequest(Map<String, String> event, Context context) {
		LambdaLogger logger = context.getLogger();
		logger.log("creds:" + System.getenv("GOOGLE_APPLICATION_CREDENTIALS"));
		
	    logger.log("EVENT TYPE: " + event.getClass());
	    
	    try (IamCredentialsClient iamCredentialsClient = IamCredentialsClient.create()) {
	    	   ServiceAccountName name = ServiceAccountName.of("-", "test-from-aws@mjc-edge-to-mesh.iam.gserviceaccount.com");
	    	   List<String> delegates = new ArrayList<>();
	    	   String audience = "https://whereami-f7fvan6eqq-uc.a.run.app";
	    	   String crUrl = "https://whereami-f7fvan6eqq-uc.a.run.app";
	    	   boolean includeEmail = true;
	    	   GenerateIdTokenResponse response =
	    	       iamCredentialsClient.generateIdToken(name, delegates, audience, includeEmail);
	    	   String idToken = response.getToken();
	    	   logger.log("ID token: " + idToken);
	    	   
	    	   GenericUrl genericUrl = new GenericUrl(crUrl);
	    	   
	    	   HttpTransport netHttpTransport = new NetHttpTransport();
	    	   HttpRequestFactory requestFactory = netHttpTransport.createRequestFactory();
	    	   HttpHeaders authHeader = new HttpHeaders().setAuthorization("Bearer " + idToken);
	    	   HttpRequest request =
	    	     requestFactory
	    	       .buildGetRequest(genericUrl)
	    	       .setHeaders(authHeader);
	    	   HttpResponse resp = request.execute();
	    	   return resp.parseAsString();
	    	   
	    	 } catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
	    
	    return null;
	}

}

```
Note: I needed to add creds AND a file under resources. File `META-INF/services/io.grpc.LoadBalancerProvider` with value `io.grpc.internal.PickFirstLoadBalancerProvider`


