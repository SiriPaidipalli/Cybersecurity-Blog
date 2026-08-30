# HTTP & HTTPS: Understanding Web Traffic From a Security Perspective

The web is built around conversations between systems. A browser requests a page, a mobile application retrieves account information, an API receives data, or a cloud service sends a response. Behind each of these actions is a structured exchange that tells one system what another system wants and how that request was handled.

HTTP provides the rules for those exchanges. HTTPS protects them while they travel across a network.

For cybersecurity, understanding this communication is fundamental because web activity appears almost everywhere: application security, API security, network monitoring, proxy logs, web server logs, authentication systems, incident investigations, vulnerability testing, cloud environments, and security tools. A request can contain user input, authentication tokens, cookies, file uploads, API data, or instructions to modify a resource. A response can reveal whether access was allowed, whether an error occurred, whether a session was created, or how a browser should treat the returned content.

The goal is therefore not simply to remember that HTTP uses port 80 and HTTPS commonly uses port 443. The useful skill is being able to look at web communication and understand what was requested, what information was supplied, what the server returned, which security controls were involved, and what conclusions the available evidence actually supports.

## The Client-Server Model

HTTP follows a request-response model. A client sends an HTTP request to a server, and the server processes that request and returns an HTTP response.

```text
Client                                                  Server

HTTP Request       ───────────────────────▶

                   ◀───────────────────────           HTTP Response
```

A web browser is a common HTTP client, but it is far from the only one. Mobile applications, command-line utilities, scripts, APIs, software agents, cloud services, vulnerability scanners, and many other programs communicate using HTTP.

The server may be a traditional web server, an API gateway, a cloud application, a reverse proxy, or another service capable of processing HTTP requests. Behind that server there may be databases, authentication services, internal APIs, storage systems, and additional application components.

A single action in a browser can therefore generate many requests. Loading one webpage may require separate requests for the HTML document, stylesheets, JavaScript files, images, fonts, API responses, advertisements, and analytics resources.

What looks like one webpage to a user may actually be dozens or hundreds of individual HTTP exchanges.

## Where HTTP Fits in Network Communication

HTTP is an **application-layer protocol**. It defines the structure and meaning of web requests and responses, but it does not independently handle routing or physical network delivery.

For traditional HTTP/1.1 communication, the relationship can be simplified as:

```text
      HTTP
       ↓
      TCP
       ↓
       IP
       ↓
Network Interface
```

Each layer has a different responsibility. HTTP describes the application message, TCP provides reliable transport, IP provides logical addressing and routing, and the network interface handles communication on the local network medium.

HTTPS adds **Transport Layer Security**, or TLS, to protect the HTTP communication. HTTP/2 also commonly operates through TLS over TCP, while HTTP/3 uses a different architecture based on QUIC over UDP.

The distinction matters because seeing TCP traffic does not automatically mean HTTP is being used, and seeing communication with a commonly associated web port does not by itself prove which application protocol is running there. Ports and protocols provide evidence, but service identification should always consider the actual communication.

## Understanding a URL

A Uniform Resource Locator, or URL, tells a client where a resource is located and how it should be accessed.

Consider:

```text
https://shop.example.com:443/products/view?id=42#reviews
```

The URL can be divided into several components:

```text
Scheme:      https
Host:        shop.example.com
Port:        443
Path:        /products/view
Query:       id=42
Fragment:    reviews
```

The **scheme** identifies how the resource should be accessed. For web traffic, the most familiar schemes are `http` and `https`.

The **host** identifies the destination by name. DNS may be used to resolve that hostname to an IP address before network communication begins.

The **port** identifies the destination service. When the standard port is used, browsers usually do not display it explicitly. HTTP commonly uses TCP port 80 and HTTPS commonly uses TCP port 443, although neither protocol is restricted to those ports.

The **path** identifies a resource or application route.

The **query string** begins after `?` and contains parameters supplied to the server. In this example, `id=42` is a query parameter.

The **fragment** begins after `#`. It is normally interpreted locally by the browser and is not included in the HTTP request sent to the server.

This distinction is useful during security analysis because not every value visible in a browser's address bar necessarily reaches the server.

## Anatomy of an HTTP Request

An HTTP request tells the server what the client wants to do.

A simplified HTTP/1.1 request might look like:

```http
GET /account/profile?id=42 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: session=<session-value>
```

The first line is the **request line**:

```text
GET /account/profile?id=42 HTTP/1.1
```

It identifies the HTTP method, request target, and protocol version.

The remaining lines are **request headers**, which provide additional information about the request and the client.

Some requests also contain a **message body**, particularly when data is being submitted.

The general structure is:

```text
Request Line
Request Headers

Optional Request Body
```

This structure is one of the most important things to understand about web communication. Web applications ultimately receive structured input from clients, and security decisions must be made using that input safely.

## HTTP Methods

The HTTP method communicates the intended operation associated with a request. Different methods have different semantics, although the server ultimately decides how each route behaves.

### GET

`GET` requests retrieve a representation of a resource.

