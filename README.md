## Helm Chart for the Single-Signon Passthru ODIC
helm name: oidc_passthrough

Project by **Jacob Vinzent** and **Christof Schwarz**

This helm starts a pseudo oidc provider for Qlik Sense Enterprise on Kubernetes (QSEoK) which enables a single-sign on possibility using a JWT token.

## How to install

### Part 1/3 - Setup Passthru OIDC
To run it, clone this project into a folder, then values.yaml and also edit the qliksense.yaml to add this as an identity-provider, then deploy this helm and upgrade qliksense. We will discuss this steps (note I am using elastic.example as the placeholder for my qlik sense url, but it works with any other unlike Qlik's simple-oidc solution):

Get this repo and edit the values
```
git clone https://github.com/ChristofSchwarz/qseok_oidc_helm
nano ./qseok_oidc_helm/values.yaml
```
Have a look at <a href="values.yaml">values.yaml</a> to see possible settings. To understand the meaning of each setting, see comments in <a href="templates/oidc_depl.yaml">oidc_depl.yaml</a>

Then roll out the helm installation. It will install a Deployment, a Service, and an Ingress
```
helm upgrade --install oidc ./qseok_oidc_helm -f ./qseok_oidc_helm/values.yaml
```
 ![alttext](https://github.com/ChristofSchwarz/pics/raw/master/oidc-screenshot1.png "screenshot")

You can check, that the settings made it into the newly created pod as env variables
```
kubectl exec $(kubectl get pods -o=name --selector app=oidcpassthrough) env|grep -v QLIK
```
Since we deployed this passthrough oidc as an ingress route within qlik sense, connect to your qlik sense server address and the path prefix (here /oidc which is the default), those endpoints should work
 - https://elastic.example/oidc
 - https://elastic.example/oidc/.well-known/openid-configuration
 - https://elastic.example/oidc/env
 - https://elastic.example/oidc/signin

### Part 2/3 - Configure QSEoK

Now that the passthru-oidc is running and able to perform single-signons, we have to configure it as a identity-provider within our qliksense deployment.

Find your qliksense.yaml which you used to install qlik sense with helm. If you are not sure, you can create qliksense.yaml with this command ('qlik' is my name of the helm deployment, could be different, check with "helm list"):
```
helm get values qlik >qliksense.yaml
```
Edit this file. Add a new entry under section "identity-providers.secrets.idpConfigs". Copy/paste the settings then modify accordingly:
```
identity-providers:
  secrets:
    idpConfigs:
      - hostname: "analytics.company.com" 
        realm: "sso"              
        discoveryUrl: "https://analytics.company.com/oidc/.well-known/openid-configuration"
        postLogoutRedirectUri: "https://mainapp.company.com"
        clientId: "foo"
        clientSecret: "thanksjviandcsw"
        scope: "openid profile"
        claimsMapping:
           name: ["name"]
           sub: ["sub", "id"]
           groups: ["groups"]
```
 - hostname : the main address of your web server (without https and without any sub-page, no "/")
 - realm: will be the prefix for all user ids within qlik
 - discoveryUrl: the full url to the ingress route of the .well-known/openid-configuration (we tested it above) 
 - clientId and clientSecret: match the values of the values.yaml from Part 1
 - postLogoutRedirectUri: Where to go back after logout, basically your main web app, same you specified as "post_logout_redirects" and "fwd_no_cookie" under values.yaml from Part 1

Apply the changes
```
helm upgrade --install qlik qlik-stable/qliksense -f qliksense.yaml
```
If your qlik sense site **doesn't use public certificates**, the edge-auth pod will not accept the OIDC (even it is an ingress under the same server url). You have to patch the edge-auth deployment:
```
kubectl patch deployment qlik-edge-auth -p '{"spec":{"template":{"spec":{"containers":[{"name":"edge-auth", "env":[{"name":"NODE_TLS_REJECT_UNAUTHORIZED","value":"0"}]}]}}}}'
```

### Part 3/3

![alttext](https://github.com/ChristofSchwarz/pics/raw/master/passthruoidc.gif "screenshot")

Example:
https://elastic.example/oidc/signin?forward=https://elastic.example&jwt=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Imp1YiIsIm5hbWUiOiJDaHJpc3RvZiBKYWNvYiIsImdyb3VwcyI6WyJFdmVyeW9uZSIsIk9FTSJdLCJpYXQiOjE2NzM4MDU3ODJ9.QFBdlGIPTLS4wS63lfCkRyqx80tapgzRIqiyYTmZpZW3EZERywgm7164SF1iDDZxOsgCAenRAW165jSgZAU7aOU1pFprl6FKd0umxKUs55TO6m2KeQHHHhDlXcKiWBtjW-KWVTFYHDVh6Md0DDHjLbyVaZQ5PIYnoSS33OTC4KMSSmDUrivvGK0uDf9naOWbWVdQgHXLRpMeV35iZwzRMVSg3XGSO0_h2CrkWWYWGHvgiR-2ZfdHE_j8emlsuFGiFrQzFXpHFXULnCmYHgOS2LuekIhr-TYjIdSBkDOcoC6WM-PZoBCQ9ZZa1M2oadhkMAZlqRNYL29Cyl29yGjk1Q


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



