---
layout: page
title: Authentication API
permalink: directIntegration/authentication/
nav_order: 4
toc_list: true
---

# Overview

This API is based on the Authorization Code Flow of OpenID Connect 1.0. It allows you to verify the identity of an end-user based on the authentication performed by an authorization server, as well as to obtain basic profile information about the end-user. 

The diagram below describes the **Authentication**  process and how your systems will be interacting with itsme® :
  
 ![Sequence diagram describing the OpenID flow](/doc/public/images/OpenID_SeqDiag.png)

To get to this result please make sure you 

<ol>
  <li>add itsme® button to your front-end page so the user can indicate he wishes to authenticate with itsme® : <a href="https://brand.belgianmobileid.be/d/CX5YsAKEmVI7/documentation#/ux/buttons-1518207548" target="blank">itsme® button specifications</a>.</li>
  <li>create the <a href="#AuthNReq" >Authorization Request</a> to authenticate the User. This request will redirect the user to the itsme® app. itsme® will then authenticates the user by asking him 
    <ul type>
      <li>to enter his phone number on the itsme® sign-in page</li> 
      <li>authorize the release of some information to your application</li>
      <li>to provide his credentials (itsme® code, fingerprint or FaceID)</li>
    </ul><br>It is also in this Authorization Request that you will be able to request claims about the user and the Authentication event.</li>
  <li><a href="#AuthNResp" >collect the Authorization Code</a> once the user has been authenticated and redirected by itsme® to your mobile or web application.</li>
  <li><a href="#TokenReq" >exchange the Authorization Code for an ID token</a> (e.g. identifying the user) and an Access Token.</li>
  <li>Obtain the additional claims by <a href="#UserInfoReq" >presenting the access token to the itsme® UserInfo Endpoint</a> if the required claims are not returned in the ID token.</li>
  <li>Confirm the success of the operation and display a success message.</li>
</ol>

# Guides

## Generate itsme® button

First, you will need to create a button to allow your users to authenticate with itsme®. Check the <a href="https://brand.belgianmobileid.be/d/CX5YsAKEmVI7/documentation#/ux/buttons-1518207548" target="blank">Button design guide</a> before you start the integration. 

Upon clicking this button, the browser will redirect the User to our Front-End. itsme® then take care of authenticating him.


## Client Authentication methods

itsme® supports the following authentication methods to protect the exchange of sensitive information and ensure the requested information gets issued to a legitimate application and not some other party :

<ul>
  <li>asymmetric RSA key pair</li>
  <li>symmetric shared secret</li>
</ul>

itsme® recommend using the asymmetric RSA key pair method and allow you to rotate your keys without the need to sync with us.  

### Asymmetric RSA 

