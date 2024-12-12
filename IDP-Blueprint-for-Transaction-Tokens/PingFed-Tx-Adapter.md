Certainly! Let’s review the sample code you provided and the full Transaction Token (Txn-Token) specification. We’ll modify the sample code to create a Transaction Token Generator that complies with the Txn-Token spec.

1. Understanding the Sample Code

The sample code you’ve provided is a simple implementation of a TokenGenerator:
	•	Class Name: SampleTokenGenerator
	•	Purpose: Generates a simple BinarySecurityToken by base64-encoding the subject attribute.
	•	Key Methods:
	•	generateToken(TokenContext attributeContext): Generates the token.
	•	configure(Configuration configuration): Configures the plugin with values from the admin console.
	•	getPluginDescriptor(): Returns the plugin descriptor.

Code Summary:
	•	Uses a single attribute, subject, from the TokenContext.
	•	Encodes the subject using Base64 and sets it as the token data.
	•	Has a simple configuration field, foo.

2. Requirements from the Transaction Token Specification

To comply with the Txn-Token spec, our token generator must:
	•	Generate a JWT: The Txn-Token is a signed JWT.
	•	JWT Header:
	•	Include typ claim set to "txn_token".
	•	Include alg claim for the signing algorithm (e.g., "RS256").
	•	Optionally include kid claim for key rotation.
	•	JWT Body:
	•	Required Claims:
	•	iss: Issuer identifier (URN uniquely identifying the workload or Txn-Token Service).
	•	iat: Issued-at time.
	•	aud: Audience (URN uniquely identifying the trust domain).
	•	exp: Expiration time.
	•	txn: Unique transaction identifier.
	•	sub_id: Unique identifier of the user or workload (MAY follow [SubjectIdentifiers]).
	•	azd: Authorization context details (JSON object).
	•	Optional Claims:
	•	req_ctx: Requester context (JSON object with optional fields).
	•	purp: Purpose of the transaction.
	•	Token Response (Section 7.2):
	•	token_type: MUST be set to "txn_token".
	•	access_token: MUST be the Txn-Token (the signed JWT).
	•	MUST NOT include expires_in, refresh_token, or scope.

3. Planning Modifications to the Sample Code

We need to:
	•	Import JWT Libraries: Use a library like Nimbus JOSE + JWT for JWT creation and signing.
	•	Add Configuration Fields: For issuer (iss), audience (aud), signing keys, and optional kid.
	•	Modify generateToken Method:
	•	Create a JWT with the required header and body claims.
	•	Sign the JWT with a private key.
	•	Return the signed JWT as the access_token.
	•	Ensure Compliance:
	•	The BinarySecurityToken should have the correct TOKEN_TYPE.
	•	Configure the plugin to exclude unwanted response fields.

4. Implementing the Changes

4.1. Update Imports

Add the necessary imports for JWT creation and signing:

import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.core.type.TypeReference;
import java.security.*;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.Base64;
import java.util.Date;
import java.util.UUID;

4.2. Update Constants and Fields

Add new constants for claim names and token types:

private static final String ATTR_NAME_TRANSACTION_DETAILS = "transactionDetails";
private static final String ATTR_NAME_REQUEST_CONTEXT = "requestContext";
private static final String ATTR_NAME_PURPOSE = "purpose";

private static final String TOKEN_TYPE = "txn_token";
private static final String ISSUED_TOKEN_TYPE = "urn:ietf:params:oauth:token-type:txn_token";

Add fields for configuration values:

// Configuration fields
private String issuer;
private String audience;
private String privateKeyPem;
private String keyId; // For 'kid' in JWT header

4.3. Update the Constructor

Modify the constructor to include the new attributes and configuration fields:

public SampleTokenGenerator() {
    Set<String> contract = new HashSet<>(Arrays.asList(
            ATTR_NAME_SUBJECT,
            ATTR_NAME_TRANSACTION_DETAILS,
            ATTR_NAME_REQUEST_CONTEXT,
            ATTR_NAME_PURPOSE
    ));
    GuiConfigDescriptor gui = new GuiConfigDescriptor(NAME);

    // Configuration fields
    gui.addField(new TextFieldDescriptor("Issuer", "The 'iss' (issuer) claim value"));
    gui.addField(new TextFieldDescriptor("Audience", "The 'aud' (audience) claim value"));
    gui.addField(new TextFieldDescriptor("Private Key PEM", "The PEM-encoded private key"));
    gui.addField(new TextFieldDescriptor("Key ID", "Key ID for JWT Header 'kid'"));

    descriptor = new TokenPluginDescriptor(NAME, this, gui, ISSUED_TOKEN_TYPE, contract);
}

