# conjur-oss-helm-chart

kubernetes and conjur oss setup for JWT Based Authnetication:

Deploy into a local (on mac) kubernetes with working k8s authenticator and test application.

1:prerequisite:

   I: Docker desktop for mac, must be running

   II:homebrew

   III:git

   IV:kubectl

   V:helm    //brew install helm

   VI:conjur cli (v7.1.x / python version)


2: KinD: a way to run kubernetes in Docker
   
    I:brew install kind
  
    ii:create kind cluster

    cat <<EOF | kind create cluster --name k8sauth --config=-
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
        kubeletExtraArgs:
            node-labels: "ingress-ready=true"
      extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
      - containerPort: 8080
        hostPort: 8080
        protocol: TCP
      - containerPort: 30987
        hostPort: 30987
        protocol: TCP
    EOF

  iii: configure kubectl to use cluster: kubectl config use-contexts kind-k8sauth 
                  to check the contexts: kubectl config get-contexts
                  
  iv: for cluster info to check kubernetes end point : kubectl cluster-info                 

  
3: Deploy Conjur with Helm:
      i: clone repo: git clone https://github.com/cyberark/conjur-oss-helm-chart
   
      ii: # Added conjur.account because k8s authenticator reads the account
       # name from a server env var which has a default value of default.
       # Added ssl.altName so the k8s internal svc can be used with the
       # same cert.
       # Also added localhost as a SAN for accessing conjur from the host.
       
       cd conjur-oss-helm-chart
       CONJUR_NAMESPACE="conjur-namespace"
       CONJUR_ACCOUNT="conjur"
       kubectl create namespace "$CONJUR_NAMESPACE"
       DATA_KEY="$(docker run --rm cyberark/conjur data-key generate)"
       echo "${DATA_KEY}" # store this
       HELM_RELEASE="conjur"
       helm install \
          -n "$CONJUR_NAMESPACE" \
          --set dataKey="$DATA_KEY" \
          --set account.name="${CONJUR_ACCOUNT}" \
          --set "ssl.altNames[0]=conjur-conjur-oss.conjur-namespace.svc.cluster.local" \
          --set "ssl.altNames[1]=localhost" \
          "$HELM_RELEASE" \
          ./conjur-oss
   
4: patch the conjur service to have the consistent Nodeport:

      cat <<EOF > service.patch
      spec:
          type: NodePort
          ports:
              - port: 443
                targetPort: https
                protocol: TCP
                name: https
                nodePort: 30987
      EOF
     

    kubectl -n "${CONJUR_NAMESPACE}" patch service conjur-conjur-oss \
      --patch-file service.patch
   
    check conjur service is reachanbel from host: curl -ksI https://localhost:30987 | head -n 1

