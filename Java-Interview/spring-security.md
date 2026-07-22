### * How user login flow and spring security works ?
- When user tries to login, first verify whether user is active or not.
- If active then authenticate the user by
  ```
  authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken(
      user.getId().toString(),  // principal is UUID
      request.getPassword()
    )
  );
  ```
- ```authenticate()``` method hands off the validation to ```DaoAuthenticationProvider```
  ```
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
  ```
- It fetches the password using userDetailsService and check the given password against encoded password from DB

### * What is SecurityContextHolder ?
- It holds the current authenticated user.
- ```AuthenticationFilter``` builds ```Authentication``` object and set it to ```SecurityContextHolder.getContext().setAuthentication(authToken);```
- Subsequent requests like ```@PreAuthorize``` gets logged in user details from holder.
- Stored in threadlocal per request and should be cleared ```SecurityContextHolder.clearContext();```
