# Transformation : JavaScript and JSON-to-XML 

*Duration : 30 mins*

*Persona : API Team*

# Use case

You have a backend service that returns a JSON payload, but the schema doesn't match what you've published to consumers. Also, a potential business partner has asked that you provide access to the same data in XML format. You don’t own the backend service, and even if you did, by the time your team changed the backend, and it passed all your enterprise integration tests, you would have lost the partnership opportunity.

| Backend Service | Consumer-facing API |
| :---: | :---: |
| ![image alt text](./media/backend-service-spec.png) | ![image alt text](./media/new-api-spec-annotated.png) |

# How can Apigee Edge help?

The [JavaScript policy](https://docs.apigee.com/api-platform/reference/policies/javascript-policy) in Apigee lets you add custom JavaScript code that executes within the context of an API call. In your custom JavaScript code, you can use the objects, methods, and properties of the Apigee object model, which lets you to read or manipulate virtually any aspect of the HTTP request (from the client) or response (from the backend service), along with a wide breadth of rich contextual information that Apigee makes available. It is especially powerful for making tweaks to the format of a JSON (JavaScript Object Notation) payload.

The [JSON to XML Policy](http://docs.apigee.com/api-services/reference/json-xml-policy) in Apigee converts request or response messages from JSON to XML.  The default configuration should work in most cases but the policy gives you fine-grained control over all of the conversion features, such as namespaces, attributes, array elements, null values, separators, and how to handle invalid characters.

In this lab, you will modify an existing API Proxy to 1) reformat the default JSON response format from the backend service, and 2) allow the API to return XML to clients who request it.  To achieve this you will perform the following -

1. Adjust the response JSON using a few simple JavaScript lines.