4.4. Update the configure Method

Retrieve configuration values entered in the admin console:

@Override
public void configure(Configuration configuration) {
    issuer = configuration.getFieldValue("Issuer");
    audience = configuration.getFieldValue("Audience");
    privateKeyPem = configuration.getFieldValue("Private Key PEM");
    keyId = configuration.getFieldValue("Key ID");
}

4.5. Update the generateToken Method

Implement the logic to create and sign the JWT.

@Override
public BinarySecurityToken generateToken(TokenContext attributeContext) throws TokenProcessingException {
    Map<String, AttributeValue> attributes = attributeContext.getSubjectAttributes();

    // Retrieve attributes
    String subject = getAttributeValue(attributes, ATTR_NAME_SUBJECT);
    String transactionDetails = getAttributeValue(attributes, ATTR_NAME_TRANSACTION_DETAILS);
    String requestContext = getAttributeValue(attributes, ATTR_NAME_REQUEST_CONTEXT);
    String purpose = getAttributeValue(attributes, ATTR_NAME_PURPOSE);

    // Validate required attributes
    if (subject == null || subject.isEmpty()) {
        throw new TokenProcessingException("'sub_id' claim (subject) is missing or empty");
    }
    if (transactionDetails == null || transactionDetails.isEmpty()) {
        throw new TokenProcessingException("'azd' claim (authorization details) is missing or empty");
    }

    // Set token times
    Date issueTime = new Date();
    Date expirationTime = new Date(issueTime.getTime() + 5 * 60 * 1000); // 5 minutes validity

    // Create 'sub_id' claim value
    Map<String, Object> subIdClaim = new HashMap<>();
    subIdClaim.put("format", "email");
    subIdClaim.put("email", subject);

    // Create 'azd' claim value
    Map<String, Object> azdClaim = parseJsonString(transactionDetails);

    // Create 'req_ctx' claim value (optional)
    Map<String, Object> reqCtxClaim = null;
    if (requestContext != null && !requestContext.isEmpty()) {
        reqCtxClaim = parseJsonString(requestContext);
    }

    // Generate 'txn' claim (unique transaction ID)
    String transactionId = UUID.randomUUID().toString();

    // Build JWT Claims
    JWTClaimsSet.Builder claimsBuilder = new JWTClaimsSet.Builder()
            .issuer(issuer) // 'iss' claim
            .issueTime(issueTime) // 'iat' claim
            .audience(audience) // 'aud' claim
            .expirationTime(expirationTime) // 'exp' claim
            .claim("txn", transactionId) // 'txn' claim
            .claim("sub_id", subIdClaim) // 'sub_id' claim
            .claim("azd", azdClaim); // 'azd' claim

    // Optional claims
    if (reqCtxClaim != null) {
        claimsBuilder.claim("req_ctx", reqCtxClaim);
    }
    if (purpose != null && !purpose.isEmpty()) {
        claimsBuilder.claim("purp", purpose);
    }

    JWTClaimsSet claimsSet = claimsBuilder.build();

    // Sign the JWT
    SignedJWT signedJWT;
    try {
        // Load the private key
        PrivateKey privateKey = loadPrivateKey();

        // Sign the JWT
        JWSSigner signer = new RSASSASigner(privateKey);
        JWSAlgorithm signingAlgorithm = JWSAlgorithm.RS256;

        // Build JWT header
        JWSHeader.Builder headerBuilder = new JWSHeader.Builder(signingAlgorithm)
                .type(new JOSEObjectType("txn_token"));
        if (keyId != null && !keyId.isEmpty()) {
            headerBuilder.keyID(keyId);
        }
        JWSHeader header = headerBuilder.build();

        signedJWT = new SignedJWT(header, claimsSet);
        signedJWT.sign(signer);
    } catch (Exception e) {
        throw new TokenProcessingException("Error generating the transaction token", e);
    }

    // Serialize the token
    String txnToken = signedJWT.serialize();

    // Create BinarySecurityToken
    BinarySecurityToken binarySecurityToken = new BinarySecurityToken(XmlIDUtil.createID(), TOKEN_TYPE);
    binarySecurityToken.setEncodedData(txnToken);

    return binarySecurityToken;
}