This method requires that each party exposes its public keys as a simple JWK Set document on a URI publicly accessible, and keep its private keys for itself. For itsme®, this URI can be retrieved from the [itsme® Discovery document](#OpenIDConfig), using the <i>"jwks_uri"</i> key.

Your private and public keys can be generated using your own tool or via Yeoman. If using Yeoman, you need to install generator-itsme with NPM:

```
$ npm install -g yo generator-itsme
```

After installation, run the generator:

```
$ yo itsme
```

The Yeoman tool will generate two files, the jwks_private.json which MUST be stored securely in your system, and the jwks_public.json which needs to be exposed as a JWK Set on a URI publicly accessible and have the HTTPS scheme.

<aside class="notice">Whatever the tool you are choosing to create your key pairs, don't forget to send your JWK Set URI by email to <a href = "mailto: onboarding@itsme.be">onboarding@itsme.be</a> and itsme® will make sure to complete the configuration for you in no time!
</aside>

### Symmetric secret 

This method requires the exchange of a static secret that will be used to authenticate with our Back-End. 

The client_secret value will be provided by itsme® when <a href="https://belgianmobileid.github.io/doc/getting-started.html#getting-started" target="blank">registering your project</a>.

## Certificates and website security



## Mapping the user

To sign in successfully in your web desktop, mobile web or mobile application, a given user must be mapped to a user account in your database. By default, your application Server will use the subject identifier, or <code>sub</code> claim, in the ID Token to identify and verify a user account. The <code>sub</code> claim is a string that uniquely identifies a given user account. The benefit of using a <code>sub</code> claim is that it will not change, even if other user attributes (email, phone number, etc) associated with that account are updated.

The <code>sub</code> claim value must be mapped to the corresponding user in your application Server. 

If no user record is storing the <code>sub</code> claim value, then you should allow the user to associate his new or existing account to the <code>sub</code>.

All these flows are depicted in the itsme® B2B portal.

In a limited number of cases (e.g. technical issue,…) a user could ask itsme® to ‘delete’ his account. As a result the specific account will be ‘archived’ (for compliancy reasons) and thus also the unique identifier(s) (e.g. "sub").

If the same user would opt to re-create an itsme® afterwards, he will need to re-bind his itsme® account with your application server, in the same way as for the initial binding. After successful re-binding you will need to overwrite the initial reference with the new <code>sub</code> claim value in your database.


# API reference

<a id="OpenIDConfig"></a>
## itsme® Discovery Document

{% tabs DiscoveryDocument %}

{% tab DiscoveryDocument RSA keys %}

<b><code>GET https://idp.<i>[e2e/prd]</i>.itsme.services/v2/.well-known/openid-configuration</code></b>

{% endtab %}

{% tab DiscoveryDocument Symmetric keys %}

<b><code>GET https://oidc.<i>[e2e/prd]</i>.itsme.services/clientsecret-oidc/csapi/v0.1/.well-known/openid-configuration</code></b>

{% endtab %}

{% endtabs %}

To simplify implementations and increase flexibility, <a href="https://openid.net/specs/openid-connect-discovery-1_0.html" target="blank">OpenID Connect allows the use of a Discovery Document</a>, a JSON document containing key-value pairs which provide details about itsme® configuration, such as the  

<ul>
  <li>Authorization, Token and userInfo Endpoints</li>
  <li>supported claims</li>
  <li>...</li>
</ul>


<a id="AuthNReq"></a>
## Authorization Request

{% tabs AuthorizationRequest %}

{% tab AuthorizationRequest RSA keys %}

<b><code>GET https://idp.<i>[e2e/prd]</i>.itsme.services/v2/authorization</code></b>

### Parameters

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="client_id" req="REQUIRED" %}</td>
      <td>It identifies your application. This parameter value is generated during registration.</td>
    </tr>
     <tr>
      <td>{% include parameter.html name="response_type" req="REQUIRED" %}</td>
      <td>This defines the processing flow to be used when forming the response. Because itsme® supports the Authorization Code Flow, this value MUST be <code>code</code>.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="scope" req="REQUIRED" %}</td>
      <td>
        It allows your application to express the desired scope of the access request. Each scope returns a set of user attributes. The scopes an application should request depend on which user attributes your application needs. Once the user authorizes the requested scopes, his details are returned in an ID Token and are also available through the UserInfo Endpoint.<br><br>All scope values must be space-separated.<br><br>The basic (and required) scopes are <code>openid</code> and <code>service</code>. Beyond that, your application can ask for additional standard scopes values which map to sets of related claims are: <code>profile</code> <code>email</code> <code>address</code> <code>phone</code><br />
        <table>
          <tr>
            <td>{% include parameter.html name="service" req="REQUIRED" %}</td><td>It indicates the itsme® service your application intends to use, e.g. <code>service:TEST_code</code> by replacing "TEST_code" with the service code generated during registration.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="openid" req="REQUIRED" %}</td><td>It indicates that your application intends to use the OpenID Connect protocol to verify a user's identity by returning a <code>sub</code> claim which represents a unique identifier for the authenticated user.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="profile" req="OPTIONAL" %}</td><td>Returns claims that represent basic profile information, specifically <code>family_name</code>, <code>given_name</code>, <code>name</code>, <code>gender</code> and <code>birthdate</code>.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="email" req="OPTIONAL" %}</td><td>Returns the <code>email</code> claim, which contains the user's email address, and <code>email_verified</code>, which is a boolean indicating whether the email address was verified by the user.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="address" req="OPTIONAL" %}</td><td>Returns the information about the user's postal address. The value of the address member is a JSON structure containing some or all of these members <code>formatted</code> <code>street_address</code> <code>postal_code</code> <code>locality</code> <code>country</code></td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone" req="OPTIONAL" %}</td><td>Returns the <code>phone</code> claim, which contains the user's phone number, and <code>phone_number_verified</code>, which is a boolean indicating whether the phone number was verified by the user.</td>
          </tr> 
        </table>
      </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="redirect_uri" req="REQUIRED" %}</td>
      <td>It is the URL to which users are redirected once the authentication is complete. It MUST match the value preregistered during the registration and use the HTTPS scheme. http://localhost is only permitted for development purposes, it’s not for use in production.<br><br><b>Note</b> : if relevant (in case you have a mobile app) make sure that your redirect URIs support the Universal links and App links mechanism. Functionally, it will allow you to have only one single link that will either open your desktop web application, your mobile app or your mobile site on the User’s device. Universal links and App links are standard web links (http://mydomain.com) that point to both a web page and a piece of content inside an app. When a Universal Link is opened, the app OS checks to see if any installed app is registered for that domain. If so, the app is launched immediately without ever loading the web page. If not, the web URL is loaded into the webbrowser.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="state" req="Strongly RECOMMENDED" %}</td>
      <td>Specifies any string value that your application uses to maintain state between your Authorization Request and the Authorization Server's response. You can use this parameter for several purposes, such as directing the user to the correct resource in your application and mitigating cross-site request forgery.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="nonce" req="Strongly RECOMMENDED" %}</td>
      <td>A string value used to associate a session with an ID Token, and to mitigate replay attacks. The value is passed through unmodified from the Authentication Request to the ID Token.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="login_hint" req="OPTIONAL" %}</td>
      <td>If your application knows which user is trying to authenticate, it can use this parameter to pre-fill the phone number of the user on the itsme® sign-in page, e.g. <code>32+123456789</code> with <code>32</code> the country code and <code>123456789</code> the user's phone number.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="ui_locales" req="OPTIONAL" %}</td>
      <td>Indicates the user's preferred languages for the itsme® sign-in page, represented as a space-separated list of language tag values, ordered by preference.<br><br>Possible values : <code>fr</code> <code>nl</code> <code>de</code> <code>en</code></td>
    </tr>
    <tr>
      <td>{% include parameter.html name="display" req="OPTIONAL" %}</td>
      <td>Specify how the itsme® sign-in page should be displayed to the user. If set to <code>touch</code>, it SHOULD displays the itsme® sign-in page with a device that leverages a touch interface. If set to <code>page</code>, the itsme® sign-in UI SHOULD be consistent with a full page view of the User-Agent. If the <code>display</code> parameter is not specified, this is the default display mode.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="acr_values" req="OPTIONAL" %}</td>
      <td>Indicates the authentication method required to process the request, represented as a space-separated list of tag values, ordered by preference.<br><br>Possible values : <code>http://itsme.services/v2/claim/acr_basic</code> <code>http://itsme.services/v2/claim/acr_advanced</code><br><br><b>Note</b> : If these two values are provided only the most constraining authentication method will be applied, e.g. <code>http://itsme.services/v2/claim/acr_advanced</code>.<br />
        <table>
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/acr_basic" req="" %}</td><td>It lets the user to choose either fingerprint usage (if device is compatible) or itsme® code. If the <code>acr_values</code> parameter is not specified, this is the default authentication method.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/acr_advanced" req="" %}</td><td>It forces the user to use his itsme® code.</td>
          </tr>
        </table>
      </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="claims" req="OPTIONAL" %}</td>
      <td>Requests specific user's details to be returned from the UserInfo Endpoint and/or in the ID Token. It is represented as a JSON object that has two members - <code>{"userinfo":{...}</code> and <code>{"id_token":{...}</code>, which content indicates which claims to return at the UserInfo Endpoint and which at the ID Token, together with indication whether the claim is voluntary (default) or essential.<br><br>Possible user's details your application can request is listed below.<br />
        <table>
          <tr>
            <td>{% include parameter.html name="name" req="OPTIONAL" %}</td><td>User's full name in displayable form including all name parts, possibly including titles and suffixes</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="given_name" req="OPTIONAL" %}</td><td>Given name(s) or first name(s) of the user. Note that in some cultures, people can have multiple given names; all can be present, with the names being separated by space characters.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="family_name" req="OPTIONAL" %}</td><td>Surname(s) or last name(s) of the user. Note that in some cultures, people can have multiple family names or no family name; all can be present, with the names being separated by space characters.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="birtdate" req="OPTIONAL" %}</td><td>User's birthday, represented as a YYYY-MM-DD format.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/birthdate_as_string" req="OPTIONAL" %}</td><td>User's birthday, represented as a string.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="gender" req="OPTIONAL" %}</td><td>User's gender. Possible values are : <code>female</code> <code>male</code></td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="email" req="OPTIONAL" %}</td><td>User's email address.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="email_verified" req="OPTIONAL" %}</td><td><code>true</code> if the user's e-mail address is verified; otherwise <code>false</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone_number" req="OPTIONAL" %}</td><td>User's phone number, in <a href="https://en.wikipedia.org/wiki/E.164" target="blank">E.164</a> format.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone_number_verified" req="OPTIONAL" %}</td><td><code>true</code> if the user's e-phone number is verified; otherwise <code>false</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="address" req="OPTIONAL" %}</td><td>Returns the information about the user's postal address. The value of the address is a JSON structure containing some or all of these members <code>formatted</code> <code>street_address</code> <code>postal_code</code> <code>locality</code> <code>country</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_citizenship" req="OPTIONAL" %}</td><td>The jurisdiction that has conferred citizenship rights to the user, in <a href="https://en.wikipedia.org/wiki/ISO_3166" target="blank">ISO 3166</a> format.</td>
          </tr> 
           <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/place_of_birth" req="OPTIONAL" %}</td><td>The location where the user was born. The value of this attribute is a JSON structure containing some or all of these members <code>formatted</code> <code>city</code> <code>country</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/physical_person_photo" req="OPTIONAL" %}</td><td>XXX</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/BEeidSn" req="OPTIONAL" %}</td><td>Returns the Belgian ID card number, 12 digits in the form xxx-xxxxxxx-yy. The check-number yy is the remainder of the division of xxxxxxxxxx by 97.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_device" req="OPTIONAL" %}</td><td>Returns information about the user's phone. The value of this attribute is a JSON structure containing some or all of these members <code>os</code> <code>appName</code> <code>appRelease</code> <code>deviceLabel</code> <code>debugEnabled</code> <code>deviceID</code>	<code>osRelease</code> <code>manufacturer</code> <code>deviceLockLevel</code> <code>smsEnabled</code> <code>rooted</code> <code>msisdn</code> <code>deviceModel</code>	<code>sdkRelease</code>.</td>       
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/transaction_info" req="OPTIONAL" %}</td><td>Returns information about the itsme® transaction. The value of this attribute is a JSON structure containing some or all of these members <code>securityLevel</code> <code>bindLevel</code> <code>appRelease</code>.</td>
           </tr> 
           <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/BENationalNumber" req="OPTIONAL" %}</td><td>Returns the unique identification number of natural persons who are registered in Belgium. This number consists of 11 digits of the form yy.mm.dd-xxx.cd where yy.mm.dd is the birthdate of the person, xxx a sequential number (odd for males and even for females) and cd a check-digit.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_luxtrust_ssn" req="OPTIONAL" %}</td><td>Returns an unique identification number of natural persons who are registered via Luxtrust. This number consists of 20 digits.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_nl_bsn" req="OPTIONAL" %}</td><td>Returns the citizen service number, a unique registration number for everyone who lives in the Netherlands. This number consists of 8 to 9 digits.</td>
          </tr> 
        </table>
      </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="request_uri" req="OPTIONAL" %}</td>
      <td>A URL using the https scheme referencing a resource containing a JWT whose claims are the request parameters. The <code>request_uri</code> parameter is used to secure parameters in the authentication request from tainting or inspection when sending the request to the itsme® Authorization Endpoint.<br><br>If used, the JWT MUST be signed then encrypted, with the result being a Nested JWT.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="request" req="OPTIONAL" %}</td>
      <td>It represents the request as a JWT whose Claims are the request parameters. The <code>request</code> parameter is used to secure parameters in the authentication request from tainting or inspection when sending the request to the itsme® Authorization Endpoint.<br><br>If used, the JWT MUST be signed then encrypted, with the result being a Nested JWT.</td>
    </tr>
  </tbody>