```http
GET /products/42 HTTP/1.1
Host: example.com
```

Browsers generate GET requests constantly for pages, images, scripts, stylesheets, API resources, and other content.

GET is intended to be a safe method, meaning it should not cause a requested state change merely by retrieving a resource. Applications do not always follow that design correctly, which can create security and reliability problems.

### POST

`POST` submits data to a server for processing.

```http
POST /login HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

username=alice&password=<value>
```

POST is frequently used for authentication, form submissions, file uploads, transactions, and API operations, but the method itself does not imply any particular business action.

### PUT

`PUT` commonly creates or replaces the representation of a resource at a known location.

```http
PUT /api/users/42 HTTP/1.1
```

### PATCH

`PATCH` applies a partial modification to a resource rather than replacing the entire representation.

```http
PATCH /api/users/42 HTTP/1.1
```

### DELETE

`DELETE` requests removal of a resource.

```http
DELETE /api/users/42 HTTP/1.1
```

### HEAD

`HEAD` requests the same response headers that a GET request would normally return but without the response body. It can be useful when a client wants information about a resource without downloading the resource itself.

### OPTIONS

`OPTIONS` asks which communication options are available for a resource. Browsers also use OPTIONS for certain CORS preflight requests before making cross-origin requests.

HTTP defines additional methods, including `CONNECT` and `TRACE`, but GET, POST, PUT, PATCH, DELETE, HEAD, and OPTIONS cover much of the traffic encountered in ordinary web applications and APIs.

The method is not an authorization mechanism. A DELETE request is not automatically dangerous, and a GET request is not automatically safe. The application must decide whether the authenticated identity is permitted to perform the requested operation.

## Parameters and Client-Controlled Input

Applications receive input from several locations within an HTTP request. Query parameters are one common source.

For example:

```text
https://example.com/search?q=security&page=2
```

contains:

```text
q=security
page=2
```

Applications use parameters for searches, filters, identifiers, pagination, sorting, configuration, and many other purposes.

Path components can also contain user-controlled values:

```text
/users/42
/orders/9812
/files/report.pdf
```

A value appearing in a URL does not become trustworthy because the application's interface generated it. A user can modify requests directly.

Suppose an application normally requests:

```text
GET /api/orders/1001
```

Changing the request to:

```text
GET /api/orders/1002
```

may be trivial for the client. The server must independently determine whether the requesting identity is authorized to access order `1002`.

This principle extends beyond URLs. Headers, cookies, JSON fields, form values, uploaded files, and other request components can all originate from an untrusted client.

## Request Bodies and Content Types

HTTP requests can carry data inside a message body.

Traditional HTML forms may submit URL-encoded data:

```text
username=alice&email=alice@example.com
```

Modern APIs frequently use JSON:

```json
{
  "username": "alice",
  "role": "user"
}
```

File uploads commonly use `multipart/form-data`, which allows multiple fields and files to be included in one request.

The `Content-Type` header describes how the body should be interpreted. Common values include:

```text
application/json
application/x-www-form-urlencoded
multipart/form-data
text/plain
text/html
application/xml
```

The server needs to parse this input correctly and validate it before using it. Client-side validation can improve usability, but it is not a security boundary because an attacker does not have to use the legitimate browser interface.

## Important Request Headers

HTTP headers carry metadata that can influence how a request is processed.

The `Host` header identifies the hostname being requested:

```http
Host: example.com
```

This allows several websites to share the same server IP address while still being routed to different applications.

The `User-Agent` header contains information the client chooses to provide about itself:

```http
User-Agent: Mozilla/5.0 ...
```

Because a client can change this value, it should not be treated as proof of identity or device type.

The `Accept` header describes content types the client can process:

```http
Accept: application/json
```

The `Content-Type` header describes the body being sent:

```http
Content-Type: application/json
```

Authentication information may appear in an `Authorization` header:

```http
Authorization: Bearer <token>
```

Browsers can send stored cookies using:

```http
Cookie: session=<session-value>
```

Other headers may carry language preferences, caching information, origin information, referrer data, forwarding information, and application-specific values.

Headers are part of the request and should be analyzed with the same caution as parameters and body data. A header supplied by an external client should not automatically be trusted simply because legitimate browsers normally generate it.

## Anatomy of an HTTP Response

After processing a request, the server returns an HTTP response.

A simplified response might look like:

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256
Cache-Control: no-cache

<html>
...
</html>
```

The first line is the **status line**. It contains the HTTP version, numerical status code, and a textual reason phrase in HTTP/1.1.

The headers provide metadata and instructions about the response, while the optional body contains the returned content.

The structure is therefore:

```text
Status Line
Response Headers

