# security-kickstart
A demo of various Apigee security capabilities such as shared flows, oauth, rate limiting, json payload protection, sql injection protection, and SOAP payload injection

## Introduction
Apigee provides a plethora of out-of-the-box security solution. In this repository you will personally test out a few of them in the format of a reproducable demo.

## Assumptions
1. You already have an Apigee X org provisionined
2. You have [Postman](https://www.postman.com/) downloaded and ready to use. Postman is a free service that we will use to call our API endpoints

## Intial Setup
1. Download OAuthQuota.zip from this repository to your computer. Then [upload it to Apigee as a shared flow bundle](https://cloud.google.com/apigee/docs/api-platform/fundamentals/shared-flows#creating-a-shared-flow) to create a new shared flow called "OAuthQuota" 
2. Download oauth.zip from this repository to your computer. Then [upload it to Apigee as a proxy bundle](https://cloud.google.com/apigee/docs/api-platform/fundamentals/download-api-proxies#upload) to create a new proxy called "oauth"
3. Download secured.zip from this repository to your computer. Then [upload it to Apigee as a proxy bundle](https://cloud.google.com/apigee/docs/api-platform/fundamentals/download-api-proxies#upload) to create a new proxy called "secured"
4. Download SecuredAPIDemo.postman_collection.json to your computer. Then [import it to Postman as a new collection](https://learning.postman.com/docs/getting-started/importing-and-exporting-data/#importing-data-into-postman) to create a new collection called "Secured API Demo"
5. From Postman, navigate to your newly imported collection's variables as shown in the below image. Update the current value for api_domain with the domain retreived from Apigee at Admin > Environments > Groups, update the value for client_id from your Apigee App, and update the value for client_secret from your Apigee App. Don't forget to save with CTRL+S. We'll update the oauth_token value in our demo.
![Collection Variables](/assets/collectionVariables.png)
6. From the Apigee console, [create a new developer](https://cloud.google.com/apigee/docs/api-platform/publish/adding-developers-your-api-product#add) with a name and email of your choosing
7. From the Apigee console, [create a new product](https://cloud.google.com/apigee/docs/api-platform/publish/create-api-products#add) called secured-product. Configure it with public access, a quota of 5 requests every 1 minute, and add an operation for the secured proxy with a path of "/". Don't forget to click save.
8. From the Apigee console, [create a new app](https://cloud.google.com/apigee/docs/api-platform/publish/creating-apps-surface-your-api#register) called secured-app. Configure it with your newly created developer, no callback URL, add your secured-product under credentials, approve the secured-product, and set it to never expire. Don't forget to save. After saving, make note of the client id and secret.
9. Navigate to the following links, update the request with our organization, and click the Execute button to create an Apigee Key Value Map (KVM) and populate it with mock username and password credentials. These credentials will be used for our AssignMessage SOAP policy.
- [Create KVM](https://cloud.google.com/apigee/docs/reference/apis/apigee/rest/v1/organizations.environments.keyvaluemaps/create?apix_params=%7B%22parent%22%3A%22organizations%2FYOUR_APIGEE_ORGANIZATION%2Fenvironments%2Feval%22%2C%22resource%22%3A%7B%22encrypted%22%3Atrue%2C%22name%22%3A%22SOAP-Creds%22%7D%7D)
- [Create KVM username entry](https://cloud.google.com/apigee/docs/reference/apis/apigee/rest/v1/organizations.environments.keyvaluemaps.entries/create?apix_params=%7B%22parent%22%3A%22organizations%2FYOUR_APIGEE_ORGANIZATION%2Fenvironments%2Feval%2Fkeyvaluemaps%2FSOAP-Creds%22%2C%22resource%22%3A%7B%22name%22%3A%22soapUsername%22%2C%22value%22%3A%22exampleUsername%22%7D%7D)
- [Create KVM password entry](https://cloud.google.com/apigee/docs/reference/apis/apigee/rest/v1/organizations.environments.keyvaluemaps.entries/create?apix_params=%7B%22parent%22%3A%22organizations%2FYOUR_APIGEE_ORGANIZATION%2Fenvironments%2Feval%2Fkeyvaluemaps%2FSOAP-Creds%22%2C%22resource%22%3A%7B%22name%22%3A%22soapPassword%22%2C%22value%22%3A%22examplePassword%22%7D%7D)


## Demonstrate
We'll be using Postman to call our API and test out our security features.

### A Few Notes First
There are a few things happening in the demo that aren't explicitely mentioned in the demo:
- Our secured proxy is configured to send requests to https://httpbin.org/anything. That endpoint will simply return the plain text of the request itself. This is helpful for this demo so we can quickly and easily understand exactly what our request looks like when we pass it off to the backend.
- In the shared flow we are not only enforcing OAuth security and our quota, but we also subtly remove the client token from the request so it doesn't unecessarily appear anywhere downstream in the request
- In the secured proxy target endpoint request preflow, we add a [cors policy](https://cloud.google.com/apigee/docs/api-platform/reference/policies/cors-policy) to allow requests from any source. In a production environment, we'd want to update this policy to only allow requests from desired sources and in desired formats
- Our api proxies are all prepended with a "/v1" path. This is best practice for production APIs as you'll want to prepare for the possibility of future versions
- As you demo in Postman, feel free to flip back and forth between Apigee as well to understand the policies your requests are hitting

### Postman Demo
1. Get our OAuth token: From the Secured API Demo Postman collection, click on the Retreive Token POST call. Send this request to exchange your client_id and client_secret values for a newly minted OAuth token. Note the OAuth token from the response. From Postman, navigate to your newly imported collection's variables as done previously for the domain, client_id, and client_secret variables. Assign your OAuth token against the oauth_token variable. All future calls with be authenticated with this OAuth token in the header.
2. Verify the OAuth token: From the Secured API Demo Postman collection, click on the Use Token GET call. Send the request and note the response. You should receive a 200 success message, verifying that your OAuth token is valid and being used properly.
3. Test JSON Payload Validation: Our API proxy is configured to only allow JSON payload with 5 or less top-level properties within it. To demonstrate navigate click on the Use Token JSON Good POST request and click send. Note the request's body before sending a request and receiving a 200 response. Next, navigate on the Use Token JSON Bad POST request and click send. Note the request's body before sending it and receiving an error message.
4. Test SQL Injection: Our API proxy is configured to check the value for any URL parameters under the name of studentId for attempts of SQL injection. To demonstrate this navigate to the Use Token SQL Good GET request and click send. Note the request's url parameter before sending a request and receiving a 200 response. Next, navigate to the Use Token SQL Bad GET request and click send. Note the request's url parameters before sending it and receiving an error message
5. Test SOAP Security: The endpoint "/soap" within our API proxy is configured to augment the incoming request to enable it to authenticate with a legacy SOAP backend.To demonstrate this navigate to the Use Token SOAP request and click send. Note that the request's lack of a payload body and the body returned in the response. This will demonstrate how our proxy added SOAP auth security to our request. Also worth pointing out in the demo that the username and password are securely stored in KVM.