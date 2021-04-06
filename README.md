# kubernetes-localdev

## Requirements

1. **Windows**: Windows version [1903 with build 18362 and above](https://docs.microsoft.com/en-us/windows/wsl/install-win10#step-2---check-requirements-for-running-wsl-2), hit WINKEY+R and run `winver`
1. **Windows**: [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/install/)
1. **Windows**: [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10) - Windows Subsystem Linux running on [Ubuntu 20.04](https://www.microsoft.com/en-il/p/ubuntu-2004-lts/9n6svws3rx71?rtc=1#activetab=pivot:overviewtab)
1. **Windows**: [VSCode](https://code.visualstudio.com/download) and [the Remote - WSL extension](https://code.visualstudio.com/blogs/2019/09/03/wsl2)
1. **Windows**: [choco](https://docs.chocolatey.org/en-us/choco/setup) - Windows package manager
   ```bash
   # PowerShell as Administrator (elevated)
   Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
   ```
1. **Windows**: [mkcert](https://github.com/FiloSottile/mkcert) - mkcert is a simple tool for making locally-trusted development certificates. It requires no configuration.
   ```
   choco install mkcert
   ```
1. **Windows**: [LENS 4.2.0](https://k8slens.dev/) - The Kubernetes IDE - [Download and install on Windows](https://github.com/lensapp/lens/releases/download/v4.2.0/Lens-Setup-4.2.0.exe)   
1. **WSL2**: [minikube](https://minikube.sigs.k8s.io/docs/start/) - a tool that lets you run Kubernetes locally
   ```
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && \
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
   ```
1. **WSL2**: Helm v3.x - the package manager for Kubernetes
    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 && \
    chmod 700 get_helm.sh && \
    ./get_helm.sh
    ```

## Create a Kubernetes Cluster (WSL2)

```bash
minikube start --driver=docker --kubernetes-version=v1.20.2
# ...
# 🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Configure Cluster Connection

1. **WSL2**: Check connectivity - HTTPS should work since we're using `ca.crt`
    ```bash
    MINIKUBE_EXPOSED_PORT="$(kubectl config view -o jsonpath='{.clusters[?(@.name == "minikube")].cluster.server}' | cut -d":" -f3)" && \
    export MINIKUBE_EXPOSED_PORT=${MINIKUBE_EXPOSED_PORT} && \
    curl -L --cacert ~/.minikube/ca.crt  "https://127.0.0.1:${MINIKUBE_EXPOSED_PORT}/version"
    ```

    A valid response
    ```json
    {
        "major": "1",
        "minor": "20",
        "gitVersion": "v1.20.2",
        "gitCommit": "faecb196815e248d3ecfb03c680a4507229c2a56",
        "gitTreeState": "clean",
        "buildDate": "2021-01-13T13:20:00Z",
        "goVersion": "go1.15.5",
        "compiler": "gc",
        "platform": "linux/amd64"
    }
    ```
1. **WSL2**: Copy KUBECONFIG to Windows host, **change the HOST_USERNAME** to your Windows host user name, mine is `unfor19`
    ```bash
    # Set variable
    HOST_USERNAME="unfor19"
    ```

    ```bash
    # Copy KUBECONFIG and certs from WSL2 to host
    mkdir -p "/mnt/c/Users/${HOST_USERNAME}/.kube/certs" && \
    cp -r "${HOME}/.kube/" "/mnt/c/Users/${HOST_USERNAME}" && \
    rm -rf "/mnt/c/Users/${HOST_USERNAME}/.kube/cache" && \
    # Change paths from `/home/$USER/*.minikube` with to `certs`
    sed 's~/home/'"${USER}"'.*.minikube~certs~g' "${HOME}/.kube/config" > "/mnt/c/Users/${HOST_USERNAME}/.kube/config"
    ```
1. **WSL2**: Copy minikube's certificates to Windows host
    ```bash
    # ALL Minikube
    # Client certificate
    cp  ~/.minikube/profiles/minikube/client.crt /mnt/c/Users/unfor19/.kube/certs/client.crt && \
    cp  ~/.minikube/profiles/minikube/client.key /mnt/c/Users/unfor19/.kube/certs/client.key && \
        # Certificate Authority (CA) certificate
        cp  ~/.minikube/ca.crt /mnt/c/Users/unfor19/.kube/certs/ca.crt && \
        # Prepare URL for Windows
        echo "Install the certificates and then open a new browser Incognito/Private window - https://127.0.0.1:${MINIKUBE_EXPOSED_PORT}/version" 
    ```
1. **Windows**: Install the certificates `ca.crt` and `client.crt` for the **Current User** in the certificate store **Trusted Root Certification Authorities** (double click both files)

    ![minikube-install-certs](./assets/minikube-install-certs.png)

    ![minikube-install-certs-store](./assets/minikube-install-certs-store.png)
1. **Windows**: Check access to the cluster's endpoint by opening the browser in `https://127.0.0.1:${MINIKUBE_EXPOSED_PORT}/version`
1. (Optional) **Windows**: Use the KUBECONFIG file in LENS when adding a cluster (Windows)
    ![lens-add-cluster](./assets/lens-add-cluster.png)

    Select **All namespaces**
    ![lens-view-pods](./assets/lens-view-pods.png)
 
### Install NGINX Ingress Controller with Helm (WSL2)

- **WSL2**: 
    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && \
    helm repo update && \
    helm upgrade --install nginx ingress-nginx/ingress-nginx --set controller.kind=DaemonSet
    ```

## Support DNS resolution in Windows host

To access the application from the Host machine (Windows) we need to map its domain name to `127.0.0.1`, which in turn will listen to ports 80 and 443 with NGINX Ingress Controller.

- **Windows**: Edit `C:\Windows\System32\drivers\etc\hosts` with Notepad or [Notepad++](https://notepad-plus-plus.org/downloads/v7.9.5/) as Administrator and add
    ```bash
    127.0.0.1 baby.kubemaster.me green.kubemaster.me dark.kubemaster.me auth.kubemaster.me oidc.kubemaster.me darker.kubemaster.me
    ```

## HTTP Application

Deploy the `baby` app - a simple application served with NGINX Ingress via HTTP

1. **WSL2**: Deploy the example application
    ```bash
    kubectl apply -f 1-baby.yaml
    ```
1. **WSL2**: Open a new terminal window and serve NGINX Ingress Controller on Windows localhost, ports 80 and 443. This will provide the `nginx-ingress-nginx-controller` service an External IP of the Windows host `127.0.0.1`. **Keep it running in the background**
    ```bash
    minikube tunnel
    # ❗  The service nginx-ingress-nginx-controller requires privileged ports to be exposed: [80 443]
    # 🔑  sudo permission will be asked for it.
    # 🏃  Starting tunnel for service nginx-ingress-nginx-controller    
    ```
1. **Windows**: Check that minikube exposes NGINX Ingress Controller service on **127.0.0.1**
    ![minikube-tunnel](./assets/minikube-tunnel.png)
1. **Windows**: Open your browser in a new Incognito/Private window and navigate to http://baby.kubemaster.me/ (port 80) you should see a cute baby cat

## HTTPS

Create a local Certificate Authority certificate and key with [mkcert](https://github.com/FiloSottile/mkcert) and install it to `Trusted Root Certificate Authorities`. We'll use this certificate authority to create TLS certificates that will be used for local development.

### Create Certificate Authority Certificate And Key

1. **Windows**: Open Windows PowerShell as Administrator (elevated)
    ```powershell
    mkcert -install # Click Yes when prompted
    # The local CA is now installed in the system trust store! ⚡️
    mkcert -CAROOT
    # C:\Users\$HOST_USER\AppData\Local\mkcert
    ```
1. **Windows**: Verify Installed Certificates
    1. Hit WINKEY+R > Run `certmgr.msc`
    1. Certificate - Current User > Trusted Root Certificate Authorities > Certificates > Issuted to and by is `mkcert $MACHINE_NAME ...`
    1. The end result should be as below

    ![mkcert-certificate-installed](./assets/mkcert-certificate-installed.png)

    **TIP**: Can't see it? Close and re-open `certmgr.msc`, it doesn't auto-refresh upon adding certificates


### Load Certificate To A Kubernetes Secret (WSL2)

1. Mount the certificates that were created with `mkcert` from the Windows host to WSL
    ```bash
    # Set variable
    HOST_USERNAME="unfor19" # <-- CHANGE THIS!
    ```

    ```bash
    CAROOT_DIR="/mnt/media/caroot" && \
    sudo ln -s "/mnt/c/Users/${HOST_USERNAME}/AppData/Local/mkcert/" "$CAROOT_DIR" || \
    ls -l $CAROOT_DIR # Verify
    ```
1. Create the [cert-manager](https://cert-manager.io/docs/installation/kubernetes/) namespace and create a [Kubernetes Secret type TLS](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets)
    ```
    kubectl create namespace cert-manager && \
    kubectl -n cert-manager create secret tls kubemaster-me-ca-tls-secret --key="${CAROOT_DIR}/rootCA-key.pem" --cert="${CAROOT_DIR}/rootCA.pem"
    ```

### Install Cert-Manager And Issue A Certificate

1. **WSL2**: Install cert manager
    ```bash
    helm repo add jetstack https://charts.jetstack.io && \
    helm repo update                                  && \
    kubectl apply -f cert-manager/cert-manager-crds-1.2.0.yaml     && \
    helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.2.0
    ```
1. **WSL2**: Create a ClusterIssuer, Certificate and deploy the `green` app. See the Troubleshooting if you experience any issues.
    ```bash
    # This issuer uses the TLS secret `kubemaster-me-ca-tls-secret` to create certificates for the ingresses
    kubectl apply -f cert-manager/clusterissuer.yaml && \
    # Create a TLS Certificate for `kubemaster.me` and `*.kubemaster.me`
    kubectl apply -f 2-certificate.yaml && \
     # Deploy sample app
    kubectl apply -f 2-green.yaml
    ```
1. **Windows**: Check connectivity to the deployed `green` app, open browser and navigate to https://green.kubemaster.me (port 443) you should see a cat in a green scenery

## Authentication - OAuth2

We'll use [oauth2-proxy]() to proxy requests to Google's authentication service.

### Create Google's Credentials

1. **Windows**: [Google Developer Console](https://console.cloud.google.com/apis/dashboard?pli=1) > [Create a New Project](https://console.cloud.google.com/projectcreate?previousPage=%2Fapis%2Fdashboard%3Fproject%3Dkubemaster-me&folder=&organizationId=0)
    - Project Name: `kubemaster`
    - Organization: Leave empty
1. **Windows**:[OAuth consent screen](https://console.cloud.google.com/apis/credentials/consent) > Select **External** > Click **CREATE**
    - App name: `kubemaster`
    - User support email: `your email address`
    - Authorised domains > Add domain > `kubemaster.me`
    - Developer contact information: `your email address`
    Click **SAVE AND CONTINUE**
1. **Windows**: **Scopes** > Click **SAVE AND CONTINTUE** - there's no need for a scope, we don't plan on using Google APIs (authorization), we just need the authentication mechanism (OAuth2/OIDC)
1. (Optional) **Windows**: **Test users** > Click **SAVE AND CONTINTUE** - it's irrelevant since either way we're allowing any Google user to login to the app since it's a local app
1. **Windows**: **Summary** > Click **BACK TO DASHBOARD** 
1. **NOTE**: There's no need to **PUBLISH APP**, keep it in sandbox mode
1. **Windows**: [Credentials](https://console.cloud.google.com/apis/credentials) > Click **CREATE CREDENTIALS** > Select **OAuth Client ID** > Select Application type **Web application**
    - Name: `kubemaster`
    - Authorised JavaScript origins **ADD URI** > `https://auth.kubemaster.me`
    - Authorised JavaScript origins **ADD URI** > `https://oidc.kubemaster.me` (will use it later on)
    - Authorised redirect URIs **ADD URI** > `https://auth.kubemaster.me/oauth2/callback`
    - Authorised redirect URIs **ADD URI** > `https://oidc.kubemaster.me/oauth2/callback` (will use it later on)
    Click **CREATE**
    - Save **Your Client ID** and **Your Client Secret** in a safe place, we'll use them in the following section

### Creating Kubernetes secrets For credentials

1. **WSL2**:

    ```bash
    # Values from Google's Developer Console - the space at the beginning of the command is on purpose to keep it out from Bash's history
    OAUTH2_PROXY_CLIENT_ID="google_oauth2_project_client_id"
    OAUTH2_PROXY_CLIENT_SECRET="google_oauth2_project_client_secret"
    ```

    ```bash
    # Requires Python 3.6+ - sudo apt-get update && sudo apt-get install -y python3
    kubectl -n default create secret generic oauth2-proxy-cookie-secret --from-literal=oauth2_proxy_cookie_secret="$(python3 -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(16)).decode())')" && \
    # Create the Kubernetes Secret
    kubectl -n default create secret generic google-credentials \
        --from-literal=google_client_id="${OAUTH2_PROXY_CLIENT_ID}" \
        --from-literal=google_client_secret="${OAUTH2_PROXY_CLIENT_SECRET}"
    ```

### Deploy OAuth2-Proxy And Protect An Application

1. **WSL**: Deploy OAuth-Proxy and the sample `dark` application
    ```bash
    # Deploy oauth2-proxy
    kubectl apply -f 3-oauth2-proxy.yaml && \
    # Deploy sample app `dark`, served with HTTPS and protected with Google authentication
    kubectl apply -f 3-dark.yaml
    ```
1. **Windows**: Open a browser in a new Incognito/Private window and navigate to https://dark.kubemaster.me and login with your Google user. You should see a cat in a dark scenery.

## Authentication - OIDC

In the previous step OAuth2 was used for authentication, though it's main purpose is for **authorization**. For **authentication** it is best to use [Open ID Connect (OIDC)](https://developers.google.com/identity/protocols/oauth2/openid-connect) whenever it's possible. The main benefit is that OIDC also provides the endpoint `/userinfo`, so the application can easily read a [JSON Web Token (JWT)](https://jwt.io/) and get the user details such as full name and email address.

As demonstrated in the below image, OIDC **does not** replace OAuth2. OIDC is an external on top of OAuth2 which provides a better way to handle authentication.

![oauth-oidc-layers](./assets/oauth-oidc-layers.png)

### Deploy OAuth2-Proxy And Use OIDC

The deployment steps are going to be exactly the same as before, though I do recommend viewing the files [4-oauth2-proxy-oidc.yaml](./4-oauth2-proxy-oidc.yaml) and [4-oauth2-proxy-oidc.yaml](./4-oauth2-proxy-oidc.yaml), while comparing them to [3-oauth2-proxy.yaml](./3-oauth2-proxy.yaml) and [3-dark.yaml](./3-dark.yaml).

The main difference is in the configuration of oauth2-proxy, where the provider is not using the default OAuth2 protocol for authentication, instead it's using the OIDC protocol.

1. **WSL**: Deploy OAuth-Proxy-OIDC and the sample `darker` application
    ```bash
    # Deploy oauth2-proxy
    kubectl apply -f 4-oauth2-proxy-oidc.yaml && \
    # Deploy sample app `darker`, served with HTTPS and protected with Google authentication (OIDC)
    kubectl apply -f 4-darker-oidc.yaml
    ```
1. **Windows**: Open a browser in a new Incognito/Private window and navigate to https://darker.kubemaster.me and login with your Google user. You should see the same dark cat in the green scenery.
    - **NOTE**: If you have an existing browser windwow, even if it's incognito, then you might be already autehnticated. You can verify it by checking if the cookies `_oauth2_proxy` exists. On the backend, the authentication was done with OIDC, to get the full flow, close all incognito windows and then open a new browser windows in incognito https://darker.kubemaster.me , this time you'll see for a split second that the authentication is done with `oidc.kubemaster.me`


## Authentication Summary

- I find it best to have a dedicated subdomain for Authentication services, as it allows using cookies with `*.kubemaster.me` and acts as an isolated service from the entire application
- It's possible to access private resources by logging in both in https://auth.kubemaster.me and https://oidc.kubemaster.me since both use the same COOKIE_SECRET (I think?)
- In case it's not clear - The *Authorised JavaScript origins* and *Authorised redirect URIs* that were declared in Google's Developer Console are used by oauth2-proxy, there's not a single time where Google tries to query your domains, this is why it's possible to make it work locally.
- Here's great 1 hour session about OAuth2 and OIDC - [OAuth 2.0 and OpenID Connect (in plain English)](https://www.youtube.com/watch?v=996OiexHze0&ab_channel=OktaDev). I watched every bit of it and it helped me to understand the whole flow.

## Cleanup

**IMPORTANT**: Quit LENS before proceeding

- **WSL2**: Delete minikube's Kubernetes Cluster and 
    ```bash
    HOST_USERNAME="unfor19"
    minikube delete && \
    rm -r "/mnt/c/Users/${HOST_USERNAME}/.kube/certs /mnt/c/Users/${HOST_USERNAME}/.kube/client.* /mnt/c/Users/${HOST_USERNAME}/.kube/ca.crt"
    ```
- **Windows**: Uninstall **mkcert**'s SSL certifictes, run PowerShell in elevated mode
    ```bash
    mkcert -uninstall
    # The local CA is now uninstalled from the system trust store(s)!    
    ```
- **Windows**: Remove **minikube**'s SSL certifictes, hit WINKEY+R and run `certmgr.msc`
    1. Certificates - Current User > Trusted Root Certification Authorities > Certificates
    1. Delete all minikube's certificates - **minikubeCA** and **minikube-user**

## Troubleshooting

1. **Ingress**: ERR_CONNECTION_REFUSED
    - Make sure you expose the cluster to the host with `minikube tunnel` before trying to access the application with the browser
    ![troubleshooting-err-connection-refused](./assets/troubleshooting-err-connection-refused.png)
1. **Ingress**: Path based ingresses issues, For example `app.kubemaster.me/baby` would not work properly because the app serves static files in the root dir. The request to the HTML page `index.hml` is successful but subsequent requests to `app.kubemaster.me/baby/images/baby.png` will fail since NGINX's upstream can't serve static content. Path based ingresses are mostly used for serving APIs, for example `app.kubemaster.me/api/v1/get/something`. Use bare (`/`) host based ingresses for serving static pages, just like I did in this project.
1. **Ingress**: version deprecation warning - ignore this warning, this is the latest version supported by the NGINX Ingress Controller
    ```bash
    Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
    ```
1. **HTTPS**: Certificate is invalid in browser - Open your browser a new Incognito/Private window
    - ![troubleshooting-err-connection-refused](./assets/troubleshooting-err-connection-refused.png)
    - ![troubleshooting-connection-is-not-private](./assets/troubleshooting-connection-is-not-private.png)
1. **cert-manager**: Errors applying cert-manager resources
    - Delete the secret `kubemaster-me-ca-tls-secret`, re-create it and then re-apply `cert-manager/clusterissuer.yaml`
        ```
        Error from server (NotFound): error when deleting "cert-manager/clusterissuer.yaml": clusterissuers.cert-manager.io "tls-ca-issuer" not found
        ```
    - Wait for cert-manager to be healthy, check all logs of the pods `cert-manager`, `cert-manager-cainjector` and `cert-manager-webhook`
        ```
        Error from server (InternalError): error when creating "cert-manager/clusterissuer.yaml": Internal error occurred: failed calling webhook "webhook.cert-manager.io": Post "https://cert-manager-webhook.cert-manager.svc:443/mutate?timeout=10s": dial tcp 10.102.252.218:443: connect: connection refused    
        ```
1. **Authentication**: After login you get redirected to `/#` 404 after a successful login - Make sure the ingress annotation is `oauth2/start?rd=https://$host$escaped_request_uri`, for example
    ```yaml
    nginx.ingress.kubernetes.io/auth-signin: https://auth.kubemaster.me/oauth2/start?rd=https://$host$escaped_request_uri
    ```
    **NOTE**: Even though you got to a 404 page, it's still possible to access private resources (dark and darker) check your Application cookies
1. **LENS**: Can't connect to cluster due to missing keys - Make sure you copied `client.crt`, `client.key` and `ca.crt` from WSL2 to the Windows host `C:\Users\$HOST_USERNAME\.kube\certs`
    ```
    error: unable to read client-key C:\Users\unfor19\.kube\certs\client.key for minikube due to open C:\Users\unfor19\.kube\certs\client.key: The system cannot find the file specified.
    ```

## Future Work

Sections that will be added to this project.

1. Local development (CI) of [docker-cats](https://github.com/unfor19/docker-cats)
1. Local deployment (CD) of [docker-cats](https://github.com/unfor19/docker-cats)
1. How to create and manage Kubernetes Secrets with [HashiCorp's Vault](https://www.vaultproject.io/)
1. How to add how to add more nodes
    ```bash
    minikube node add --worker=true
    ```

## Authors

Created and maintained by [Meir Gabay](https://github.com/unfor19)

## License

This project is licensed under the MIT License - see the [LICENSE](https://github.com/unfor19/kubernetes-localdev/blob/master/LICENSE) file for details