</table>

<a id="AuthNResp"></a>
### Response

<code>302</code> <code>application/json</code>

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="code" req="" %}</td>
      <td>An intermediate opaque credential used to retrieve the ID Token and Access Token.<br><br><b>Note</b> : the code has a lifetime of 3 minutes.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="state" req="" %}</td>
      <td>The string value provided in the Authorization Request. You SHOULD validate that the value returned matches the one supplied.</td>
    </tr>
  </tbody>
</table>

{% endtab %}

{% tab AuthorizationRequest Symmetric key %}

<b><code>GET https://oidc.<i>[e2e/prd]</i>.itsme.services/clientsecret-oidc/csapi/v0.1/connect/authorize</code></b>

### Parameters

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="client_id" req="REQUIRED" %}</td>
      <td>It identifies your application. This parameter value is generated during registration.</td>
    </tr>
     <tr>
      <td>{% include parameter.html name="response_type" req="REQUIRED" %}</td>
      <td>This defines the processing flow to be used when forming the response. Because itsme® supports the Authorization Code Flow, this value MUST be <code>code</code>.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="scope" req="REQUIRED" %}</td>
      <td>
        It allows your application to express the desired scope of the access request. Each scope returns a set of user attributes. The scopes an application should request depend on which user attributes your application needs. Once the user authorizes the requested scopes, his details are returned in an ID Token and are also available through the UserInfo Endpoint.<br><br>All scope values must be space-separated.<br><br>The basic (and required) scopes is <code>openid</code> and <code>service</code>. Beyond that, your application can ask for additional standard scopes values which map to sets of related claims : <code>profile</code> <code>email</code> <code>address</code> <code>phone</code><br />
        <table>
          <tr>
            <td>{% include parameter.html name="service" req="REQUIRED" %}</td><td>It indicates the itsme® service your application intends to use, e.g. <code>service:TEST_code</code> by replacing "TEST_code" with the service code generated during registration.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="openid" req="REQUIRED" %}</td><td>It indicates that your application intends to use the OpenID Connect protocol to verify a user's identity by returning a <code>sub</code> claim which represents a unique identifier for the authenticated user.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="profile" req="OPTIONAL" %}</td><td>Returns claims that represent basic profile information, specifically <code>family_name</code>, <code>given_name</code>, <code>name</code>, <code>gender</code> and <code>birthdate</code>.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="email" req="OPTIONAL" %}</td><td>Returns the <code>email</code> claim, which contains the user's email address, and <code>email_verified</code>, which is a boolean indicating whether the email address was verified by the user.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="address" req="OPTIONAL" %}</td><td>Returns the information about the user's postal address. The value of the address member is a JSON structure containing some or all of these members <code>formatted</code> <code>street_address</code> <code>postal_code</code> <code>locality</code> <code>country</code></td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone" req="OPTIONAL" %}</td><td>Returns the <code>phone</code> claim, which contains the user's phone number, and <code>phone_number_verified</code>, which is a boolean indicating whether the phone number was verified by the user.</td>
          </tr> 
        </table>
      </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="redirect_uri" req="REQUIRED" %}</td>
      <td>It is the URL to which users are redirected once the authentication is complete. It MUST match the value preregistered during the registration and use the HTTPS scheme. http://localhost is only permitted for development purposes, it’s not for use in production.<br><br><b>Note</b> : if relevant (in case you have a mobile app) make sure that your redirect URIs support the Universal links and App links mechanism. Functionally, it will allow you to have only one single link that will either open your desktop web application, your mobile app or your mobile site on the User’s device. Universal links and App links are standard web links (http://mydomain.com) that point to both a web page and a piece of content inside an app. When a Universal Link is opened, the app OS checks to see if any installed app is registered for that domain. If so, the app is launched immediately without ever loading the web page. If not, the web URL is loaded into the webbrowser.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="state" req="Strongly RECOMMENDED" %}</td>
      <td>Specifies any string value that your application uses to maintain state between your Authorization Request and the Authorization Server's response. You can use this parameter for several purposes, such as directing the user to the correct resource in your application and mitigating cross-site request forgery.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="nonce" req="Strongly RECOMMENDED" %}</td>
      <td>A string value used to associate a session with an ID Token, and to mitigate replay attacks. The value is passed through unmodified from the Authentication Request to the ID Token.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="claims" req="OPTIONAL" %}</td>
      <td>Requests specific user's details to be returned from the UserInfo Endpoint and/or in the ID Token. It is represented as a JSON object that has two members - <code>{"userinfo":{...}</code> and <code>{"id_token":{...}</code>, which content indicates which claims to return at the UserInfo Endpoint and which with the ID Token, together with indication whether the claim is voluntary (default) or essential.<br><br>Possible user's details your application can request is listed below.<br />
        <table>
          <tr>
            <td>{% include parameter.html name="name" req="OPTIONAL" %}</td><td>User's full name in displayable form including all name parts, possibly including titles and suffixes</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="given_name" req="OPTIONAL" %}</td><td>Given name(s) or first name(s) of the user. Note that in some cultures, people can have multiple given names; all can be present, with the names being separated by space characters.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="family_name" req="OPTIONAL" %}</td><td>Surname(s) or last name(s) of the user. Note that in some cultures, people can have multiple family names or no family name; all can be present, with the names being separated by space characters.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="birtdate" req="OPTIONAL" %}</td><td>User's birthday, represented as a YYYY-MM-DD format.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/birthdate_as_string" req="OPTIONAL" %}</td><td>User's birthday, represented as a string.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="gender" req="OPTIONAL" %}</td><td>User's gender. Possible values are : <code>F</code> <code>M</code></td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="email" req="OPTIONAL" %}</td><td>User's email address.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="email_verified" req="OPTIONAL" %}</td><td><code>true</code> if the user's e-mail address is verified; otherwise <code>false</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone_number" req="OPTIONAL" %}</td><td>User's phone number, in <a href="https://en.wikipedia.org/wiki/E.164" target="blank">E.164</a> format.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="phone_number_verified" req="OPTIONAL" %}</td><td><code>true</code> if the user's e-phone number is verified; otherwise <code>false</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="address" req="OPTIONAL" %}</td><td>Returns the information about the user's postal address. The value of the address is a JSON structure containing some or all of these members <code>formatted</code> <code>street_address</code> <code>postal_code</code> <code>locality</code> <code>country</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_citizenship" req="OPTIONAL" %}</td><td>The jurisdiction that has conferred citizenship rights to the user, in <a href="https://en.wikipedia.org/wiki/ISO_3166" target="blank">ISO 3166</a> format.</td>
          </tr> 
           <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/place_of_birth" req="OPTIONAL" %}</td><td>The location where the user was born. The value of this attribute is a JSON structure containing some or all of these members <code>formatted</code> <code>city</code> <code>country</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/physical_person_photo" req="OPTIONAL" %}</td><td>XXX</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/BEeidSn" req="OPTIONAL" %}</td><td>Returns the Belgian ID card number, 12 digits in the form xxx-xxxxxxx-yy. The check-number yy is the remainder of the division of xxxxxxxxxx by 97.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_device" req="OPTIONAL" %}</td><td>Returns information about the user's phone. The value of this attribute is a JSON structure containing some or all of these members <code>os</code> <code>appName</code> <code>appRelease</code> <code>deviceLabel</code> <code>debugEnabled</code> <code>deviceID</code>	<code>osRelease</code> <code>manufacturer</code> <code>hasSimEnabled</code>	<code>deviceLockLevel</code> <code>smsEnabled</code> <code>rooted</code> <code>imei</code> <code>deviceModel</code>	<code>sdkRelease</code>.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/transaction_info" req="OPTIONAL" %}</td><td>Returns information about the itsme® transaction. The value of this attribute is a JSON structure containing some or all of these members <code>securityLevel</code> <code>bindLevel</code> <code>appRelease</code>.</td>
           </tr> 
           <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/BENationalNumber" req="OPTIONAL" %}</td><td>Returns an unique identification number of natural persons who are registered in Belgium. This number consists of 11 digits of the form yy.mm.dd-xxx.cd where yy.mm.dd is the birthdate of the person, xxx a sequential number (odd for males and even for females) and cd a check-digit.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_luxtrust_ssn" req="OPTIONAL" %}</td><td>Returns an unique identification number of natural persons who are registered via Luxtrust. This number consists of 20 digits.</td>
          </tr> 
          <tr>
            <td>{% include parameter.html name="http://itsme.services/v2/<br>claim/claim_nl_bsn" req="OPTIONAL" %}</td><td>Returns the citizen service number, a unique registration number for everyone who lives in the Netherlands. This number consists of 8 to 9 digits.</td>
          </tr> 
        </table>
      </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="request_uri" req="OPTIONAL" %}</td>
      <td>A URL using the https scheme referencing a resource containing a JWT whose claims are the request parameters. The <code>request_uri</code> parameter is used to secure parameters in the authentication request from tainting or inspection when sending the request to the itsme® Authorization Endpoint.<br><br>If used, the JWT MUST be signed.<!--This is the only difference regarding JWKSet text--></td>
    </tr>
  </tbody>