Optional Response Body
```

The body may contain HTML, JSON, JavaScript, an image, an archive, a document, an error message, or another type of data.

## HTTP Status Codes

Status codes provide a standardized way for servers to describe the result of a request. They are divided into five classes based on the first digit.

### 1xx: Informational

Informational responses indicate that processing is continuing or communicate intermediate protocol information.

One example is:

```text
101 Switching Protocols
```

which can appear when the connection transitions to another protocol, such as during a WebSocket upgrade.

### 2xx: Success

These responses indicate successful processing.

Common examples include:

```text
200 OK
201 Created
202 Accepted
204 No Content
```

`200 OK` indicates a successful response in the general case. `201 Created` commonly follows creation of a resource. `202 Accepted` indicates that a request has been accepted for processing but may not yet be complete. `204 No Content` indicates successful processing without a response body.

### 3xx: Redirection

Redirection responses tell the client that another location or action is involved.

Common examples include:

```text
301 Moved Permanently
302 Found
304 Not Modified
307 Temporary Redirect
308 Permanent Redirect
```

`304 Not Modified` is related to caching rather than ordinary navigation. `307` and `308` preserve the original request method during redirection, which distinguishes them from some historical behavior associated with `301` and `302`.

### 4xx: Client Error

These indicate that the server could not fulfill the request because of something associated with the request.

Common examples include:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
408 Request Timeout
409 Conflict
413 Content Too Large
415 Unsupported Media Type
429 Too Many Requests
```

Despite the wording, `401 Unauthorized` generally relates to missing or unsuccessful authentication. `403 Forbidden` usually means the server understood the request but refuses to authorize it.

### 5xx: Server Error

These indicate failures while the server or an upstream component was processing the request.

Common examples include:

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

Status codes are extremely useful, but applications can use them inconsistently. A login page might return `200 OK` while displaying an authentication failure inside the response body. Security analysis should therefore consider the response behavior rather than treating the numerical code as a complete description of what happened.

## Response Headers

Response headers tell the client how the returned content should be interpreted and can also control important browser behavior.

`Content-Type` identifies the type of returned content:

```http
Content-Type: application/json
```

`Content-Length` may specify the length of the response body:

```http
Content-Length: 1256
```

`Location` identifies a redirect destination:

```http
Location: /login
```

`Cache-Control` controls how content may be cached.

`Set-Cookie` tells a browser to create or update a cookie:

```http
Set-Cookie: session=<value>; Secure; HttpOnly; SameSite=Lax
```

Other response headers implement browser-side security controls, including HSTS, Content Security Policy, Referrer Policy, and MIME-type protections.

## HTTP Is Stateless

HTTP does not automatically remember a user between independent requests. This property is commonly described as **statelessness**.

A web application still needs to remember information such as whether a user has authenticated, which account is being used, or what items are stored in a shopping cart. Applications therefore add mechanisms for maintaining state.

One common approach is session-based authentication. After successful authentication, the server creates session state and provides the browser with an identifier.

```text
            Login Request
                 ↓
       Authentication Succeeds
                 ↓
       Server Creates Session
                 ↓
  Browser Receives Session Identifier
                 ↓
Identifier Is Sent With Later Requests
```

The server can associate the identifier with information stored about the authenticated session.

The identifier itself becomes sensitive because possession of a valid session value may allow another party to act within that session.

## Cookies

Cookies are small pieces of data that a server can instruct a browser to store.

A response may contain:

```http
Set-Cookie: session=abc123
```

The browser can later return the cookie:

```http
Cookie: session=abc123
```

Cookies have many uses, including authentication sessions, preferences, shopping carts, analytics, and tracking.

Several attributes influence cookie security.

The **Secure** attribute instructs the browser to send the cookie only over secure HTTPS connections.

The **HttpOnly** attribute prevents ordinary client-side JavaScript from accessing the cookie through interfaces such as `document.cookie`. This can reduce certain session-token theft opportunities if cross-site scripting occurs, although it does not fix the underlying XSS vulnerability.

The **SameSite** attribute influences whether a cookie is included with cross-site requests. Common values are `Strict`, `Lax`, and `None`, and modern browsers generally require `SameSite=None` cookies to also use `Secure`.

Other useful attributes include `Domain`, which controls the domains to which a cookie applies, `Path`, which limits where it is sent, and `Expires` or `Max-Age`, which influence its lifetime.

Session security also depends on properties beyond cookie attributes. Applications need appropriate session expiration, logout behavior, token rotation where applicable, protection against fixation, and secure handling of session identifiers throughout their lifecycle.

## Authentication and Authorization

Authentication and authorization solve different security problems.

**Authentication** establishes an identity. A password, certificate, security key, one-time code, biometric-backed credential, or another mechanism may participate in determining who is making a request.

**Authorization** determines what that authenticated identity is permitted to access or modify.

A user can therefore authenticate successfully and still receive:

```text
403 Forbidden
```

when attempting to access a resource outside their permissions.

This distinction is critical because many application vulnerabilities are authorization failures rather than authentication failures. An application may correctly determine who a user is but fail to verify whether that user should be allowed to access another account's data, administrative functionality, or a particular API object.

## Tokens and Bearer Authentication

APIs and modern web applications frequently use tokens for authentication and authorization.

A request might contain:

```http
Authorization: Bearer <token>
```

