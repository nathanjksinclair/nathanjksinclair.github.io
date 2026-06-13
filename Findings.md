# Findings when trying to make a database version of checklist

According to brave, the usage of https://github.com/login/oauth/access_token can only be used on server.

The following error occurs.

```
Access to fetch at 'https://github.com/login/oauth/access_token' from origin 'https://nathanjksinclair.github.io' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

This is to prevent the leakage of secrets.

I:

1. Modified the repo so that the secret is not found.
2. Made the current secret invalid.
3. Searched for ways to handle server side for webpage.


[this article](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)

is helpful in getting github oauth set up.