4.6. Add Helper Methods

Add methods to load the private key and parse JSON strings.

private String getAttributeValue(Map<String, AttributeValue> attributes, String key) {
    AttributeValue attrValue = attributes.get(key);
    return (attrValue != null) ? attrValue.getValue() : null;
}

private Map<String, Object> parseJsonString(String jsonString) throws TokenProcessingException {
    try {
        ObjectMapper objectMapper = new ObjectMapper();
        return objectMapper.readValue(jsonString, new TypeReference<Map<String, Object>>() {});
    } catch (Exception e) {
        throw new TokenProcessingException("Invalid JSON format in attribute value", e);
    }
}

private PrivateKey loadPrivateKey() throws Exception {
    String privateKeyContent = privateKeyPem
            .replaceAll("\\n", "")
            .replace("-----BEGIN PRIVATE KEY-----", "")
            .replace("-----END PRIVATE KEY-----", "")
            .trim();

    byte[] keyBytes = Base64.getDecoder().decode(privateKeyContent);

    PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    return keyFactory.generatePrivate(keySpec);
}

4.7. Remove Unnecessary Code

	•	Remove the Base64 encoding and any code that is no longer relevant.
	•	Update the imports accordingly.

5. Ensuring Compliance with the Specification

	•	JWT Header (Section 5.1):
	•	typ: Set to "txn_token".
	•	alg: Set to the signing algorithm used ("RS256").
	•	kid: Included if keyId is provided.
	•	JWT Body (Section 5.2):
	•	Required Claims:
	•	iss: Configured issuer.
	•	iat: Current time.
	•	aud: Configured audience.
	•	exp: 5 minutes after iat.
	•	txn: Unique transaction ID.
	•	sub_id: Contains format and email of the subject.
	•	azd: Parsed from transactionDetails.
	•	Optional Claims:
	•	req_ctx: Parsed from requestContext if provided.
	•	purp: From purpose attribute if provided.
	•	Token Response (Section 7.2):
	•	The token_type in the response will be set to "txn_token".
	•	The access_token will be the serialized JWT (txnToken).
	•	Ensure that expires_in, refresh_token, and scope are not included in the response.
	•	Configure PingFederate to exclude these fields if necessary.
	•	Security Considerations (Section 9):
	•	Token Lifetime: Set to 5 minutes.
	•	Avoid Including Access Tokens: The code does not include any access tokens within the Txn-Token.
	•	Sender-Constrained Tokens: Not implemented here but can be considered for additional security.

6. Full Updated Code

package com.pingidentity.generator;

import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.*;

import org.sourceid.saml20.adapter.attribute.AttributeValue;
import org.sourceid.saml20.adapter.conf.Configuration;
import org.sourceid.saml20.adapter.gui.FieldDescriptor;
import org.sourceid.saml20.adapter.gui.TextFieldDescriptor;
import org.sourceid.wstrust.model.BinarySecurityToken;
import org.sourceid.wstrust.plugin.TokenProcessingException;
import org.sourceid.wstrust.plugin.generate.TokenContext;
import org.sourceid.wstrust.plugin.generate.TokenGenerator;

import com.nimbusds.jose.*;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.*;
import com.pingidentity.sdk.GuiConfigDescriptor;
import com.pingidentity.sdk.PluginDescriptor;
import org.sourceid.wstrust.plugin.process.TokenPluginDescriptor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.core.type.TypeReference;
import java.util.Base64;
import java.util.Date;
import java.util.UUID;

public class TransactionTokenGenerator implements TokenGenerator {
    private static final String ATTR_NAME_SUBJECT = "subject";
    private static final String ATTR_NAME_TRANSACTION_DETAILS = "transactionDetails";
    private static final String ATTR_NAME_REQUEST_CONTEXT = "requestContext";
    private static final String ATTR_NAME_PURPOSE = "purpose";