A bearer token is security-sensitive because possession of a valid token may be enough to use the privileges associated with it.

Tokens can have expiration times, scopes, audiences, signatures, claims, and other properties depending on the technology being used.

One common token format is the **JSON Web Token**, or JWT. A JWT contains encoded sections representing a header, payload, and signature:

```text
header.payload.signature
```

A JWT is not automatically encrypted. Its payload may be readable by anyone who obtains the token unless separate encryption is used. The signature is intended to protect integrity and authenticity according to the token's cryptographic design.

Applications must validate tokens correctly rather than merely decoding them. Relevant checks can include the signature, expiration, issuer, audience, and other claims required by the application's security model.

## Why Plain HTTP Is Insecure for Sensitive Communication

HTTP by itself does not provide confidentiality or cryptographic integrity for application content.

If an observer has sufficient visibility into plaintext HTTP traffic, information such as request paths, query parameters, headers, cookies, form data, response content, and authentication material may be readable.

That does not mean every person on the Internet can automatically observe every HTTP request. Network visibility depends on the path, infrastructure, and observation point. The problem is that HTTP itself provides no cryptographic protection against an observer who does have access to the traffic.

Plain HTTP also lacks cryptographic protection against undetected modification of the application messages while they are in transit.

Modern web communication therefore relies heavily on HTTPS.

## HTTPS and TLS

HTTPS is HTTP communication protected by **Transport Layer Security**.

TLS is designed to provide three major properties: confidentiality, integrity, and authentication.

**Confidentiality** prevents an ordinary passive observer without the necessary cryptographic secrets from reading protected application data.

**Integrity** allows endpoints to detect unauthorized modification of protected traffic.

**Authentication** allows the client to verify the server identity represented by its certificate when certificate validation succeeds.

The result is that HTTP requests and responses can travel across untrusted networks without exposing their plaintext content to ordinary passive observers.

The protection applies to the communication between the TLS endpoints. Once data reaches an endpoint and is decrypted, the endpoint and application can access it normally.

## The TLS Handshake

Before protected application data can be exchanged, the client and server establish the cryptographic context for the TLS connection.

In modern TLS, the client begins with a **ClientHello**. This message advertises information needed for negotiation, including supported TLS versions, cryptographic capabilities, and protocol extensions.

The server replies with a **ServerHello** selecting compatible parameters. Certificate-related information allows the server to authenticate itself, while a cryptographic key exchange allows both endpoints to derive the secrets used to protect the session.

After the handshake completes successfully, encrypted application data can be exchanged.

The details differ between TLS versions, particularly between TLS 1.2 and TLS 1.3. TLS 1.3 simplified the protocol, removed obsolete cryptographic mechanisms, improved handshake efficiency, and encrypts more handshake information than older versions.

Modern deployments primarily use TLS 1.2 and TLS 1.3. SSL and early TLS versions are obsolete and should not be treated as acceptable modern protection, even though people still casually use expressions such as “SSL certificate.”

## Symmetric and Asymmetric Cryptography in TLS

TLS uses different forms of cryptography for different purposes.

**Asymmetric cryptography** uses a mathematically related public and private key pair. Certificates contain public-key information, while the corresponding private key remains protected by its owner.

Modern TLS key exchange mechanisms allow the endpoints to establish shared secrets securely.

Once the connection is established, **symmetric encryption** protects the bulk application data because symmetric cryptography is efficient for large amounts of traffic.

This means HTTPS should not be understood as simply “the server encrypts everything with its public key.” TLS combines several cryptographic mechanisms to provide authentication, key establishment, confidentiality, and integrity.

## Certificates and Server Identity

A TLS certificate binds public-key information to an identity under a certificate trust model.

A server certificate can contain information including:

```text
Subject
Subject Alternative Names
Issuer
Validity Period
Public Key
Signature Algorithm
Certificate Signature
```

For websites, the **Subject Alternative Name**, or SAN, extension is particularly important because it identifies hostnames for which the certificate is valid.

When a browser connects to a website, it does not simply check whether a certificate exists. It validates several properties, including whether the certificate is valid for the requested hostname, whether it is within its validity period, whether its signatures validate, and whether it can be linked through a valid certificate chain to a trusted authority.

A certificate can therefore be cryptographically valid but still fail hostname validation if it was issued for a different domain.

## Certificate Authorities and the Chain of Trust

Browsers and operating systems maintain stores of trusted **Certificate Authorities**, or CAs.

A website certificate is commonly linked to a trusted root through one or more intermediate certificates:

```text
Trusted Root CA
      ↓
Intermediate CA
      ↓
Server Certificate
```

The root CA is trusted by the client through its trust store. The intermediate CA certificate is signed by another authority in the chain, and the server certificate is signed by the intermediate CA.

The client validates the chain until it reaches a root it already trusts.

Certificate authorities therefore provide the trust infrastructure that allows a browser to determine whether a certificate presented by a server can be trusted for the claimed identity.