</table>

<a id="AuthNResp"></a>
### Response

<code>302</code> <code>application/json</code>

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="code" req="" %}</td>
      <td>An intermediate opaque credential used to retrieve the ID Token and Access Token.<br><br><b>Note</b> : the code has a lifetime of 3 minutes.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="state" req="" %}</td>
      <td>The string value provided in the Authorization Request. You SHOULD validate that the value returned matches the one supplied.</td>
    </tr>
  </tbody>
</table>

{% endtab %}

{% endtabs %}


### Example

{% tabs AuthorizationExample %}

{% tab AuthorizationExample RSA keys %}

**Request**

```http
GET /authorize HTTP/1.1
Host: server.example.com

response_type=code
  &client_id=s6BhdRkqt3
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &scope=openid%20service:TEST_code%20profile%20email
  &nonce=n-0S6_WzA2Mj
  &state=af0ifjsldkj
  &acr_values=http://itsme.services/V2/claim/acr_basic
```

**Response**

```http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  code=SplxlOBeZQQYbYS6WxSbIA
  &state=af0ifjsldkj
```

{% endtab %}

{% tab AuthorizationExample Symmetric key %}


**Request**

```http
GET /authorize HTTP/1.1
Host: server.example.com

response_type=code
  &client_id=s6BhdRkqt3
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &scope=openid%20service:TEST_code%20profile%20email
  &nonce=n-0S6_WzA2Mj
  &state=af0ifjsldkj
  &acr_values=http://itsme.services/V2/claim/acr_basic
```

