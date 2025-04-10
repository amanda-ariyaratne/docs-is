# Build your own Push Notification Authenticator Application

A push authenticator application should have the following capabilities:

1. **Register a device**: The user should be able to enroll their device with the authentication server. This involves a set of actions as mentioned below.

    1. **Scan the QR code**: The application should be able to scan the QR code displayed on the authentication server to retrieve the necessary information to register the device.
    2. **Generate a key pair**: The application should be able to generate a public and private key and securely store the private key on the device along with the user information received from the QR code.
    3. **Invoke the registration API**: The application should be able to invoke the registration API of the authentication server with the signed user information, device information, and the public key.

2. **Receive push notifications**: The application should be able to receive push notifications sent through the push notification service and display the authentication request information to the user.

3. **Allow the user to approve or deny the authentication request**: The application should allow the user to approve or deny the authentication request received through the push notification.

4. **Invoke the authentication API**: The application should be able to invoke the authentication API of the authentication server with the signed authentication response.

{{product_name}} provides the necessary APIs to build your own push authenticator application. The below given guide explains how to build a push authenticator application using the APIs
provided by {{product_name}}.


## Register push notification device

For the users to receive push notifications during an authentication flow, they need to register their devices with the {{product_name}}.

### **Step 1**: Scan the QR code

The registration process starts with the user scanning a QR code that contains the necessary information to register the device with the IAM system. This QR code
can be accessed from the MyAccount portal or when push device progressive enrollment is enabled for an authentication flow of an application.