Certificate revocation mechanisms such as **Certificate Revocation Lists**, or CRLs, and the **Online Certificate Status Protocol**, or OCSP, can also provide information about certificates that should no longer be trusted before their normal expiration date.

## A Valid Certificate Does Not Mean a Safe Website

A certificate answers a specific question about authenticated TLS identity. It does not provide a general security rating for the application.

A malicious website can obtain a legitimate certificate for a domain controlled by the attacker. A vulnerable application can have a perfectly configured certificate. A compromised server can continue using HTTPS.

The browser's HTTPS indicator therefore should not be interpreted as proof that a website is trustworthy, free from vulnerabilities, or safe to interact with.

It indicates that the browser established a TLS-protected connection whose certificate satisfied the applicable validation requirements.

## What HTTPS Protects and What It Leaves Exposed

Inside a correctly established HTTPS connection, HTTP content such as paths, query parameters, headers, cookies, request bodies, response bodies, and authentication information is protected from ordinary passive network observation.

The surrounding network communication still exposes some metadata because packets must reach their destination. Depending on the protocols and configuration involved, an observer may still learn source and destination IP addresses, connection timing, packet sizes, traffic volume, transport behavior, and connection duration.

Some TLS handshake information can also remain visible. Historically, **Server Name Indication**, or SNI, commonly exposed the requested hostname in the ClientHello. **Encrypted Client Hello**, or ECH, is designed to protect more of this handshake information when supported.

Domain visibility may also come from DNS if traditional plaintext DNS is used. DNS over HTTPS and DNS over TLS can protect DNS queries from ordinary network observation between their respective endpoints.

HTTPS therefore protects application content in transit without making the entire network interaction invisible.

## Security Problems HTTPS Cannot Solve

Transport encryption cannot compensate for insecure application logic.

An HTTPS application can still contain SQL injection, cross-site scripting, broken access control, server-side request forgery, insecure file handling, weak authentication, poor session management, vulnerable dependencies, or business-logic flaws.

Phishing sites can use HTTPS. Malware can communicate over HTTPS. Stolen credentials can be submitted through HTTPS. An attacker exploiting a vulnerable API can do so over a correctly encrypted connection.

The correct security boundary is therefore clear: TLS protects the communication channel, while the application must protect the operations and data processed through that channel.

## Security Headers

Browsers support several HTTP response headers that allow servers to request additional security behavior.

### HTTP Strict Transport Security

**HTTP Strict Transport Security**, or HSTS, tells compatible browsers to access a site using HTTPS for a specified period.

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

This helps prevent users from being downgraded to plaintext HTTP after the browser has learned the HSTS policy. Browsers can also include certain domains in preload lists so that HTTPS enforcement is known before the first connection.

### Content Security Policy

**Content Security Policy**, or CSP, controls which sources a browser is permitted to use for scripts, styles, images, frames, and other resources.

A simplified policy might look like:

```http
Content-Security-Policy: default-src 'self'
```

CSP can significantly reduce the impact of some content-injection attacks, particularly certain cross-site scripting scenarios, but it should complement rather than replace secure output handling.

### X-Content-Type-Options

A server can send:

```http
X-Content-Type-Options: nosniff
```

to instruct browsers not to perform certain MIME-type guessing behavior and instead respect the declared content type.

### Referrer-Policy

`Referrer-Policy` controls how much referrer information browsers include when making subsequent requests.

### Permissions-Policy

`Permissions-Policy` can restrict access to browser capabilities such as the camera, microphone, geolocation, and other features for the page and embedded content.

These headers illustrate an important characteristic of web security: the server can use HTTP not only to return content but also to tell the browser how that content should be handled.

## Same-Origin Policy

Browsers execute content from many websites, sometimes simultaneously. Without isolation between those websites, a malicious page could potentially read information from another website where the user is authenticated.

The **same-origin policy** provides an important browser security boundary.

An origin is generally defined by the combination of:

```text
Scheme
Host
Port
```

For example:

```text
https://example.com
```

and:

```text
https://api.example.com
```

are different origins because the hosts differ.

Likewise:

```text
http://example.com
```

and:

```text
https://example.com
```

have different origins because the schemes differ.

The same-origin policy restricts how documents and scripts from one origin can interact with resources belonging to another.

## Cross-Origin Resource Sharing

Legitimate applications often need resources from another origin, so browsers implement **Cross-Origin Resource Sharing**, or CORS.

A server can explicitly indicate which origin may access a resource:

```http
Access-Control-Allow-Origin: https://example.com
```

Certain cross-origin operations trigger a **preflight request**. The browser sends an OPTIONS request containing information about the intended operation, and the server responds with the permissions it is willing to grant.

CORS controls what browser-based JavaScript is permitted to read or send under the browser's cross-origin security model. It is not a replacement for authentication or authorization.

An API remains responsible for verifying whether a requester is permitted to access sensitive data regardless of its CORS configuration.

## CSRF and Automatically Sent Credentials

Cookies create an important browser behavior: the browser may automatically attach them to requests that satisfy the cookie's rules.

This behavior can contribute to **Cross-Site Request Forgery**, or CSRF.

