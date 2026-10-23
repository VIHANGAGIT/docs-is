# Add identity verification with Daon TrustX

[Daon TrustX](https://www.daon.com/){: target="_blank"} is an identity verification platform that verifies a person against a government-issued identity document such as a passport, driver's license, or national ID card, combined with facial biometrics and liveness detection.

By integrating Daon TrustX with {{ product_name }}, you can verify a user's identity against a document when their account gets created, populate or validate their profile with the verified attributes, and later re-verify the same person with a live biometric check when they sign in to a sensitive application or recover their password.

## How it works

{{ product_name }} connects to Daon TrustX as a federated OpenID Connect (OIDC) connection.

- When a user's account gets created, {{ product_name }} redirects the user to Daon TrustX, where they capture their identity document, take a selfie, and complete a liveness check. This creates an **enrollment** for the user.

- Daon TrustX returns the attributes it read from the document as OIDC verified claims. Depending on the flow, {{ product_name }} either writes these attributes to the user's profile or uses Daon TrustX to confirm that they match the profile an administrator created.

- Later, when the enrolled user signs in to an application or resets their password, {{ product_name }} redirects them to Daon TrustX again to confirm, with a live biometric check, that they are the same person who enrolled.

Daon TrustX uses two connection types in {{ product_name }}:

<table>
    <thead>
        <tr>
            <th>Connection</th>
            <th>Purpose</th>
            <th>Configuration it holds</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><b>Daon TrustX Identity Verifier</b></td>
            <td>Enrolls users during account creation. It acts as the anchor for all enrollments.</td>
            <td>OIDC client credentials, endpoints, scopes, the enroll process definition, and the attribute mappings.</td>
        </tr>
        <tr>
            <td><b>Daon TrustX Authenticator</b></td>
            <td>Re-verifies enrolled users at login and during password recovery.</td>
            <td>A reference to a Daon TrustX Identity Verifier connection and a login process definition.</td>
        </tr>
    </tbody>
</table>

A Daon TrustX Authenticator connection doesn't hold credentials of its own. It reads the OIDC configuration from the Identity Verifier connection it references and shares that connection's enrollments. This lets you create several authenticator connections, for example, one per application with a different login process definition, on top of a single set of enrollments.

![Daon TrustX integration architecture]({{base_path}}/assets/img/guides/identity-verification/daon/architecture.png)

The following table lists the supported use cases.

| Use case | Flow | Connection to use |
|---|---|---|
| [Self-registration with verified onboarding](#self-registration) | Self registration | Daon TrustX Identity Verifier |
| [Invited user registration with profile validation](#invited-user-registration) | Invited user registration | Daon TrustX Identity Verifier |
| [Step-up identity verification at login](#login-re-verification) | Application login flow | Daon TrustX Authenticator |
| [Identity verification for password recovery](#password-recovery) | Password recovery | Daon TrustX Authenticator |

!!! note "Enrollment is only supported for newly created accounts"

    Currently, a user can only enroll with Daon TrustX while their account gets created, through the self-registration or invited user registration flow. Progressive enrollment, which enrolls existing users later, for example during login, isn't supported yet.

    As a result, login re-verification and password recovery with Daon TrustX work only for users who enrolled when their account got created. Users who existed before you set up Daon TrustX, or who registered through a flow without the Daon TrustX step, can't use these flows and fail with the `DAON-60001` error.

## Prerequisites

Before you begin, ensure you have the following:

- A Daon TrustX tenant with administrator access. To get one, contact [Daon](https://www.daon.com/){: target="_blank"}.

- At least one **enroll** process definition and one **login** process definition in your Daon TrustX tenant.

- [{{ product_name }} Account](https://asgardeo.io/signup): Access to your {{ product_name }} Console with administrative privileges.

## Register an OIDC client in Daon TrustX

{{ product_name }} communicates with Daon TrustX through an OIDC client. To register one,

1. Log in to the Daon TrustX administration console.

2. Create a **confidential** OIDC client.

3. Enable the `openid`, `profile`, and `document` scopes for the client.

4. Take note of the following details:

    - The **client ID** and **client secret** of the client.
    - The **authorization endpoint** and **token endpoint** URLs.
    - The name and version of the enroll process definition and the login process definition, each in the `<ProcessDefinitionName>:<Version>` format. For example, `EnrolProcess:1` and `LoginProcess:1`.

You will add the redirect URI of the client after creating the connection in {{ product_name }}.

!!! note

    By default, {{ product_name }} sends the client credentials to Daon TrustX in the request body. If your Daon TrustX client requires HTTP Basic authentication, set the `IsBasicAuthEnabled` property of the connection to `true` using the [Identity Provider API]({{base_path}}/apis/idp/). This property isn't available on the Console.

## Create a Daon TrustX Identity Verifier connection

The Daon TrustX Identity Verifier connection enrolls users during account creation. To create one,

1. On the {{ product_name }} Console, go to **Connections** and click **New Connection**.

2. Find the **Daon TrustX Identity Verifier** card and click **Create**.

3. Enter a name for the connection and the following details:

    <table>
        <thead>
            <tr>
                <th>Field</th>
                <th>Description</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>Client ID</td>
                <td>The client ID of the OIDC client you registered in Daon TrustX.</td>
            </tr>
            <tr>
                <td>Client Secret</td>
                <td>The client secret of the OIDC client.</td>
            </tr>
            <tr>
                <td>Authorization Endpoint URL</td>
                <td>The authorization endpoint of your Daon TrustX tenant. For example, <code>https://&lt;tenant&gt;.&lt;region&gt;.trustx.com/auth/realms/&lt;tenant&gt;/protocol/openid-connect/auth</code></td>
            </tr>
            <tr>
                <td>Token Endpoint URL</td>
                <td>The token endpoint of your Daon TrustX tenant. For example, <code>https://&lt;tenant&gt;.&lt;region&gt;.trustx.com/auth/realms/&lt;tenant&gt;/protocol/openid-connect/token</code></td>
            </tr>
            <tr>
                <td>Scopes</td>
                <td>The scopes to request. Use <code>openid profile document</code>.</td>
            </tr>
            <tr>
                <td>Enroll Process Definition</td>
                <td>The enroll process definition in the <code>&lt;ProcessDefinitionName&gt;:&lt;Version&gt;</code> format. {{ product_name }} sends it as the <code>acr_values</code> parameter on every enrollment.</td>
            </tr>
        </tbody>
    </table>

4. Click **Create**.

5. Go to the **Settings** tab of the created connection and copy the **Authorized redirect URI**.

6. Return to the Daon TrustX administration console and add the copied URI as a redirect URI of the OIDC client. If you use Daon TrustX in registration or password recovery flows hosted on your own portal, add the callback URL of that portal as well.

!!! warning "Create only one Identity Verifier connection"

    An enrollment belongs to the Identity Verifier connection that the user enrolled through. A user enrolled through one Identity Verifier connection can only be verified by Daon TrustX Authenticator connections that reference the same Identity Verifier. There is no sharing or migration of enrollments between Identity Verifier connections.

    In most cases, create exactly one Identity Verifier connection and create as many Daon TrustX Authenticator connections as you need on top of it. Create a second Identity Verifier only if you need two separate groups of enrolled users, for example, for a different Daon TrustX tenant or OIDC client.

## Map the Daon TrustX attributes

The attribute mappings of the Identity Verifier connection decide which attributes {{ product_name }} requests from Daon TrustX and how it stores the verified attributes that Daon TrustX returns.

1. Open the created **Daon TrustX Identity Verifier** connection and go to its **Attributes** tab.

2. Click **Add Attribute Mapping**, enter the Daon TrustX claim name, and select the corresponding {{ product_name }} user attribute.

    ![Daon TrustX attribute mappings]({{base_path}}/assets/img/guides/identity-verification/daon/attribute-mapping.png){: width="800" style="display: block; margin: 0;"}

3. Click **Update** to save the changes.

When a user enrolls, {{ product_name }} handles each mapped attribute as follows:

- If the flow already knows the attribute's value, for example, because the user entered it earlier in the registration flow or an administrator set it on the user's profile, {{ product_name }} sends the value to Daon TrustX to compare against the document.

- If the flow doesn't know the attribute's value yet, {{ product_name }} asks Daon TrustX to read it from the document and return it.

{{ product_name }} never sends unmapped attributes to Daon TrustX. If the connection has no attribute mappings, {{ product_name }} doesn't request any verified claims.

Keep the following in mind when you map attributes:

- {{ product_name }} only reads attribute mappings from the Identity Verifier connection. It ignores the **Attributes** tab of Daon TrustX Authenticator connections.

- The following claims can be verified against a document: `given_name`, `family_name`, `family_name_and_given_name`, `birthdate`, `document_number`, and `document_personal_number`. The invited user registration flow requires at least one of these to be mapped and set on the user's profile. Email, phone number, and address alone don't satisfy this requirement.

- When you map `given_name` or `family_name`, {{ product_name }} also requests `family_name_and_given_name` for documents that store the full name in a single field. It splits this value on the `^` character in the ICAO 9303 order, `<family name>^<given name>`.

- {{ product_name }} always requests the document claims `document_type`, `document_classification`, `document_date_of_expiry`, `document_number`, and `document_personal_number`, but only stores them on the user's profile if you map them.

- {{ product_name }} stores the `formatted` value of the Daon TrustX `address` claim. Other claims with a JSON object value are stored as a raw JSON string.

## Create a Daon TrustX Authenticator connection

The Daon TrustX Authenticator connection re-verifies enrolled users at login and during password recovery. To create one,

1. On the {{ product_name }} Console, go to **Connections** and click **New Connection**.

2. Find the **Daon TrustX Authenticator** card and click **Create**.

3. Enter a name for the connection and the following details:

    - **Daon Identity Verifier**: Select the Daon TrustX Identity Verifier connection you created earlier.
    - **Login Process Definition**: The login process definition in the `<ProcessDefinitionName>:<Version>` format. {{ product_name }} sends it as the `acr_values` parameter on every re-verification.

4. Click **Create**.

You can create as many authenticator connections as you need, for example, one per application with a different login process definition. All of them use the OIDC configuration and share the enrollments of the Identity Verifier connection they reference.

!!! note

    - A Daon TrustX Authenticator connection and the Identity Verifier connection it references must belong to the same organization.
    - Users enrolled through a different Identity Verifier connection can't be verified by this connection.

## Add Daon TrustX to your flows

The following table summarizes where to add each connection.

| Flow | Connection to add | Placement |
|---|---|---|
| Self registration | Daon TrustX Identity Verifier | After the attribute collection step, before the password step. |
| Invited user registration | Daon TrustX Identity Verifier | Before the set password step. |
| Application login | Daon TrustX Authenticator | A step after the user gets identified, for example, step 2. |
| Password recovery | Daon TrustX Authenticator | Where the identity check belongs in the recovery flow. |

In the flow builder, Daon TrustX appears as the **Daon TrustX Verification** component.

### Self-registration

Use this flow to onboard a new user whose profile gets populated from a verified identity document instead of from the values they typed in.

![Self-registration flow with Daon TrustX]({{base_path}}/assets/img/guides/identity-verification/daon/self-registration-flow-diagram.png)

To configure the flow,

1. On the {{ product_name }} Console, go to **Flows** and select **Self Registration**.

2. Add the **Daon TrustX Verification** component after the step that collects the user's attributes and before the password step.

3. Select your **Daon TrustX Identity Verifier** connection for the component.

    ![Self-registration flow with the Daon TrustX Verification component]({{base_path}}/assets/img/guides/identity-verification/daon/self-registration-flow-builder.png)

4. Save and enable the flow.

When a user registers, the following happens:

1. {{ product_name }} redirects the user to Daon TrustX with the enroll process definition. For each mapped attribute, it either sends the value the user already entered for Daon TrustX to compare against the document, or asks Daon TrustX to read the value from the document.

2. The user captures their identity document, takes a selfie, and completes the liveness check.

3. Daon TrustX redirects the user back to {{ product_name }} with the verified claims.

4. {{ product_name }} overwrites the attributes the user entered with the values read from the document and links the user's account to their Daon TrustX enrollment.

!!! note

    - Placement matters. {{ product_name }} only sends an attribute for comparison if it's both mapped on the Identity Verifier connection and collected before the Daon TrustX step runs.
    - Values read from the document replace any mapped attribute the user entered. Users can't correct a misread document field during registration.

### Invited user registration

Use this flow when an administrator creates a user account and fills in the profile. Daon TrustX proves that the person accepting the invitation is the person the profile describes.

![Invited user registration flow with Daon TrustX]({{base_path}}/assets/img/guides/identity-verification/daon/invited-user-registration-flow-diagram.png)

To configure the flow,

1. On the {{ product_name }} Console, go to **Flows** and select **Invited User Registration**.

2. Add the **Daon TrustX Verification** component before the set password step.

3. Select your **Daon TrustX Identity Verifier** connection for the component.

    ![Invited user registration flow with the Daon TrustX Verification component]({{base_path}}/assets/img/guides/identity-verification/daon/invited-user-registration-flow-builder.png)

4. Save and enable the flow.

When an invited user accepts the invitation, the following happens:

1. {{ product_name }} reads the mapped attributes from the profile the administrator created and sends them to Daon TrustX for comparison against the document.

2. The user captures their identity document, takes a selfie, and completes the liveness check.

3. Daon TrustX compares the profile attributes against the document. If they match, the user proceeds to set their password and {{ product_name }} links the account to the user's Daon TrustX enrollment.

!!! note

    - This flow validates the profile but doesn't update it. The profile stays as the administrator created it.
    - At least one document-verifiable attribute (`given_name`, `family_name`, `family_name_and_given_name`, `birthdate`, `document_number`, or `document_personal_number`) must be mapped and set on the user's profile. Otherwise, the step doesn't run and fails with the `DAON-65023` error.
    - Daon TrustX performs the comparison. If an attribute doesn't match, the step fails with the `DAON-60003` error, but doesn't indicate which attribute failed.

### Login re-verification

Use this flow to step up an enrolled user with a live biometric check before they access a sensitive application.

![Login re-verification with Daon TrustX]({{base_path}}/assets/img/guides/identity-verification/daon/login-flow-diagram.png)

To configure the login flow,

1. On the {{ product_name }} Console, go to **Applications** and select your application.

2. Go to its **Login Flow** tab.

3. Keep a first step that identifies the user, such as **Username & Password**, and click **+** to add a second step.

4. Add your **Daon TrustX Authenticator** connection to the second step.

    ![Login flow with Daon TrustX as the second step]({{base_path}}/assets/img/guides/identity-verification/daon/login-flow.png)

5. Click **Update** to save the changes.

When a user signs in, the following happens:

1. After the first step identifies the user, {{ product_name }} looks up the user's Daon TrustX enrollment and redirects the user to Daon TrustX with the login process definition.

2. The user completes the biometric check.

3. {{ product_name }} confirms that the user Daon TrustX verified is the same user it requested. This prevents a user from passing the step by verifying their own identity for another account.

!!! note

    - The Daon TrustX step can't be the first step of the login flow. It's a step-up mechanism, not a login mechanism.
    - The login step doesn't enroll users. Users who haven't enrolled fail with the `DAON-60001` error and see the login retry page.
    - {{ product_name }} only records that a user is enrolled. It doesn't track when the verification happened or how long it's valid, so you need to enforce policies such as "verified within the last 12 months" outside {{ product_name }}.

### Password recovery

Use this flow to prove that the person resetting the password is the enrolled account holder.

![Password recovery flow with Daon TrustX]({{base_path}}/assets/img/guides/identity-verification/daon/password-recovery-flow-diagram.png)

To configure the flow,

1. On the {{ product_name }} Console, go to **Flows** and select **Password Recovery**.

2. Add the **Daon TrustX Verification** component where you want the identity check to happen.

3. Select your **Daon TrustX Authenticator** connection for the component.

    ![Password recovery flow with the Daon TrustX Verification component]({{base_path}}/assets/img/guides/identity-verification/daon/password-recovery-flow-builder.png)

4. Save and enable the flow.

You can also offer more than one Daon TrustX Authenticator connection in the same flow. In the following example, after the emailed link, the user chooses between a push notification and a face verification, and each option routes to its own Daon TrustX Verification component.

![Password recovery flow with two Daon TrustX options]({{base_path}}/assets/img/guides/identity-verification/daon/password-recovery-flow-builder-multi-step.png)

When a user recovers their password, the following happens:

1. {{ product_name }} looks up the account's Daon TrustX enrollment and redirects the user to Daon TrustX with the login process definition.

2. The user completes the biometric check.

3. {{ product_name }} confirms that the user Daon TrustX verified is the account holder and lets the user reset their password.

!!! note

    - This flow is an identity check only. {{ product_name }} doesn't read or update any profile attributes.
    - Accounts that haven't enrolled, or that enrolled through a different Identity Verifier connection, fail immediately with the `DAON-60001` error without being redirected to Daon TrustX.

## Reference

The following sections provide more details on how {{ product_name }} works with Daon TrustX.

### Enrollment

An enrollment links a {{ product_name }} user to their Daon TrustX identity. When a user completes the Daon TrustX step in the self-registration or invited user registration flow, {{ product_name }} stores the Daon TrustX `preferred_username` as a federated association of the user, against the Identity Verifier connection. You can see this association under **Linked accounts** in the user's My Account portal.

Login re-verification and password recovery use this association to identify the user at Daon TrustX.

### Authorization request parameters

{{ product_name }} extends the standard OIDC authorization request with the following parameters.

| Parameter | Value | When it's sent |
|---|---|---|
| `acr_values` | The enroll or login process definition, in the `<ProcessDefinitionName>:<Version>` format. | Always, when a process definition is configured on the connection. |
| `claims` | An OIDC `verified_claims` request under the `daon-identify-1` trust framework. Mapped attributes with a known value are sent as value requests for Daon TrustX to compare, and the rest are requested from the document. | During enrollment, in the self-registration and invited user registration flows. |
| `login_hint` | The Daon TrustX `preferred_username` stored on the user's enrollment. | During re-verification, in the login and password recovery flows. |

{{ product_name }} handles the token exchange, `state`, `nonce`, PKCE, ID token validation, and standard claim mapping in the same way as any other OIDC connection.

### Flow behavior

The following table summarizes how each supported flow behaves.

| Flow | Process definition | Sends `claims` | Sends `login_hint` | Updates the profile |
|---|---|---|---|---|
| Self registration | Enroll | Yes | No | Yes, with the values read from the document. |
| Invited user registration | Enroll | Yes | No | No, Daon TrustX only validates the profile. |
| Application login | Login | No | Yes | No |
| Password recovery | Login | No | Yes | No |

### Limitations

- **Enrollment only for newly created accounts.** Users can only enroll during self-registration or invited user registration. Progressive enrollment of existing users isn't supported yet.

- **Supported flow types.** The **Daon TrustX Verification** component is only available in the self-registration, invited user registration, and password recovery flows.

- **Results on redirect only.** {{ product_name }} receives the verification result when Daon TrustX redirects the user back. It doesn't support webhooks or polling, so it can't pick up a Daon TrustX process that completes after the redirect.

- **No logout.** {{ product_name }} doesn't support OIDC logout or single logout with Daon TrustX.

- **Fixed trust framework.** {{ product_name }} always uses the `daon-identify-1` trust framework. A response with a different trust framework fails with the `DAON-65022` error.

- **One process definition per connection.** To use a different login process definition per application, create a separate Daon TrustX Authenticator connection for each application.

- **No verification lifetime.** {{ product_name }} only records that a user enrolled. It doesn't store when the verification happened, its assurance level, or the type of document used.

- **Attribute values in the request URL.** {{ product_name }} sends value requests as query parameters of the authorization request. Values containing `&`, `=`, `${`, or `$authparam{` are dropped from the request or cause it to fail with the `DAON-65029` error. Large sets of attributes can also exceed URL length limits at a proxy or at Daon TrustX, since pushed authorization requests (PAR) aren't supported.

- **Name order.** {{ product_name }} splits the `family_name_and_given_name` claim assuming the ICAO 9303 order, `<family name>^<given name>`. Documents that use the reverse order result in swapped names.

### Error codes

The following are the common errors you may encounter.

| Error code | Description |
|---|---|
| `DAON-60001` | The user hasn't enrolled with Daon TrustX, or enrolled through a different Identity Verifier connection. |
| `DAON-60003` | The attribute values sent to Daon TrustX for comparison didn't match the document. |
| `DAON-65022` | The verified claims in the Daon TrustX response use an unexpected trust framework. |
| `DAON-65023` | None of the mapped attributes on the invited user's profile can be verified against a document. |
| `DAON-65029` | A process definition, enrollment identifier, or attribute value contains characters that can't be sent in the authorization request. |