    private static final String TOKEN_TYPE = "txn_token";
    private static final String ISSUED_TOKEN_TYPE = "urn:ietf:params:oauth:token-type:txn_token";
    private static final String NAME = "Transaction Token Generator";

    private TokenPluginDescriptor descriptor;

    // Configuration fields
    private String issuer;
    private String audience;
    private String privateKeyPem;
    private String keyId; // For 'kid' in JWT header

    public TransactionTokenGenerator() {
        Set<String> contract = new HashSet<>(Arrays.asList(
                ATTR_NAME_SUBJECT,
                ATTR_NAME_TRANSACTION_DETAILS,
                ATTR_NAME_REQUEST_CONTEXT,
                ATTR_NAME_PURPOSE
        ));
        GuiConfigDescriptor gui = new GuiConfigDescriptor(NAME);

        // Configuration fields
        gui.addField(new TextFieldDescriptor("Issuer", "The 'iss' (issuer) claim value"));
        gui.addField(new TextFieldDescriptor("Audience", "The 'aud' (audience) claim value"));
        gui.addField(new TextFieldDescriptor("Private Key PEM", "The PEM-encoded private key"));
        gui.addField(new TextFieldDescriptor("Key ID", "Key ID for JWT Header 'kid'"));

        descriptor = new TokenPluginDescriptor(NAME, this, gui, ISSUED_TOKEN_TYPE, contract);
    }

    @Override
    public BinarySecurityToken generateToken(TokenContext attributeContext) throws TokenProcessingException {
        Map<String, AttributeValue> attributes = attributeContext.getSubjectAttributes();

        // Retrieve attributes
        String subject = getAttributeValue(attributes, ATTR_NAME_SUBJECT);
        String transactionDetails = getAttributeValue(attributes, ATTR_NAME_TRANSACTION_DETAILS);
        String requestContext = getAttributeValue(attributes, ATTR_NAME_REQUEST_CONTEXT);
        String purpose = getAttributeValue(attributes, ATTR_NAME_PURPOSE);

        // Validate required attributes
        if (subject == null || subject.isEmpty()) {
            throw new TokenProcessingException("'sub_id' claim (subject) is missing or empty");
        }
        if (transactionDetails == null || transactionDetails.isEmpty()) {
            throw new TokenProcessingException("'azd' claim (authorization details) is missing or empty");
        }

        // Set token times
        Date issueTime = new Date();
        Date expirationTime = new Date(issueTime.getTime() + 5 * 60 * 1000); // 5 minutes validity

        // Create 'sub_id' claim value
        Map<String, Object> subIdClaim = new HashMap<>();
        subIdClaim.put("format", "email");
        subIdClaim.put("email", subject);

        // Create 'azd' claim value
        Map<String, Object> azdClaim = parseJsonString(transactionDetails);

        // Create 'req_ctx' claim value (optional)
        Map<String, Object> reqCtxClaim = null;
        if (requestContext != null && !requestContext.isEmpty()) {
            reqCtxClaim = parseJsonString(requestContext);
        }

        // Generate 'txn' claim (unique transaction ID)
        String transactionId = UUID.randomUUID().toString();

        // Build JWT Claims
        JWTClaimsSet.Builder claimsBuilder = new JWTClaimsSet.Builder()
                .issuer(issuer) // 'iss' claim
                .issueTime(issueTime) // 'iat' claim
                .audience(audience) // 'aud' claim
                .expirationTime(expirationTime) // 'exp' claim
                .claim("txn", transactionId) // 'txn' claim
                .claim("sub_id", subIdClaim) // 'sub_id' claim
                .claim("azd", azdClaim); // 'azd' claim

        // Optional claims
        if (reqCtxClaim != null) {
            claimsBuilder.claim("req_ctx", reqCtxClaim);
        }
        if (purpose != null && !purpose.isEmpty()) {
            claimsBuilder.claim("purp", purpose);
        }

        JWTClaimsSet claimsSet = claimsBuilder.build();

        // Sign the JWT
        SignedJWT signedJWT;
        try {
            // Load the private key
            PrivateKey privateKey = loadPrivateKey();

            // Sign the JWT
            JWSSigner signer = new RSASSASigner(privateKey);
            JWSAlgorithm signingAlgorithm = JWSAlgorithm.RS256;

            // Build JWT header
            JWSHeader.Builder headerBuilder = new JWSHeader.Builder(signingAlgorithm)
                    .type(new JOSEObjectType("txn_token"));
            if (keyId != null && !keyId.isEmpty()) {
                headerBuilder.keyID(keyId);
            }
            JWSHeader header = headerBuilder.build();

            signedJWT = new SignedJWT(header, claimsSet);
            signedJWT.sign(signer);
        } catch (Exception e) {
            throw new TokenProcessingException("Error generating the transaction token", e);
        }

        // Serialize the token
        String txnToken = signedJWT.serialize();

        // Create BinarySecurityToken
        BinarySecurityToken binarySecurityToken = new BinarySecurityToken(XmlIDUtil.createID(), TOKEN_TYPE);
        binarySecurityToken.setEncodedData(txnToken);

        return binarySecurityToken;
    }

