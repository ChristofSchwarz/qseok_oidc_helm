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
### Idea

### Solution