!!! note
    Learn more about enrolling push notification devices [via My Account](../../guides/user-self-service/register-push-notification-device.md) 
    and [progressive enrollment](../../guides/authentication/mfa/add-push-auth-login.md#enable-push-notification-device-progressive-enrollment).

The content encoded in the QR code is retrieved from the response of the push device registration discovery data API endpoint. 
This response contains the following information:

<table>
     <tr>
       <th style="width: 350px;">Attributes</th>
       <th>Description</th>
     </tr>
     <tr>
       <td><code>deviceId</code></td>
       <td>A unique identifier generated by {{product_name}} to identify the device.</td>
     </tr>
     <tr>
       <td><code>username</code></td>
       <td>The username of the user trying to register a device.</td>
     </tr>
     <tr>
       <td><code>host</code></td>
       <td>The host of the {{product_name}} server.</td>
     </tr>
     <tr>
       <td><code>tenantDomain</code></td>
       <td>The tenant domain of the root organization user.</td>
     </tr>
     <tr>
       <td><code>organizationId</code></td>
       <td>Organization ID of an organization user.</td>
     </tr>
     <tr>
       <td><code>organizationName</code></td>
       <td>Organization name of an organization user.</td>
     </tr>
     <tr>
       <td><code>challenge</code></td>
       <td>A UUID to be signed by the push authentication application</td>
     </tr>
</table>

!!! note
    - For a root organization user (tenant user), only the **tenantDomain** will be available in the response.
    - For an organization user, only **organizationId** and **organizationName** will be available in the response.

From above response, information such as the **device ID**, **tenant domain**, **organization ID**, and the **username** should be extracted and stored on the device.

!!! note
    It is advised to store the private key securely against the device ID since, during the authentication process, the device ID will be sent in the notification data
    and it can be used to retrieve the private key to sign the authentication response.

!!! important
    Since the push authenticator app is not expected to maintain any user sessions, there won't be any access tokens involved in the registration process. If you are building
    a push authenticator app which also maintains user sessions, you can use the access token with relevant permissions to invoke the push device registration discovery data.
    With that, you can avoid the overhead of scanning the QR code and extracting the information from the QR code.

### **Step 2**: Generate push authenticator app unique identifier

Push notifications are sent to a unique device of a user during the authentication process. These notifications are sent through different push notification services. 
These services identify the device using a **unique identifier**. Since, the IAM system triggers the push notification through the 
push notification service, the device unique identifier should be registered with the IAM system.

{{product_name}} supports Firebase Cloud Messaging (FCM) to send push notifications. Firebase identifies the devices using a unique identifier called the **registration token**.
This token will be generated by Firebase and assigned to the specific app instance in the device to uniquely identify it.

!!! note
    Learn more about [Firebase Cloud Messaging Architecture](https://firebase.google.com/docs/cloud-messaging/fcm-architecture){:target="_blank"}.

### **Step 3**: Generate public and private key pair

Since push notification based authentication leverages asymmetric cryptography to secure the authentication process,
during the registration process, the device should generate a **public and private key** and securely store the private key on the device.
The private key is used to sign the authentication response during the authentication process and the public key is used by the IAM system 
to verify the signed authentication response.

{{product_name}} supports public and private key pairs generated using the **RSA algorithm** with a key size of **2048 bits**. 
In order to send the public key in the registration request, it should be converted to a **PEM format**, **headers and footers should be removed**, 
and the content should be **base64 encoded**.

### **Step 4**: Generate the signature for registration verification

In order to validate whether the registration request is coming from a legitimate device, the device should sign the challenge received from the IAM system and
send it along with the registration request. The signature should be generated using the private key generated during the registration process.

{{product_name}} expects the device to generate the signature with the following steps:

1. Retrieve the value of the **challenge** attribute from the QR code data.
2. Retrieve the **device token** for the app instance from the push notification service.
3. Concatenate the challenge and the device token as a single string, using . as the separator, with no spaces. (Example: challenge.deviceToken).
4. Sign the concatenated string using the **RSA private key** that was generated during the registration process. Use **PKCS#1 v1.5 padding** and **SHA-256 hashing** before encryption.
5. Encode the signature in **base64** format.

### **Step 5**: Invoke the registration API

As the final step of the registration process, the device should invoke the registration API of the {{product_name}} with the following information:

<table>
     <tr>
       <th style="width: 150px;">Key</th>
       <th>Description</th>
     </tr>
     <tr>
       <td><code>deviceId</code></td>
       <td>A unique identifier generated by {{product_name}} to identify the device (From Step 1).</td>
     </tr>
     <tr>
       <td><code>name</code></td>
       <td>Name of the registering device.</td>
     </tr>
     <tr>
       <td><code>model</code></td>
       <td>Model of the registering device.</td>
     </tr>
     <tr>
       <td><code>deviceToken</code></td>
       <td>The unique identifier token generated by the notification provider service for the app instance (From Step 2).</td>
     </tr>
     <tr>
       <td><code>publicKey</code></td>
       <td>Base64 encoded, header and footer less PEM value of the public key generated by the registering device (From Step 3).</td>
     </tr>
     <tr>
       <td><code>signature</code></td>
       <td>Base64 encoded, hashed and encrypted string value signed by the private key from the registering device (From Step 4).</td>
     </tr>
</table>

Based on the registration discovery data response, the push authenticator app should conditionally invoke the relevant
registration API endpoint with a **tenanted path** or an **organization path**.

If response contains the **tenant domain**, that means it is a **root organization user**. Hence, the registration API should be invoked with the **tenanted path** as shown below.

    {{baseUrl}}/t/{tenant-domain}/api/users/v1/me/push/devices/

If response contains the **organization ID**, that means it is an **organization user**, the registration API should be invoked with the **organization path**.

    {{baseUrl}}/o/{organization-id}/api/users/v1/me/push/devices/

The below given is a sample request payload to be sent to the registration API.

    {
      "deviceId": <device unique identifier>,
      "model": <device model>,
      "name": <device name>,
      "deviceToken": <device unique identifier token>,
      "publicKey": <base64 encoded public key>,
      "signature": <base64 encoded signature>
    }

Upon successful registration, the registration request will return a **201 Created** response.


## Receive push notifications

During a push notification based authentication flow, {{product_name}} builds the data required for the authentication and send it as a push notification through
the push notification service. The push authenticator app should be able to receive these push notifications and display the authentication request information to the user.
The data sent through the push notification contains the following information:

<table>
     <tr>
       <th style="width: 200px;">Attribute</th>
       <th>Description</th>
     </tr>
     <tr>
       <td><code>username</code></td>
       <td>Username of the user trying to authenticate.</td>
     </tr>
     <tr>
       <td><code>tenantDomain</code></td>
       <td>Tenant domain of the root organization user (tenant user).</td>
     </tr>
     <tr>
       <td><code>organizationId</code></td>
       <td>Organization ID of the organization user.</td>
     </tr>
     <tr>
       <td><code>organizationName</code></td>
       <td>Organization Name of the organization user.</td>
     </tr>
     <tr>
       <td><code>userStoreDomain</code></td>
       <td>Userstore domain of the user.</td>
     </tr>
     <tr>
       <td><code>deviceId</code></td>
       <td>The deviceId generated by {{product_name}} during device registration.</td>
     </tr>
     <tr>
       <td><code>applicationName</code></td>
       <td>Name of the application user is trying to authenticate.</td>
     </tr>
     <tr>
       <td><code>notificationScenario</code></td>
       <td>The scenario purpose of the push notification (Example: AUTHENTICATION)</td>
     </tr>
     <tr>
       <td><code>pushId</code></td>
       <td>Unique identifier to identify the push authentication flow related to the push notification.</td>
     </tr>
     <tr>
       <td><code>challenge</code></td>
       <td>A UUID value that has to be sent back with the authentication request for verification.</td>
     </tr>
     <tr>
       <td><code>numberChallenge</code></td>
       <td>A number between 1 and 100 generated to add additional security over the challenge.</td>
     </tr>
     <tr>
       <td><code>ipAddress</code></td>
       <td>IP address of the user trying to authenticate.</td>
     </tr>
     <tr>
       <td><code>deviceOS</code></td>
       <td>Operating system of the device.</td>
     </tr>
     <tr>
       <td><code>browser</code></td>
       <td>Browser used by the user.</td>
     </tr>
</table>

!!! note
    - For a root organization user (tenant user), only the **tenantDomain** will be available in the push notification data.
    - For an organization user, only **organizationId** and **organizationName** will be available in the push notification data.

With the above information, the push authenticator app should display the authentication request information to the user.
The user should be able to approve or deny the authentication request based on the information displayed.


## Invoke the authentication API

The push authenticate API endpoint has to be invoked to send an authentication response to the {{product_name}} server from the push authenticator application. 
With this request, the {{product_name}} server will validate the authentication response and complete the authentication flow.

The authentication from the push authenticator app should be in the form of a JWT token signed with the private key generated during the registration process.

The JWT token should contain the following claims in the header:

<table>
     <tr>
       <th style="width: 200px;">claim</th>
       <th>Description</th>
     </tr>
     <tr>
       <td><code>deviceId</code></td>
       <td>Unique identifier of the device</td>
     </tr>
</table>

The JWT token should contain the following claims in the payload:

<table>
     <tr>
       <th style="width: 200px;">claim</th>
       <th>Description</th>
     </tr>
     <tr>
       <td><code>pushAuthId</code></td>
       <td>Unique identifier of the ongoing push authentication flow sent in the notification data</td>
     </tr>
     <tr>
       <td><code>challenge</code></td>
       <td>UUID string sent as challenge in the notification data</td>
     </tr>
     <tr>
       <td><code>response</code></td>
       <td>Response of the user for the authentication request. (Either APPROVED or DENIED)</td>
     </tr>
     <tr>
       <td><code>numberChallenge</code> (Optional)</td>
       <td>Number provided by the user to validate the authentication request.  (If number challenge is enabled)</td>
     </tr>
     <tr>
       <td><code>exp</code> (Expiry Time)</td>
       <td>Expiry time of the token. (Max. 10 minutes from generated time)</td>
     </tr>
</table>

According to the stored user information, the push authenticator app should conditionally invoke the relevant API authenticate endpoint with a **tenanted path** or an **organization path**.

If the user information contains the **tenant domain**, the authentication API should be invoked with the **tenanted path** as shown below.

    {{baseUrl}}/t/{tenant-domain}/push-auth/authenticate

If the user information contains the **organization ID**, the authentication API should be invoked with the **organization path**.

    {{baseUrl}}/o/{organization-id}/push-auth/authenticate

The below given is a sample request payload to be sent to the authentication API.

    {
      "authResponse": <JWT token>
    }

Once the authentication request is sent successfully, the push authenticator app will receive a **202 Accepted** response. With that, the authentication flow
will be continued and the user will be authenticated based on the response sent by the push authenticator app.
