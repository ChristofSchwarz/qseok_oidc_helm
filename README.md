## Helm Chart for the Single-Signon Passthru ODIC
helm name: oidc_passthrough

Project by **Jacob Vinzent** and **Christof Schwarz**

This helm starts a pseudo oidc provider for Qlik Sense Enterprise on Kubernetes (QSEoK) which enables a single-sign on possibility using a JWT token.

To run it, clone this project into a folder, edit values.yaml, and deploy it
```
git clone https://github.com/ChristofSchwarz/qseok_oidc_helm
nano ./qseok_oidc_helm/values.yaml
helm install -n oidc ./qseok_oidc_helm
```
 ![alttext](https://github.com/ChristofSchwarz/pics/raw/master/oidc-screenshot1.png "screenshot")
 
### Idea

In the first year of Qlik Sense Enterprise on Kubernetes, a few things are yet missing which Qlik Sense had
on its Windows version: The concept of Single-Signon (SSO) with a so-called virtual proxy. In some client discussion,
especially in the OEM business, the lack of SSO became a killer criteria for the Kubernetes version, so we
started this innovation project.

### Solution

The solution is based on this git https://github.com/panva/node-oidc-provider and on Qlik's own interpretation 
of that git, which is bundled into QSEoK when you launch Qlik Sense with parameters edge-auth.oidc.enabled=true 
... 

In contrast to a normal behaviour of an OIDC provider, where the user would be redirected to the OIDC to enter his/her 
password and to give his/her consent for QSEoK to access his/her profile, this simplified version has no user
management or password prompt. It expects a special cookie, which a newly written endpoint had left for moment 
when Qlik Sense redirects the unauthenticated, incoming user to the OIDC provider of choice. Then this becomes
a seemless single-sign on, as our manipulated version of that OIDC provider will read the cookie instead of 
prompting anything and then proceeds in the OIDC process (send the user back to Qlik, then edge-auth pod does
some more communication with the IDP to validate the access_token etc - standard OAuth 2.0 behaviour). 

So in short: We extended the 3-legged implicit OAuth 2.0 flow (which is at the core of OIDC) by a 4th leg up front, that 
is where you will start the single-signon process.