**Response**

```http
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  code=SplxlOBeZQQYbYS6WxSbIA
  &state=af0ifjsldkj
```

{% endtab %}

{% endtabs %}


<a id="TokenReq"></a>
## Token Request

{% tabs TokenRequest %}

{% tab TokenRequest RSA keys %}

<b><code>POST https://idp.<i>[e2e/prd]</i>.itsme.services/v2/token</code></b>

To assert the identity of the user, the <code>code</code> received previously needs to be exchanged for an ID token and access token. During this step, your application has to authenticate itself to our server using RSA keys. 

### Parameters

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="grant_type" req="REQUIRED" %}</td>
      <td>Set this to <code>authorization_code</code> to tell the Token Endpoint that your application wants to exchange an authorization code for an ID koken and access token. </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="code" req="REQUIRED" %}</td>
      <td>The intermediate opaque credential received in the Authorization Response.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="redirect_uri" req="REQUIRED" %}</td>
      <td>It is the URL to which users are redirected once the authentication is complete. It MUST match the value used in the Authorization Request.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="client_assertion_type" req="REQUIRED" %}</td>
      <td>Specifies the type of assertion. Set this to <code>urn:ietf:params:oauth:client-assertion-type:jwt-bearer</code>.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="client_assertion" req="REQUIRED" %}</td>
      <td>Contains a JWT token that is signed, and then encrypted, using the client RSA keys. This ensures that the request to get the id token and access token is made only from your application, and not from a potential attacker that may have intercepted the authorization code.<br><br>The JWT contains the following claims.<br />       
        <table>
          <tr>
            <td>{% include parameter.html name="iss" req="REQUIRED" %}</td><td>The issuer of the token. This value must be the same as the <code>client_id</code>.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="sub" req="REQUIRED" %}</td><td>The subject of the token. This value must be the same as the <code>client_id</code>.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="aud" req="OPTIONAL" %}</td><td>The full URL of the resource you're using the JWT to authenticate to. Set this to <code>https://idp.<i>[e2e/prd]</i>.itsme.services/v2/token</code>.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="jti" req="OPTIONAL" %}</td><td>An unique identifier for the token. This parameter has a maximum length of 255 characters.</td>
          </tr>
          <tr>
            <td>{% include parameter.html name="exp" req="OPTIONAL" %}</td><td>The expiration time of the token in seconds since January 1, 1970 UTC.</td>
          </tr> 
        </table>
      </td>
    </tr>
  </tbody>
