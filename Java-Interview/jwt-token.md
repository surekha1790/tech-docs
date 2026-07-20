### * What is JWT Token ?
- JWT (JSON web token) is used for authentication and authorization.
- When user logs in, server generates an access token and send it to client(UI).
- Client sends this token for every request from user and server validates it.
- If matches then only request can go through otherwise user gets 401 unauthorised access.

### * What are the three parts of a JWT?
- **Header:** says how it's signed (e.g., algorithm HS256).
  ```
  {
    "alg": "HS256",
    "typ": "JWT"
  }```
- **Payload:**  the actual data ("claims"): who you are, your roles, when it expires. In our system: sub (user id), email, name, roles, userType.
  ```
  {
    "sub":"john",
    "role":"ADMIN",
    "exp":123456789
  }```
- **Signature:**  a cryptographic seal created with a secret key. If anyone changes even one character of the payload, the signature no longer matches and the token is rejected.

### * How does JWT authentication work?
- User submits login request with username and password.
- Auth service validates it.
- Once it is validated, auth service generates JWT token and send it to client.
- client save it and send it in ```Authorization: bearer <toke>``` header with every request.
- If valid, Spring Security creates an authenticated user context.

### * Why is JWT called stateless?
- JWT is stateless because it does not required to be stored anywhere unlike session
- Server just need to validate the token

### * Where is JWT stored?
It is stored client side 
- Http only secure cookie
- Local storage
- Mobile secure storage

### * Is JWT encrypted?
- No, it is not encrypted
- It is Base64URL encoded which can be decoded.
- Secure key/Signature can protect it from tampering.

### * Can someone read the payload?
- Yes, since it is just encoded message, it can be decoded.
- We should not store information like password, credit card details and sensitive personal etc.

### * How does the server know the token wasn't modified?
- Server calculates signature using secret/public key and validate.
- If does not match then it is tampered.

### * Why do JWTs have an expiration time?
- Token will not be valid and can not go through the request once expired.
- If token is stolen then it should not be validated and redirected.
- So, setting expiration time helps in avoiding such scenarios and restrict access

### * Which Spring Security class validates JWT?
- JwtAuthenticationFilter -> every request passes through this filter.

### * What is SecurityContextHolder?
- Set authenticated user for current request.
  ```
  Authentication auth =
  SecurityContextHolder
  .getContext()
  .getAuthentication();
  ```

### * Why do we configure SessionCreationPolicy.STATELESS?
- Since JWT is stateless, we do not need session.
- If it does not set then spring creates HttpSession

### * What are claims?
- Claims are pieces of information which are included in JWT payload

### * What are registered claims?
  ```
    sub
    iss
    aud
    exp
    iat
    nbf
    jti
  ```
### * What is an Access Token?
- Access token is to authorise the user and it is short lived.
- Once issues can not be revoked and we need to wait until it is expired.
- Security issues if it is stolen and need to wait untill expired.

### What is Refresh Token?
- Refresh token is long lived token.
- It is stored in DB.
- Once access token in expired, refresh token will be issues in the backend to validate in future requests.
- Since it is short lived, it is easy to revoke and reissue new token.
- Purpose of this token is, to keep user logged in and make it secured with new token.

### What algorithms are used to sign JWT?
- Common algorithms:
  ``` HS256 (HMAC using a shared secret)
      RS256 (RSA public/private key)
      ES256 (Elliptic Curve)
  ```
### * A request reaches your API Gateway. What happens?
- Read the Authorization header.
- Extract the Bearer token.
- Validate the signature.
- Check expiration and other claims.
- Create an Authentication object.
- Store it in SecurityContextHolder.
- Forward the request to the appropriate microservice.
- Return 401 Unauthorized if validation fails.

### * What HTTP status codes are typically returned?
- 401 : Unauthorized - Token is not valid/missing/expired.
- 403 : Forbidden - Token is valid but required permissions.
- 200 : Success.

  