5: Configure conjur Account:

  CONJUR_ACCOUNT: conjur

  POD_NAME=$(kubectl get po -n "$CONJUR_NAMESPACE" \
              -l "app=conjur-oss, release="$HELM_RELEASE" \
              -o -jsonpath="{.items[0].metadata.name}")

  echo $POD_NAME

  kubectl exec --namespace $CONJUR_NAMESPACE \
             $POD_NAME \
             --container=conjur-oss \
             -- conjurctl account create $CONJUR_ACCOUNT | tail -1            

  
  Note: Once account has been created store admin creds for later use:      
        # Your API key will be different
          Created new account 'conjur'
          API key for admin: 115wcm32s10x828te4b487mjn91fkfnjq12bbtam107kjdp25244vy


6: conjur certificate(conjur init will download it)
   Login into the ConjurCLI(download cli required)

   #localhost is a valid SAN for conjur (see SANs in helm conjur deploy step above)

     conjur init -u https://localhost:30987 -a conjur --self-signed
     conjur login --id admin


7: Kubernetes authenticator set up

   i: discover the kubernetes resource(collect the information from the kubernetes cluster configuration that needed to configure JWT Authenticator)
      a: check the service account discovery service in kubernetes is publicly available:
          curl $(kubectl get --raw /.well-known/openid-configuration | jq -r '.jwks_uri') | jq
      b: kubectl get --raw /.well-known/openid-configuration | jq -r '.jwks_uri'   //for jwks_uri
      c: kubectl get --raw $(kubectl get --raw /.well-known/openid-configuration | jq -r '.jwks_uri') > jwks.json  // saves public key in jwks.jsob 
      
      d: service account token issuer 
          kubectl get --raw /.well-known/openid-configuration | jq -r '.issuer' 

    ii: JWT Authenticator in conjur(make sure proper indendation)
     jwt-authenticator-webservice.yaml

                - !policy
                  id: conjur/authn-jwt/dev-cluster
                  body:
                    - !webservice
                
                    # Uncomment one of following variables depending on the public availability
                    # of the Service Account Issuer Discovery service in Kubernetes 
                    # If the service is publicly available, uncomment 'jwks-uri'.
                    # If the service is not available, uncomment 'public-keys'
                    # - !variable jwks-uri
                    - !variable public-keys
                    - !variable issuer
                    - !variable token-app-property
                    - !variable identity-path
                    - !variable audience
                    
                    # Group of applications that can authenticate using this JWT Authenticator
                    - !group apps
                  
                    - !permit
                      role: !group apps
                      privilege: [ read, authenticate ]
                      resource: !webservice
                  
                    - !webservice status
                  
                    # Group of users who can check the status of the JWT Authenticator
                    - !group operators
                  
                    - !permit
                      role: !group operators
                      privilege: [ read ]
                      resource: !webservice status
       
       load the policy: conjur policy load -f jwt-authenticator-webservice.yaml -b root

      iii: populate the variables:
           //public key 
            a: conjur variable set -i conjur/authn-jwt/dev-cluster/public-keys -v "{\"type\":\"jwks\", \"value\":$(cat jwks.json)}"
          //issuer
            b: conjur variable set -i conjur/authn-jwt/dev-cluster/issuer -v "https://kubernetes.default.svc.cluster.local"
          // token-app-property
            c: conjur variable set -i conjur/authn-jwt/dev-cluster/token-app-property -v "sub"
          //identity path
             d: conjur variable set -i conjur/authn-jwt/dev-cluster/identity-path -v app-path
          //audience
             e: conjur variable set -i conjur/authn-jwt/dev-cluster/audience -v "https://kubernetes.default.svc.cluster.local"
          
8: Enable the JWT Authenticator
   SERVICE_ID=dev-cluster
   CONJUR_ACCOUNT=conjur          

   helm upgrade \
    -n conjur-namespace \
    --reuse-values \
    --set authenticators="authn\,authn-jwt/dev-cluster" \
    "$HELM_RELEASE" \
    cyberark/conjur-oss    


9: set up the workload
    # Make sure you have Kubernetes admin permissions on a Kubernetes cluster (version 1.21 or later)
    # Make sure you have access to the Kubernetes namespace for your workload, for example, test-app-namespace
    # Make sure that Helm is installed and that you have access to the Helm charts for preparing namespaces in Kubernetes:
     (https://github.com/cyberark/helm-charts)
    # Make sure you have access to a running Conjur Server
    # Make sure that a JWT Authenticator has been configured and allowlisted

    i: helm repo add cyberark https://cyberark.github.io/helm-charts
    ii: helm update repo

    # JWT Authenticator= authn-jwt/dev-cluster
    # ConurAccount=conjur
    # conjur Appliance url: https://conjur-conjur-oss.conjur-namespace.svc.cluster.local  //https://<service_name>.<namespace_name>.svc.cluster.local>
    # certificate file

   iii: Prepare the Kubernetes cluster and Golden ConfigMap.
        Install the clusterprep helm chart which creates the Golden ConfigMap,conjur-configmap, in $CONJUR_NAMESPACE,conjur-namespace, name in your cluster.
        This Golden ConfigMap contains the conjur connection details that can be used by all the supported workload set up.(https://docs.conjur.org/Latest/en/Content/Integrations/k8s-ocp/k8s-jwt-set-up-apps.htm#Supporte)

      #run the below chart with the required details.

            helm upgrade "cluster-prep" cyberark/conjur-config-cluster-prep  -n "conjur-namespace" \
            --create-namespace \
            --set conjur.account="conjur" \
            --set conjur.applianceUrl="https://conjur-conjur-oss.conjur-namespace.svc.cluster.local" \
            --set conjur.certificateBase64="${CONJUR_CERT_B64}" \
            --set authnK8s.authenticatorID="dev-cluster" \
            --set authnK8s.clusterRole.create=false \
            --set authnK8s.serviceAccount.create=false



10: prepare the application namespace and service account so that the workload can communicate with conjur and retrieve secrets
    once prepared the application namespace can contains the folowing resources:
        #conjur connection configMap
        # Authenticator RoleBinding

        i: run helm install command, replacing the parameters where relevant. the following will create test-app-namespace namespace

          helm install namespace-prep cyberark/conjur-config-namespace-prep \
          --create-namespace \
          --namespace test-app-namespace \
          --set conjurConfigMap.authnMethod="authn-jwt" \
          --set authnK8s.goldenConfigMap="conjur-configmap" \
          --set authnK8s.namespace="conjur-namespace" \
          --set authnRoleBinding.create="false"

        ii: Create the k8s service account for the workload.
            a: kubectl create serviceaccount test-app-sa -n test-app-namespace
            b: creat a token for test-app-sa
               kubectl create token test-app-sa -n test-app-namespace //token lifetime default 1hour
            c: get the cluster info using sa token,test.
                i: kubectl cluster-info //will give the cluster ip
                ii: curl https://clusterip/api --insecure --header "Authorixatio: Bearer <copy the toke here>" //outputs the cluster info
                          
 11: Define application as a conjur host policy and grant host permissons to the JWt Authenticator:
     #test-app.yaml
   
        - !policy
          id: app-path
          body:
          - !host
            id: system:serviceaccount:test-app-namespace:test-app-sa
            annotations:
              authn-jwt/dev-cluster/kubernetes.io/namespace: test-app-namespace
              authn-jwt/dev-cluster/kubernetes.io/serviceaccount/name: test-app-sa


        #grant the host permissions to the jwt authenticator
        - !grant
          roles:
          - !group conjur/authn-jwt/dev-cluster/apps
          members:
          - !host app-path/system:serviceaccount:test-app-namespace:test-app-sa
  
  conjur policy load -f test-app.yaml -b root

12: Define secrets and grant the application access to the secrets
     #app-secrets.yaml                                
      - !policy
        id: secrets
        body:
          - !group consumers
          - &variables
            - !variable username
            - !variable password
          - !permit
            role: !group consumers
            privilege: [ read, execute ]
            resource: *variables


      - !grant
        role: !group secrets/consumers
        member: !host app-path/system:serviceaccount:test-app-namespace:test-app-sa

  conjur policy load -f app-secrets.yaml -b root

13: populate secret values  
    i: conjur variable set -i secrets/username -v myUser
    ii: conjur variable set -i secrets/password -v MyP@ssw0rd!



14: deployment manifest file
    #replace with the req env variables 

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test-app
  name: test-app
  namespace: test-app-namespace
spec:
  selector:
    matchLabels:
      app: test-app
  replicas: 1
  template:
    metadata:
      labels:
        app: test-app
    spec:
      serviceAccountName: test-app-sa
      containers:
        - name: test-app
          image: test-app:test-app
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
      initContainers:
      - image: cyberark/conjur-authn-k8s-client
        imagePullPolicy: Always
        name: cyberark-secrets-provider-for-k8s
        volumeMounts:
        - name: jwt-token
          mountPath: /var/run/secrets/tokens
        env:
          - name: JWT_TOKEN_PATH
            value: /var/run/secrets/tokens/jwt
          - name: CONTAINER_MODE
            value: init
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: K8S_SECRETS
            value: db-credentials
          - name: SECRETS_DESTINATION
            value: k8s_secrets
        envFrom:
          - configMapRef:
              name: conjur-connect
      volumes:
      - name: jwt-token
        projected:
          sources:
            - serviceAccountToken:
                path: jwt
                expirationSeconds: 6000
                audience: https://kubernetes.default.svc.cluster.local
    
