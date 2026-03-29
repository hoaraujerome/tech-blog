+++
date = '2021-12-12T15:29:02-04:00'
title = '5 Responsibilities Regarding the JWT as an API Provider'
+++

As a reminder, a JWT (JSON Web Token) is a way for securely transmitting information between parties as a JSON object.

As an API provider, here are the actions to take on the received JWT:

1.  Validate the signature of the JWT (**mandatory**)
    
2.  Check if the scope necessary to use your API is present (**mandatory**). Your API may require more than one scope.
    
3.  Check if the JWT is not expired (**mandatory**)
    
4.  Validate the audience to make sure that you are the target of the JWT (optional but **recommended**)
    
5.  Validate the issuer to ensure that a trusted source issued the JWT (optional but **recommended**)
    

Modern programming frameworks - Spring Security (Java) - do some or all of these validations for you. Otherwise, you have to implement them manually.

Learn more JSON Web Tokens [here](https://auth0.com/docs/security/tokens/json-web-tokens) .