    private String getAttributeValue(Map<String, AttributeValue> attributes, String key) {
        AttributeValue attrValue = attributes.get(key);
        return (attrValue != null) ? attrValue.getValue() : null;
    }

    private Map<String, Object> parseJsonString(String jsonString) throws TokenProcessingException {
        try {
            ObjectMapper objectMapper = new ObjectMapper();
            return objectMapper.readValue(jsonString, new TypeReference<Map<String, Object>>() {});
        } catch (Exception e) {
            throw new TokenProcessingException("Invalid JSON format in attribute value", e);
        }
    }

    private PrivateKey loadPrivateKey() throws Exception {
        String privateKeyContent = privateKeyPem
                .replaceAll("\\n", "")
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .trim();

        byte[] keyBytes = Base64.getDecoder().decode(privateKeyContent);

        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePrivate(keySpec);
    }

    @Override
    public void configure(Configuration configuration) {
        issuer = configuration.getFieldValue("Issuer");
        audience = configuration.getFieldValue("Audience");
        privateKeyPem = configuration.getFieldValue("Private Key PEM");
        keyId = configuration.getFieldValue("Key ID");
    }

    @Override
    public PluginDescriptor getPluginDescriptor() {
        return descriptor;
    }
}

7. Configuring PingFederate

To ensure the response conforms to the spec:
	•	Token Mapping:
	•	Map access_token to the value returned by binarySecurityToken.getEncodedData().
	•	Set token_type to "txn_token".
	•	Set issued_token_type to "urn:ietf:params:oauth:token-type:txn_token".
	•	Exclude Unwanted Fields:
	•	Ensure expires_in, refresh_token, and scope are not included in the response.
	•	This can be configured in the Token Generator Mapping or via custom policies.

8. Testing and Validation

	•	Deploy the Plugin:
	•	Compile the code and package it into a JAR file.
	•	Place the JAR in the appropriate directory for PingFederate plugins.
	•	Configure the Plugin in PingFederate:
	•	Add a new Token Generator using the Transaction Token Generator.
	•	Provide configuration values for Issuer, Audience, Private Key PEM, and Key ID.
	•	Set Up Token Exchange:
	•	Configure PingFederate to handle OAuth 2.0 Token Exchange requests.
	•	Map the necessary attributes from incoming tokens to the attributes expected by the plugin.
	•	Test Token Issuance:
	•	Send a Token Exchange request as per Section 7.1 of the spec.
	•	Verify that the response conforms to Section 7.2.
	•	Validate the Txn-Token:
	•	Decode the JWT to ensure all required claims are present and correctly formatted.
	•	Verify the signature using the corresponding public key.

9. Conclusion

By adapting the sample code and integrating the requirements from the Transaction Token specification, we’ve created a TransactionTokenGenerator that complies with the spec.
	•	Key Changes:
	•	Implemented JWT creation and signing.
	•	Included required and optional claims as per the spec.
	•	Configured the token response to match the spec’s requirements.
	•	Security Considerations:
	•	Ensured the private key is handled securely.
	•	Set a short token lifetime.
	•	Avoided including sensitive information in logs or error messages.

If you have any questions or need further assistance with any part of the implementation, feel free to ask!