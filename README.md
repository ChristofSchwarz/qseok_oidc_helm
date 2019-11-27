## Helm Chart for the Single-Signon Passthru ODIC
helm name: oidc_passthrough

Project by **Jacob Vinzent** and **Christof Schwarz**

This helm starts a pseudo oidc provider for Qlik Sense Enterprise on Kubernetes (QSEoK) which enables a single-sign on possibility using a JWT token.

### How to install

To run it, clone this project into a folder, then values.yaml and also edit the qliksense.yaml to add this as an identity-provider, then deploy this helm and upgrade qliksense. We will discuss this steps:

Get this repo and edit the values
```
git clone https://github.com/ChristofSchwarz/qseok_oidc_helm
nano ./qseok_oidc_helm/values.yaml
```

```
helm upgrade --install oidc ./qseok_oidc_helm -f values.yaml
# check the settings that the pod has received from your values (env variables)
kubectl exec $(kubectl get pods -o=name --selector app=oidcpassthrough) env|grep -v QLIK
```
 ![alttext](https://github.com/ChristofSchwarz/pics/raw/master/oidc-screenshot1.png "screenshot")
 
Test that the Kubernetes components (the ingress, the service, the deployment, the pod) work, this address should resolve with a reply (assuing you have another hostname than 192.168.56.234)
https://192.168.56.234/oidc/.well-known/openid-configuration

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