In a CSRF scenario, a victim is authenticated to an application, and another site attempts to cause the victim's browser to submit an unwanted request to that application. If the target application relies only on automatically attached credentials and cannot distinguish the unwanted request from a legitimate one, an unauthorized action may occur in the victim's session.

Defenses can include anti-CSRF tokens, appropriate SameSite cookie settings, origin validation, and application-specific controls.

CSRF is different from cross-site scripting. CSRF abuses the browser's ability to send authenticated requests, while XSS involves executing attacker-controlled script in the security context of a website.

## Caching and Sensitive Information

HTTP supports caching to improve performance and reduce unnecessary transfers.

Relevant headers include:

```text
Cache-Control
Expires
ETag
Last-Modified
```

A client can sometimes ask whether a previously downloaded resource has changed rather than downloading the entire resource again. This can lead to responses such as:

```text
304 Not Modified
```

Caching also has security implications. Sensitive content should not be stored in inappropriate shared caches or retained longer than intended.

Applications handling confidential data therefore need caching policies appropriate to the sensitivity of their responses.

## MIME Types and Content Interpretation

HTTP uses the `Content-Type` header to identify the format of message content.

Examples include:

```text
text/html
text/css
application/javascript
application/json
image/png
application/pdf
```

Browsers process these formats differently. HTML may be rendered as a document, JavaScript may be executed, JSON may be processed as structured data, and images may be decoded and displayed.

Incorrect content-type handling can create security issues when a browser interprets data differently from what the server intended.

MIME handling is therefore more than a formatting detail. It influences how returned data is treated by the client.

## Redirects

HTTP redirects tell a client to retrieve another location.

For example:

```http
HTTP/1.1 302 Found
Location: /login
```

Browsers use redirects for authentication flows, navigation, moved resources, URL normalization, and many other legitimate purposes.

If an application allows an untrusted user to control the destination without sufficient validation, it may create an **open redirect**. An attacker could then construct a URL on a trusted domain that redirects a victim to an attacker-controlled location.

The vulnerability is not the redirect mechanism itself. It is the unsafe handling of the destination.

## Sensitive Information in URLs

Sensitive values require careful handling even when HTTPS is used.

Consider:

```text
https://example.com/reset?token=<secret>
```

HTTPS protects the request target from ordinary passive network observers while it travels inside the encrypted connection. The value can still appear elsewhere after being processed, including browser history, application logs, reverse proxy logs, analytics systems, monitoring platforms, debugging output, or screenshots.

This is why sensitive information should not be placed unnecessarily in URLs. Encryption in transit protects one stage of the data lifecycle, not every location where the data may later be stored or displayed.

## Reverse Proxies, Load Balancers, and Web Architecture

The server processing application logic is not always the first system that receives an HTTP request.

Modern applications frequently use architectures such as:

```text
          Client
            ↓
           CDN
            ↓
Load Balancer / Reverse Proxy
            ↓
      Web Application
            ↓
   Application Services
            ↓
         Database
```

A **reverse proxy** accepts requests on behalf of backend servers. It may perform routing, TLS termination, caching, authentication integration, compression, filtering, or other functions.

A **load balancer** distributes requests among multiple backend systems.

A **Content Delivery Network**, or CDN, can serve cached content from geographically distributed locations and may also provide security and traffic-management services.

Understanding these intermediaries matters because the IP address observed by the application may belong to a proxy rather than the original client.

## Forwarded Client Information

Proxies may add headers containing information about the original request.

Examples include:

```text
Forwarded
X-Forwarded-For
X-Forwarded-Proto
X-Forwarded-Host
```

A header might contain:

```http
X-Forwarded-For: 203.0.113.25
```

indicating an original client address.

These headers require explicit trust boundaries. If an application blindly trusts forwarding headers supplied directly by arbitrary Internet clients, an attacker may be able to spoof information used for logging, rate limiting, URL generation, or security decisions.

Applications should know which proxies are trusted and accept forwarded information accordingly.

## TLS Termination and HTTPS Inspection

TLS does not always terminate directly on the application server.

A load balancer or reverse proxy may terminate TLS and forward the resulting HTTP request to an internal application service. Security devices can also perform authorized HTTPS inspection.

An enterprise inspection architecture may conceptually look like:

```text
   Client
     ↓
TLS Connection
     ↓
Security Proxy
     ↓
Separate TLS Connection
     ↓
External Server
```

Managed client systems may trust an organizational certificate authority that allows the proxy to establish trusted TLS connections with those clients. The proxy can then inspect the HTTP communication according to organizational policy before creating another encrypted connection toward the destination.

The proxy becomes part of the trust boundary because it has access to the decrypted application traffic.

## HTTP Message Boundaries

HTTP receivers need to determine where one message ends and another begins.

HTTP/1.1 can use mechanisms such as `Content-Length`:

```http
Content-Length: 1256
```

to indicate the body length.

HTTP/1.1 has also used chunked transfer encoding:

```http
Transfer-Encoding: chunked
```