</table>


### Response

<code>200</code> <code>application/json</code>

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="access_token" req="" %}</td>
      <td>Allows an application to retrieve consented user information from the UserInfo Endpoint.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="token_type" req="" %}</td>
      <td>Provides your application with the information required to successfully utilize the access token. Returned value is <code>Bearer</code>..</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="id_token" req="" %}</td>
      <td>A security token that contains information about the authentication of an user, and potentially other requested claim data's. The <code>id_token</code> value is represented as a signed and encrypted JSON Web Token. So, before being able to use the ID Token claim's you will have to decrypt and verify it using the RSA keys.</td>      
    </tr>
  </tbody>
</table>

{% endtab %}

{% tab TokenRequest Symmetric key %}

<b><code>POST https://oidc.<i>[e2e/prd]</i>.itsme.services/clientsecret-oidc/csapi/v0.1/connect/token</code></b>

To asserts the identity of the user, the <code>code</code> received previously need to be exchanged for an ID token and access token. During this step, your application has to authenticate itself to our server using a symmetric key. 

### Parameters

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="grant_type" req="REQUIRED" %}</td>
      <td>Set this to <code>authorization_code</code> to tell the Token Endpoint that your application wants to exchange an authorization code for an ID koken and access token. </td>
    </tr>
    <tr>
      <td>{% include parameter.html name="client_id" req="REQUIRED" %}</td>
      <td>It identifies your application. This parameter value is generated during registration.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="code" req="REQUIRED" %}</td>
      <td>The intermediate opaque credential received in the Authorization Response.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="redirect_uri" req="REQUIRED" %}</td>
      <td>It is the URL to which users are redirected once the authentication is complete. It MUST match the value used in the Authorization Request.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="client_secret" req="REQUIRED" %}</td>
      <td>Contains the a key you reveiced when registering your application. This ensures that the request to get the id token and access token is made only from your application, and not from a potential attacker that may have intercepted the authorization code.</td>
    </tr>
  </tbody>
