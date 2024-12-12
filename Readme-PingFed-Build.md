ant jar-deploy   
ant jar-plugin
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home       
export PATH=$JAVA_HOME/bin:$PATH

# RAR - Rich Authorization Request (RFC 9396)

RAR is an extension to the OAuth protocol enabling an application to display more details to resource owners during an authorization request.
 RAR can be used in addition to OAuth SCOPE and can be considered a more expressive version of SCOPE (on a high level).

The RFC defines the additional request parameter **authorization_details**. This parameter carries a JSON structure of any shape, only **type** is a required claim.

This example showcases an implementation of a simple authorization_details processor for PingFederate.

The implementation supports the example payload as used at the RFC itself ([https://datatracker.ietf.org/doc/html/rfc9396#name-introduction](https://datatracker.ietf.org/doc/html/rfc9396#name-introduction)).

## Prepare your environment

At this point in time (current PingFederate: 12.1) the below tools and products are required for this example:

- PingFederate
  - download a zip file (i.e.: pingfederate-12.1.zip)
  - place it under **./products/pingfederate** and unzip it
  - a matching license is required if the PingFederate instance is also used to test the plugin
- Ant
  - in future versions of PingFederate it will be replaced by Maven
  - install ANT and have the command **ant** executable within this project
- Java 11
- Optional: OAuthPlayground
  - download from here: [https://www.pingidentity.com/en/resources/downloads/pingfederate.html](https://www.pingidentity.com/en/resources/downloads/pingfederate.html) under the tab **Add-Ons**
  - place it under **./products/oauthplayground** and unzip it
- Optional: an IDE such as IntelliJ (the one and only)

**Info:** This example was developed on a MacBook. For Windows or Linux machines, some commands may be different.

## Getting started

The authorization_details processor will be implemented as a plugin for PingFederate. The unzipped PingFederate directory includes a directory called **SDK**.

To verify that your environment is ready for implementation:

- open this link in a browser: [SDK Developer Guide](https://docs.pingidentity.com/pingfederate/latest/sdk_developers_guide/pf_sdk_develop_guide.html)
  - briefly read through that website to get a better picture of what is done here
- navigate to **SDK Directory Structure**, take a look, get familiar with the directory structure
- navigate to **Developing your own plugin**
  - this is the overall guideline for this project. Keep that page open for later

In a terminal (or in your IDE) do this:

- `cd ./products/pingfederate/pingfederate-12.1.0/pingfederate/sdk`
- `ant jar-plugin`

This should result in the message **BUILD SUCCESSFUL**. This means your environment is able to compile and build a plugin!
If that message does not appear check if **ant** is available (i.e.: ant -version), if java is found, if the command was executed in the correct directory.

To start our example project follow the instructions as documented for **Developing your own plugin**:

- the terminal should still be at **.../sdk**
- create this directory structure and java file:
- `./plugin-src/authorization-details-example/java/com/pingfederate/webinar/adapters/AuthorizationDetailsHandler.java`
  - the java file will contain our implementation
- update `build.local.properties` and configure **target-plugin.name=authorization-details-example**
- run `ant jar-plugin`

Check the content of the directory **./products/pingfederate/pingfederate-12.1.0/pingfederate/sdk/plugin-src/authorization-details-example**
- find **build/classes** which is still empty
- find **build/jar/pf.plugins.authorization-details-example.jar** which is the plugin for PingFederate. Of course, we have not implemented anything yet.

If this has all worked out we are ready to start the implementation. If there were any errors fix them first.

## Set up the source code

Copy this into the empty java file **AuthorizationDetailsHandler.java** which is our starting point:

```java
package com.pingfederate.webinar.adapters;

public class AuthorizationDetailsHandler implements AuthorizationDetailProcessor {

  private static final Logger LOGGER = Logger.getLogger(AuthorizationDetailsHandler.class.getName());
  
}
```

If you open the file in IntelliJ it will show errors due to missing dependencies and not identifying the source code.

To fix that do the following:

- mark **products/pingfederate/pingfederate-12.1.0/pingfederate/sdk/plugin-src/authorization-details-example/java** as **Source Root**
- update the project settings and include this jar as library:
  - **./products/pingfederate/pingfederate-12.1.0/pingfederate/server/default/lib/pf-protocolengine.jar**

This fixes the dependencies, now use IntelliJ to **implement** the methods of the interface **AuthorizationDetailProcessor**.

Run `ant jar-plugin`. If this worked ... hooray, ready to implement!

## Implementation details

The java class should have stubs for these methods:

- public AuthorizationDetailValidationResult validate(...)
  - validate the incoming authorization_details
- public AuthorizationDetail enrich(...) throws AuthorizationDetailProcessingException
  - add additional details to process in other methods
- public boolean isEqualOrSubset(...) throws AuthorizationDetailProcessingException
  - check if the requested authorization_details is the same as the granted one
- public void configure(...)
- public String getUserConsentDescription(...) throws AuthorizationDetailProcessingException
  - return the content for the approval screen (consent screen)
- public PluginDescriptor getPluginDescriptor()
  - the plugin configuration for PingFederate

Complete the following methods as they are not as relevant for this example:

```java
    @Override
    public AuthorizationDetailValidationResult validate(AuthorizationDetail authorizationDetail, AuthorizationDetailContext authorizationDetailContext, Map<String, Object> map) {
        return AuthorizationDetailValidationResult.createValidResult();
    }

    @Override
    public AuthorizationDetail enrich(AuthorizationDetail authorizationDetail, AuthorizationDetailContext authorizationDetailContext, Map<String, Object> map) throws AuthorizationDetailProcessingException {
        return authorizationDetail;
    }

    @Override
    public boolean isEqualOrSubset(AuthorizationDetail requestedAuthorizationDetail, AuthorizationDetail acceptedAuthorizationDetail, AuthorizationDetailContext authorizationDetailContext, Map<String, Object> map) throws AuthorizationDetailProcessingException {
      LOGGER.info("acceptedAuthorizationDetail.getIdentifier(): " + acceptedAuthorizationDetail.getIdentifier());
      return true;
    }
    
    @Override
    public void configure(Configuration configuration) {
      LOGGER.info("configure");
    }    
```

The two important methods for this example are these:

- **getUserConsentDescription(...)** - this will construct the message that is displayed to the resource owner at the approval screen (consent screen) next to requested SCOPE values
- **getPluginDescriptor()** - details about the plugin and its configuration

### getUserConsentDescription(...)

This is an important method as it is used to construct the resource owner friendly message that has to contain all important details that need the resource owners attention!

As mentioned, this implementation processes an authorization_detail as shown at the RFC (but as an array):

```json
[
  {
    "type": "payment_initiation",
    "locations": [
      "https://example.com/payments"
    ],
    "instructedAmount": {
      "currency": "EUR",
      "amount": "123.50"
    },
    "creditorName": "Merchant A",
    "creditorAccount": {
      "bic": "ABCIDEFFXXX",
      "iban": "DE02100100109307118603"
    },
    "remittanceInformationUnstructured": "Ref Number Merchant"
  }
]
```

The code below receives this JSON payload as part of the parameter **authorizationDetail**:

```java
    @Override
    public String getUserConsentDescription(AuthorizationDetail authorizationDetail, AuthorizationDetailContext authorizationDetailContext, Map<String, Object> map) throws AuthorizationDetailProcessingException {
        StringBuilder msg = new StringBuilder();
        msg.append("Payment recipient: ").append(((List<String>)authorizationDetail.getDetail().get("locations")).get(0));
        Map<String, Object> instructedAmount = (Map<String, Object>) authorizationDetail.getDetail().get("instructedAmount");
        msg.append(", amount and currency: ").append(instructedAmount.get("amount")).append(" ").append(instructedAmount.get("currency")).append("\n");
        return msg.toString();
    }
```

The code is pretty straight forward and builds a simple string for the approval screen (consent screen).

A few things to consider:

- the claim **type** is ignored. If this implementation process multiple types a branch per type would be required
- the code assumes the payload is always correct. A production grade implementation would have to validate the payload, check for optional values, manage defaults, etc.

Copy that code snippet and replace the stub method in our java class.

### getPluginDescriptor()

This method is used by PingFederate to invoke the configuration screens. The text blocks are displayed to an administrator, therefore it should make sense in the target environment.

```java
    @Override
    public PluginDescriptor getPluginDescriptor() {

        GuiConfigDescriptor guiConfigDescriptor = new GuiConfigDescriptor();
        guiConfigDescriptor.setDescription("Webinar Demo AuthorizationDetailsProcessor");

        AuthorizationDetailProcessorDescriptor descriptor = new AuthorizationDetailProcessorDescriptor(
                "First AuthorizationDetailsProcessor",
                this,
                guiConfigDescriptor,
                "1.0.0"
        );

        Set<String> attributeContractSet = new HashSet<>();
        attributeContractSet.add("WebinarAttributeContractSetKey");

        Map<String, Object> metaData = new HashMap<>();
        metaData.put("WebinarMetaDataKey", "WebinarMetaDataKeyValue");

        Set<String> authorizationDetailsSupported = new HashSet<>();
        authorizationDetailsSupported.add("payment_initiation");

        descriptor.setMetadata(metaData);
        descriptor.setAttributeContractSet(attributeContractSet);
        descriptor.setSupportedAuthorizationDetailTypes(authorizationDetailsSupported);
        descriptor.setSupportsExtendedContract(false);

        return descriptor;
    }
```

Once we launch this processor and open PingFederates UI it becomes obvious what this code is for.

**Important:**
- this line: **authorizationDetailsSupported.add("payment_initiation")** has to match the **type** claims of the supported authorizaiont details payload
- for multiple type,s multiple types need to be added

Copy that code snippet and replace the stub method in our java class.

### Install and test the plugin

Time run build and deploy the plugin:

- `ant jar-plugin`  // as before, build the plugin
- `ant deploy-plugin`  // installs the plugin at: **products/pingfederate/pingfederate-12.1.0/pingfederate/server/default/deploy**

If a different PingFederate instance is used for testing purposes deploy the plugin in that instance.

**Launch PingFederate**

- `cd products/pingfederate/pingfederate-12.1.0/pingfederate/bin/run.sh`
- `sh run.sh`
- open a browser at **https://localhost:9999**
- accept the license, create and administrative account
- do NOT connect to PingOne as this is not relevant for this example

**Find our implementation**

- open **System - OAuth Settings - Authorization Detail Processor**
- select **Create New Instance**
- instance name: **fadp**, instance ID: **fadp**
- select **First AuthorizationDetailsProcessor**
- click **Next**, **Next** and **Save**

- open **System - OAuth Settings - Authorization Detail Types**
- select **Add Authorization Detail Type**
- Type: **payment_initiation**
- select **fadp** as the authorization details processor

The plugin is now available and configured!

### Use the plugin

The plugin can now be assigned to any OAuth client. Try this:
- **Applications - OAuth Clients - Add Client**
- on that screen scroll down and look for **Authorization Details Types**
  - check **Allow Authorization Details**
  - select **payment_initiation**

If this OAuth client would be registered it would be able to process authorization_details of type **payment_initiation** and would display the implemented message for the approval screen (consent screen). If you know how to register and configure an OAuth client in PingFederate, go ahead and enable the authorization_details processor.

### OAuth Playground

To keep things simple this demo uses OAuthPlayground which is an OAuth test client and registered itself very easily.

To install it, do the following:
- `cp -r ./products/oauthplayground/OAuthPlayground-4.4/dist/deploy/* ./products/pingfederate/pingfederate-12.1.0/pingfederate/server/default/deploy`
- `cp -r ./products/oauthplayground/OAuthPlayground-4.4/dist/conf/* ./products/pingfederate/pingfederate-12.1.0/pingfederate/server/default/conf`

Launch PingFederate and navigate to: **https://localhost:9031/OAuthPlayground**.

Select **Setup**:
- Admin Host: **localhost:9999**
- Certificate Issues: **Ignore**
- Admin Username: **whatever you configured**
- Admin Password: **whatever you configured**
- **Next**
- Skip CIBA
- **Next**
- **Done**

OAuthPlayground has now registered a few oauth clients. For the purpose of this example, do the following:

Navigate to: **https://localhost:9999** and sign in
- **Applications - OAuth Clients**
- select **ac_oidc_client**
- enable and select the authorization details type

Now go back to OAuthPlayground to initiate an OpenID Connect flow:

- on the welcome page select **Authorization Code**
- select **Use OpenID Connect**
- select **Add Parameter** and set **authorization_details**
- open and copy the content of **./doc/example-authorization-details.txt**
  - this is the string version of the example authorization_Details message
- paste it as payload for the authorization_details parameter
- select **Submit**
- Username: **joe**, password: **2Federate**
- on the appearing approval screen you can find the message that we developed earlier:
  - **Payment Recipient: https://example.com/payments, Amount and Currency: 123.50 EUR**
- hit **Submit** on the next screen where the received authorization_code is exchanged for an access_token
- click on **Show raw** at the top and copy the JSON part of the response into a tool that pretty prints JSON (i.e.: https://jsonlint.org) 
- the granted authorization_detail is part of the response

Well done!

### Debugging the code

Debugging the code is great to fix the implementation but also to understand which method is being invoked when during the authorization flow.

To enable debugging do the following in the terminal in which run.sh will be executed:
- `export JAVA_OPTS="$JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"`
- `sh run.sh`
- Set break points in the methods of the processor implementation
- configure a **Remote JVM Debugging** configuration in IntelliJ
  - give it a name
  - configure JDK >=9
  - use default port 5005
  - run this debug configuration
  - run through the authorization flow and each method will be invoked

# Info

The example class can be found here: **./doc/AuthorizationDetailsHandler.java**.

# Links

- Download for PingFederate: [https://www.pingidentity.com/en/resources/downloads/pingfederate.html](https://www.pingidentity.com/en/resources/downloads/pingfederate.html)
- Documentation for OAuth Playground: [https://docs.pingidentity.com/pingfederate/latest/developers_reference_guide/pf_oauth_endpoints.html](https://docs.pingidentity.com/pingfederate/latest/developers_reference_guide/pf_oauth_endpoints.html)

# DISCLAIMER

This is not a production environment and is meant for educational purposes only.

It does not come with any kind of warranty, it is provided as is, it was not reviewed for potential performance or security weaknesses.

Logs are verbose to provide insight into the configuration and may include secrets used during this demo.

Use this project to evaluate this setup. Modify it as much or as little as you like.

Report any issues here in GitHub, PingIdentities support team cannot provide assistance of any kind.

This code is to be used exclusively in connection with Ping Identity Corporation software or services. Ping Identity Corporation only offers such software or services to legal entities who have entered into a binding license agreement with Ping Identity Corporation.

# License

This is licensed under the Apache License 2.0.