to transmit a body as a sequence of chunks.

Message framing becomes security-sensitive when multiple systems process the same traffic. A reverse proxy and backend server need to agree about where each request begins and ends.

If different components interpret ambiguous message boundaries differently, unexpected requests can reach the backend.

## HTTP Request Smuggling

**HTTP request smuggling** exploits inconsistencies in how different HTTP-processing systems determine message boundaries.

Consider an architecture:

```text
    Client
      ↓
Front-End Proxy
      ↓
Back-End Server
```

If the proxy interprets the length of a request differently from the backend, specially constructed traffic can cause part of one request to be interpreted as the beginning of another.

The detailed exploitation techniques are more advanced, but the underlying security lesson is broadly useful: protocol correctness matters at every layer, especially when multiple components independently parse the same data.

## Host Header Security

Virtual hosting makes the `Host` header important for routing.

Two requests reaching the same IP address might contain:

```http
Host: shop.example.com
```

and:

```http
Host: admin.example.com
```

and be directed to different applications.

If an application assumes that the Host value is trustworthy, manipulated Host headers can sometimes affect password-reset links, generated URLs, routing, caching, or other application behavior.

The general rule is the same as with other request fields: data originating from the client must be validated before it influences a security-sensitive decision.

## Web Application Firewalls

A **Web Application Firewall**, or WAF, examines HTTP traffic and applies security rules before requests reach an application or before responses reach a client.

Depending on its design, a WAF may evaluate:

```text
Methods
Paths
Query parameters
Headers
Request bodies
Response characteristics
Known attack patterns
Request rates
IP reputation
Protocol anomalies
```

For HTTPS traffic, inspection must occur at a point where the HTTP content is available in decrypted form.

WAFs can block, rate-limit, challenge, or log suspicious requests, but they do not eliminate vulnerabilities from the underlying application. They provide an additional defensive layer rather than replacing secure development and proper authorization.

## HTTP/1.1

HTTP/1.1 uses a textual message format that makes requests and responses relatively easy for humans to inspect.

It also supports persistent connections, allowing multiple HTTP interactions to use the same TCP connection instead of creating a new TCP connection for every resource.

Despite its age, HTTP/1.1 remains important because its message structure provides the conceptual foundation for understanding HTTP semantics.

## HTTP/2

HTTP/2 preserves concepts such as methods, paths, headers, status codes, and bodies but changes how those messages are represented on the wire.

Instead of the plain textual framing associated with HTTP/1.1, HTTP/2 uses binary frames. It can multiplex multiple independent streams over one TCP connection, allowing several requests and responses to progress without requiring a separate TCP connection for each one.

HTTP/2 also compresses headers using HPACK and introduces features designed to improve efficiency.

The application-level meaning of HTTP remains familiar even though the network representation changes.

## HTTP/3 and QUIC

HTTP/3 moves away from TCP and uses **QUIC over UDP**.

QUIC integrates capabilities that previously depended on separate TCP and TLS behavior and supports multiple independent streams while reducing some delays associated with connection establishment and TCP-level head-of-line blocking.

HTTP/3 traffic commonly uses UDP port 443.

This is important during network analysis because web traffic on port 443 should not automatically be assumed to use TCP. Modern environments may contain both TCP-based HTTPS using HTTP/1.1 or HTTP/2 and UDP-based HTTP/3 traffic.

## APIs Are HTTP Applications Too

An API may not display a traditional webpage, but its communication often uses the same HTTP concepts.

A request might look like:

```http
GET /api/users/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer <token>
Accept: application/json
```

and the response may contain:

```json
{
  "id": 42,
  "username": "alice"
}
```

API security therefore depends on familiar controls: authentication, object-level and function-level authorization, input validation, secure token handling, TLS, rate controls, safe error handling, and appropriate logging.

The absence of a graphical browser interface does not reduce the importance of web security. APIs expose application functionality directly through structured requests and can contain the same classes of security weaknesses as browser-facing applications.

## Common Web Vulnerabilities Through HTTP

Once requests and responses are understood, many web vulnerabilities become easier to reason about because the vulnerability often involves how the application handles information supplied through HTTP.

### Broken Access Control

An authenticated user changes a resource identifier and gains access to another user's data because the server fails to verify authorization.

The weakness is not that the identifier could be changed. Clients are always capable of changing their own requests. The weakness is the missing server-side access-control decision.

### Injection

User-controlled input reaches an interpreter in an unsafe form. Depending on the affected technology, this can include SQL injection, command injection, LDAP injection, template injection, and other variants.

HTTP carries the input to the application, while the vulnerability occurs when the application uses that input unsafely.

### Cross-Site Scripting

Cross-site scripting, or XSS, occurs when attacker-controlled content reaches a browser in a context where it can execute as script.

Different forms of XSS can involve server responses, stored application data, or client-side JavaScript behavior.

The core problem is an unsafe transition from untrusted data to executable browser content.

### Cross-Site Request Forgery