</table>


### Response

<code>200</code> <code>application/json</code>

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="access_token" req="" %}</td>
      <td>Allows an application to retrieve consented user information from the UserInfo Endpoint.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="token_type" req="" %}</td>
      <td>Provides your application with the information required to successfully utilize the access token. Returned value is <code>Bearer</code>.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="id_token" req="" %}</td>
      <td>A security token that contains information about the authentication of an user, and potentially other requested claim data's. The <code>id_token</code> value is represented as a signed JSON Web Token. So, before being able to use the ID Token claim's you will have to verify it using the symmetric key.</td>      
    </tr>
  </tbody>
</table>

{% endtab %}

{% endtabs %}


### Example

{% tabs TokenExample %}

{% tab TokenExample RSA keys %}

**Request**

```http
POST /token HTTP/1.1
Host: openid.c2id.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
 &code=SplxlOBeZQQYbYS6WxSbIA
 &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
 &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
 &client_assertion=PHNhbWxwOl ... ZT
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ.ewogImlzc
    yI6ICJodHRwOi8vc2VydmVyLmV4YW1wbGUuY29tIiwKICJzdWIiOiAiMjQ4Mjg5
    NzYxMDAxIiwKICJhdWQiOiAiczZCaGRSa3F0MyIsCiAibm9uY2UiOiAibi0wUzZ
    fV3pBMk1qIiwKICJleHAiOiAxMzExMjgxOTcwLAogImlhdCI6IDEzMTEyODA5Nz
    AKfQ.ggW8hZ1EuVLuxNuuIJKX_V8a_OMXzR0EHR9R6jgdqrOOF4daGU96Sr_P6q
    Jp6IcmD3HP99Obi1PRs-cwh3LO-p146waJ8IhehcwL7F09JdijmBqkvPeB2T9CJ
    NqeGpe-gccMg4vfKjkM8FcGvnzZUN4_KSP0aAp1tOJ1zZwgjxqGByKHiOtX7Tpd
    QyHE5lcMiKPXfEIQILVq0pc_E2DzL7emopWoaoZTF_m0_N0YzFC6g6EJbOEoRoS
    K5hoDalrcvRYLSrQAZZKflyuVCyixEoV9GfNQC3_osjzw2PAithfubEEBLuVVk4
    XUVrWOLrLl0nx7RkKU8NXNHq-rvKMzqg"
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "expires_in": 3600,
}
```

{% endtab %}

{% tab TokenExample Symmetric key %}

**Request**