2. If a request has an Accept header of "application/xml"

    1. Remove the (XML) Accept header to allow the target to return JSON

    2. Implement the [JSON to XML Policy](http://docs.apigee.com/api-services/reference/json-xml-policy) to convert the response body from JSON to XML

    3. Inject a response header of "Content-type: application/xml".

# Pre-requisites

Ensure you have an Apigee account: (https://apigee.com/edge)

# Instructions

## Create an Open API Specification

During the course of this lab, the sample HTTP service we will expose as an API endpoint, is the Hipster Products service located at [http://cloud.hipster.s.apigee.com/products](http://cloud.hipster.s.apigee.com/products).
First, we are going to design and create an OpenAPI specification for the different resource endpoints, i.e. /products and /products/{productId}. 

1. Go to [https://apigee.com/edge](https://apigee.com/edge) and log in. This is the Edge management UI. 

2. Select **Develop → Specs** in the side navigation menu

![image alt text](https://github.com/apigee/apijam/raw/master/Module-1/Labs/Lab%201/media/image_0.png)

3. As we have a pre-designed sample of the spec available for this lab, we will be importing it into your Apigee Org's Spec Store. Click **+Spec**. Click on **Import URL** to add a new spec from existing source.

![image alt text](https://github.com/apigee/apijam/raw/master/Module-1/Labs/Lab%201/media/image_1.png)

4. Enter spec details.

   * Import Name: Products API
   * Import Uul: https://raw.githubusercontent.com/krisrowe/apigee-labs/master/Transformation/products-api-spec.yaml

![image alt text](./media/ImportSpecByURL.png)

5. Verify the values and click **Import**. Spec has been imported into Apigee and is ready to use. You should see your spec in the list. For example,

![image alt text](./media/SpecImported.png)

6. Click on Products API spec from the list to access Open API spec editor and interactive documentation that lists API details & API Resources.

![image alt text](./media/ProductsAPISpec.png)

## Create an API Proxy

1. It’s time to create Apigee API Proxy from Open API Specification. Click on **Develop → API Proxies** from side navigation menu.

![image alt text](https://github.com/apigee/apijam/raw/master/Module-1/Labs/Lab%201/media/image_5.png)

2. Click **+Proxy** The Build a Proxy wizard is invoked. 

![image alt text](https://github.com/apigee/apijam/raw/master/Module-1/Labs/Lab%201/media/image_6.png)

3. Select **Reverse proxy**, Click on **Use OpenAPI** below reverse proxy option.

![image alt text](https://github.com/apigee/apijam/raw/master/Module-1/Labs/Lab%201/media/image_7.png)

4. You should see a popup with list of specs. Select the spec you previously imported. 

![image alt text](./media/SelectSpecForProxy.png)

5. Enter details in the proxy wizard. Replace **{your-initials}** with the initials of your name. 

    * Proxy Name: Products

    * Proxy Base Path: /v1/products

    * Existing API: Observe the field value which is auto filled from OpenAPI Spec. Add "/products" to the end as shown below.

![image alt text](./media/EnterProxyRoutingDetails.png)

6. Verify the values and click **Next**.

7. Select **Pass through (none)** for the authorization in order to choose not to apply any security policy for the proxy. Click Next. 

![image alt text](./media/SelectPassThrough.png)

8. You can select, de-select list of API Proxy Resources that are pre-filled from OpenAPI Spec. Select all and click on **Next**

![image alt text](./media/SelectResourcesForFlows.png)

9. Go with the **secure Virtual Host** configuration. Leave the **default** one unchecked.

![image alt text](,/media/host-selection.png)

10. For simplicity, go ahead and select all environments for deployment, then click **Create and Deploy** 

![image alt text](./media/select-environments.png)

11. Once the API proxy is created and deployed click **Edit Proxy** to view your proxy in the proxy editor. 

![image alt text](./media/proxy-created.png)

12. *Congratulations!* ...You have now built a reverse proxy for an existing backend service. You should see the proxy **Overview** tab.

![image alt text](./media/new-proxy-overview.png)

13. Invoke the API at Apigee to see the default JSON format coming from the backend service.

![image alt text](./media/OriginalJSONFormatProducts.png)

## Add transformation of the JSON response

1. Ensure the flow named GetProducts is selected, which handles GET requests that retrieve the complete collection rather than a single product.

![image alt text](./media/VerifyGetProductsSelected.png)

2. Click the button to add a step to the response flow

![image alt text](./media/SelectAddStepForGetProducts.png)

3. Select the JavaScript policy and add it

![image alt text](./media/SelectAddJSPolicy.png)

4. Open the new .js file created for the new JavaScript policy and paste the following

```JavaScript
// Get the array of products from the service response.
var products = context.proxyResponse.content.asJSON.products;
// Loop through each product and standardize the format.
for (i = 0; i < products.length; i++) {
  products[i].price = products[i].priceUsd; // add price attribute
  delete products[i].priceUsd; // remove priceUsd attribute
  products[i].currency = "USD"; // add currency attribute 
}
// Remove the outer object named 'products' and make
// the array the root in the final response.
context.proxyResponse.content.asJSON = products;
```

![image alt text](./media/JSCode.png)

5. Save the changes and ensure they are deployed


6. Invoke the URL to see the tranformed JSON response payload

![image alt text](./media/NewJSONFormat.png)

### Proxy source code

The source code for the proxy that we have constructed so far can also be found [here](./revisions/rev-1_json-transform), for reference.  

## Add support for XML format

1. Click to add a second policy to the response flow

![image alt text](./media/AddSecondPolicy.png)

2. Select the JSONToXML policy and add it

![image alt text](./media/AddJSONToXMLPolicy.png)

3. Review the default configuration for the JSON to XML policy

![image alt text](./media/JSONToXMLPolicyAdded.png)

4. Adjust the policy configuration to change the name given to the root XML element 

![image alt text](./media/JSONToXMLPolicyUpdated.png)

5. Save the changes and ensure the proxy revision is deployed

6. Invoke the API to see the XML returned

![image alt text](./media/XMLResponse.png)

### Proxy source code

The source code for the proxy that we have constructed so far can also be found [here](./revisions/rev-2_json-to-xml), for reference.  