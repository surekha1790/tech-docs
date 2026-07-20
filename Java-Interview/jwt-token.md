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