CSRF abuses authenticated browser behavior to cause unwanted requests on behalf of a user. It becomes particularly relevant when credentials such as cookies are attached automatically.

### Server-Side Request Forgery

**Server-Side Request Forgery**, or SSRF, occurs when an application can be manipulated into making unintended network requests.

An application might legitimately retrieve a URL supplied by a user. Without appropriate restrictions, an attacker may attempt to make it contact internal services, cloud metadata endpoints, localhost services, or other destinations that the attacker cannot reach directly.

### File Upload Vulnerabilities

Applications that accept uploaded files need to consider file type validation, storage location, filename handling, permissions, content processing, and whether uploaded content can become executable or publicly accessible.

### Path Traversal

If user input influences filesystem paths without adequate controls, an attacker may attempt to access files outside the intended directory using path manipulation.

### Insecure Deserialization and Application Logic

Some vulnerabilities occur after HTTP input reaches application frameworks or business logic. The HTTP request may appear structurally valid while the application handles the contained data in a dangerous way.

Understanding the protocol therefore helps identify where attacker-controlled information enters the system, while secure application design determines whether that input becomes harmful.

## HTTP Logs as Security Evidence

Web servers, reverse proxies, API gateways, WAFs, cloud platforms, and applications frequently record HTTP activity.

A simplified access log might contain:

```text
10.0.0.25 - - [time] "GET /login HTTP/1.1" 200
```

Depending on configuration, logs can include source addresses, timestamps, methods, paths, status codes, response sizes, user agents, referrers, request duration, hostnames, and other fields.

A sequence such as:

```text
POST /login     401
POST /login     401
POST /login     401
POST /login     200
GET  /account   200
```

deserves investigation, but it does not explain itself. It could represent a legitimate user mistyping a password, password guessing, credential stuffing, automated testing, or another workflow.

An analyst would consider the frequency of attempts, account involved, source information, MFA events, historical behavior, surrounding authentication telemetry, and activity following the successful-looking response.

The same principle applies to status codes. A `200` response does not necessarily mean a login succeeded, because some applications return a successful HTTP response containing an application-level error message.

## Reading an HTTP Event Properly

When analyzing a request, several questions help establish what actually happened. Which client sent it? Which host and path were requested? What method was used? Which parameters, headers, cookies, or body values were supplied? Was authentication material present? Which parts of the request were controlled by the client, and does the request fit normal application behavior?

The response adds another layer of evidence. The status code shows how the server categorized the result, while response headers can reveal redirects, cookies, caching instructions, security controls, and content types. The response body may provide application-specific context that the status code alone cannot provide.

A sequence of requests is usually more informative than one isolated event. Authentication attempts, session creation, resource access, errors, redirects, API operations, and logout activity can form a timeline that explains how an interaction developed.

Strong analysis separates what was **observed** from what was **inferred**. A request to TCP port 443 does not prove which webpage was viewed. A DNS lookup does not prove that the returned address was contacted. A `200` response does not always prove that an application operation succeeded. A valid session cookie does not identify the human who used it.

HTTP provides detailed evidence, but conclusions should remain within the limits of that evidence.

## The Security Perspective

HTTP gives web applications a structured language for exchanging information. Methods express intended operations, URLs identify resources, headers carry metadata and state, request bodies transport data, and responses communicate results.

Those same mechanisms create many of the places where security decisions need to happen. Parameters need validation. Objects need authorization checks. Session identifiers need protection. Tokens need verification. Uploaded content needs safe handling. Browser behavior needs appropriate security policies. Proxies need defined trust boundaries. Logs need enough context to support investigations.

HTTPS adds cryptographic protection to the communication channel through TLS. Certificates help clients authenticate servers, encryption protects application content from ordinary passive observation, and integrity protection helps detect unauthorized modification in transit.

Neither protocol should be reduced to a port number or a padlock icon.

Understanding HTTP means being able to look beneath a webpage and recognize the requests, responses, identities, state, trust decisions, and application behavior that produced it. Understanding HTTPS means knowing exactly which part of that process TLS protects and which security responsibilities still belong to the application, browser, server, infrastructure, and people operating them.

Once web traffic is viewed this way, HTTP becomes much more than a protocol used to load websites. It becomes one of the clearest ways to understand how modern applications communicate and where many of their security decisions succeed or fail.

## From Web Traffic to the Systems Behind It

Understanding HTTP and HTTPS shows us how applications communicate across the web, but security work rarely stops at the traffic itself. Behind those requests and responses are operating systems running web servers, applications, services, processes, configuration files, and logs.

Linux appears throughout that environment. It powers a large portion of server infrastructure, forms the foundation of Kali Linux, and provides the command-line environment used by many security tools. Working comfortably with Linux makes it much easier to navigate systems, inspect files and permissions, examine processes, search logs, manage services, and understand what is happening on a machine.

The goal is not to memorize every Linux command available. It is to understand the commands that repeatedly become useful when working with systems and security tools, what information they provide, and how to combine them when investigating a problem.

Next:

**Linux Commands You Actually Need for Cybersecurity Labs**