```http
POST /token HTTP/1.1
Host: openid.c2id.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
 &code=SplxlOBeZQQYbYS6WxSbIA
 &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
 &client_id=s6BhdRkqt3
 &client_secret=PHNhbWxwOl ... ZT
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ.ewogImlzc
    yI6ICJodHRwOi8vc2VydmVyLmV4YW1wbGUuY29tIiwKICJzdWIiOiAiMjQ4Mjg5
    NzYxMDAxIiwKICJhdWQiOiAiczZCaGRSa3F0MyIsCiAibm9uY2UiOiAibi0wUzZ
    fV3pBMk1qIiwKICJleHAiOiAxMzExMjgxOTcwLAogImlhdCI6IDEzMTEyODA5Nz
    AKfQ.ggW8hZ1EuVLuxNuuIJKX_V8a_OMXzR0EHR9R6jgdqrOOF4daGU96Sr_P6q
    Jp6IcmD3HP99Obi1PRs-cwh3LO-p146waJ8IhehcwL7F09JdijmBqkvPeB2T9CJ
    NqeGpe-gccMg4vfKjkM8FcGvnzZUN4_KSP0aAp1tOJ1zZwgjxqGByKHiOtX7Tpd
    QyHE5lcMiKPXfEIQILVq0pc_E2DzL7emopWoaoZTF_m0_N0YzFC6g6EJbOEoRoS
    K5hoDalrcvRYLSrQAZZKflyuVCyixEoV9GfNQC3_osjzw2PAithfubEEBLuVVk4
    XUVrWOLrLl0nx7RkKU8NXNHq-rvKMzqg"
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "expires_in": 3600,
}
```

{% endtab %}

{% endtabs %}

<a id="UserInfoReq"></a>
## UserInfo Request

{% tabs UserInfoRequest %}

{% tab UserInfoRequest RSA keys %}

<b><code>GET https://idp.<i>[e2e/prd]</i>.itsme.services/v2/userinfo</code></b>

The UserInfo Endpoint returns previously consented user profile information to your application. In other words, if the required claims are not returned in the ID Token, you can obtain the additional claims by presenting the access token to the itsme® UserInfo Endpoint. This is achieved by sending a HTTP GET request to the Userinfo Endpoint, passing the access token value in the Authorization header using the Bearer authentication scheme.

This is illustrated in the example below.


### Response

The UserInfo Response is a signed and encrypted JSON Web Token. So, before being able to extract the claims you will have to decrypt and verify it using the RSA keys.


{% endtab %}

{% tab UserInfoRequest Symmetric key %}

<b><code>GET https://oidc.<i>[e2e/prd]</i>.itsme.services/clientsecret-oidc/csapi/v0.1/connect/userinfo</code></b>

The UserInfo Endpoint returns previously consented user profile information to your application. In other words, if the required claims are not returned in the ID Token, you can obtain the additional claims by presenting the access token to the itsme® UserInfo Endpoint. This is achieved by sending a HTTP GET request to the Userinfo Endpoint, passing the access token value in the Authorization header using the Bearer authentication scheme.

This is illustrated in the example below.


### Response

The UserInfo Response is a signed JSON Web Token. So, before being able to extract the claims you will have to verify it using the symmetric key.

{% endtab %}

{% endtabs %}


### Example

{% tabs UserInfoExample %}

{% tab UserInfoExample RSA keys %}

**Request**

```http
GET /userinfo HTTP/1.1
Host: server.example.com
Authorization: Bearer SlAV32hkKG
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

  {
   "sub": "248289761001",
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "preferred_username": "j.doe",
   "email": "janedoe@example.com",
   "picture": "http://example.com/janedoe/me.jpg"
  }
```

{% endtab %}

{% tab UserInfoExample Symmetric key %}

**Request**

```http
GET /userinfo HTTP/1.1
Host: server.example.com
Authorization: Bearer SlAV32hkKG
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json

  {
   "sub": "248289761001",
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "preferred_username": "j.doe",
   "email": "janedoe@example.com",
   "picture": "http://example.com/janedoe/me.jpg"
  }
```

{% endtab %}

{% endtabs %}



## Revoke Request

{% tabs RevokeRequest %}

{% tab RevokeRequest RSA keys %}

Not applicable.

{% endtab %}

{% tab RevokeRequest Symmetric keys %}

<b><code>POST https://oidc.<i>[e2e/prd]</i>.itsme.services/clientsecret-oidc/csapi/v0.1/connect/revoke</code></b>

The Revocation Endpoint enables your application to notify itsme® that a previously obtained access token is no longer needed and MUST be revoked.

 ### Parameters

<table>
  <tbody>
    <tr>
      <td>{% include parameter.html name="client_id" req="REQUIRED" %}</td>
      <td>It identifies your application. This parameter value is generated during registration.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="client_secret" req="REQUIRED" %}</td>
      <td>Contains the a key you reveiced when registering your application.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="token" req="REQUIRED" %}</td>
      <td>The <code>access_token</code> previously obtained that you want to revoke.</td>
    </tr>
    <tr>
      <td>{% include parameter.html name="token_type_hint" req="OPTIONAL" %}</td>
      <td>If used, this is set to <code>access_token</code> because itsme® API don't support refresh tokens.</td>
    </tr>
  </tbody>
</table>

### Response

The response is very simple: it’s always an HTTP 200 if the token is revoked or unknown.

{% endtab %}

{% endtabs %}

### Example

{% tabs RevokeExample %}

{% tab RevokeExample RSA keys %}

Not applicable.

{% endtab %}

{% tab RevokeExample Symmetric key %}

**Request**


**Response**

{% endtab %}

{% endtabs %}
