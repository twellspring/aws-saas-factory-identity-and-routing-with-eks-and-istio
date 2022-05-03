# SaaS Identity and Routing with Istio Service Mesh and Amazon EKS

The code shared here is intended to provide a sample implementation of a SaaS Identity and Routing solution based on Istio Service Mesh and Amazon EKS. The goal is to provide SaaS developers and architects with working code that will illustrate how multi-tenant SaaS applications can be design and delivered on AWS using Istio Service Mesh and Amazon EKS. The solution implements an identity model that simplifies the mapping of individual tenants and routing of traffic to isolated  tenant environments. The focus here is more on giving developers a view into the working elements of the solution without going to the extent of making a full, production-ready solution.

Note that the instructions below are intended to give you step-by-step, how-to instructions for getting this solution up and running in your own AWS account. For a general description and overview of the solution, please see the 
[blog post here](https://aws.amazon.com/blogs/apn/saas-identity-and-routing-with-istio-service-mesh-and-amazon-eks/).

## local workstation prerequisites
- aws cli
- eksctl
- istioctl (assumes ~/bin is already in your path)
  ```bash
  curl --no-progress-meter -L https://istio.io/downloadIstio | sh -
  cd istio-<istio version>
  bin/istioctl version
  cp -v bin/istioctl ~/bin
  ```
- pyyaml (`pip3 -q install PyYAML`)
- kubectl
- helm


## Setting up the environment

1. Setup workstation AWS environment
   - Do the normal awsapps.com login
   - Add credentials to terminal
   - export AWS_REGION=us-east-2
   - unset AWS_PROFILE
   - export PREFIX=`<your username>`
2. Clone/cd to repo
   ```bash
    git clone git@github.com:twellspring/aws-saas-factory-identity-and-routing-with-eks-and-istio.git
    cd aws-saas-factory-identity-and-routing-with-eks-and-istio
    git checkout remove_cloud9
    mkdir yaml
    export YAML_PATH=yaml
    ```
    <s>changes from original repo
    - global search/replace `#!/usr/bin/bash` --> `#!/usr/bin/env bash`
    - global search/replace `istio-ref-arch` --> `${PREFIX}-istio-ref-arch`
    - global search/replace `istio-saas` --> `${PREFIX}-istio-saas`
    </s>

5. Setup Keys (from setup.sh)

   ```bash
   ssh-keygen -t rsa -f ~/.ssh/istio-saas -N ''
   aws ec2 import-key-pair --key-name "${PREFIX}-istio-saas" --public-key-material fileb://~/.ssh/istio-saas.pub
   aws kms create-alias --alias-name alias/${PREFIX}-istio-ref-arch --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
   export MASTER_ARN=$(aws kms describe-key --key-id alias/${PREFIX}-istio-ref-arch --query KeyMetadata.Arn --output text)
   ```
6. Create the EKS Cluster
    * Run the following script to create a cluster configuration file, and subsequently provision the cluster using `eksctl`:

    ```bash
    chmod +x deploy1.sh
    ./deploy1.sh
    ```

    The cluster will take approximately 30 minutes to deploy.

    After EKS Cluster is set up, the script also deploys AWS Load Balancer Controller on the cluster.

7. Deploy Istio Service Mesh

    ```bash
    chmod +x deploy2.sh
    ./deploy2.sh
    ```

    This [script](./deploy2.sh) deploys the Istio Service Mesh demo profile, disabling the Istio Egress Gateway, while enabling the Istio Ingress Gateway along with Kubernetes annotations that signal the AWS Load Balancer Controller to automatically deploy a Network Load Balancer and associate it with the Ingress Gateway service.

8. Deploy Cognito User Pools
    > :warning: Close the terminal window that you create the cluster in, and open a new terminal before starting this step otherwise you may get errors about your AWS_REGION not set.
    * Open a **_NEW_** terminal window and `cd` back into `aws-saas-factory-identity-and-routing-with-eks-and-istio` and run the following script:

    ```bash
    chmod +x deploy-userpools.sh
    ./deploy-userpools.sh
    ```

    This [script](./deploy-userpools.sh) deploys Cognito User Pools for two (2) example tenants: tenanta and tenantb. Within each User Pool. The script will ask for passwords that will be set for each user.

    The script also generates the following YAML files for OIDC proxy configuration which will be deployed in the next step: 

    1. oauth2-proxy configuration for each tenant

    2. External Authorization Policy for Istio Ingress Gateway

9. Configure Istio Ingress Gateway
    > :warning: Close the terminal window that you create the cluster in, and open a new terminal before starting this step otherwise you may get errors about your AWS_REGION not set.
    * Open a **_NEW_** terminal window and `cd` back into `aws-saas-factory-identity-and-routing-with-eks-and-istio` and run the following script:

    ```bash
    chmod +x configure-istio.sh
    ./configure-istio.sh
    ```

    This [script](./configure-istio.sh) creates the following in preparation for configuring Istio Ingress Gateway:

    a. Self-signed Root CA Cert and Key

    b. Istio Ingress Gateway Cert signed by the Root CA

    It also completes the following steps:

    a. Creates TLS secret object for Istio Ingress Gateway Cert and Key

    b. Creates namespaces for Gateway, Envoy Reverse Proxy, OIDC Proxies, and the example tenants

    c. Deploys an Istio Gateway resource

    d. Deploys an Envoy reverse proxy
       - Create an ECR Repo for Envoy container image
       - Build Envoy container image adding the configuration YAML
       - Push the container image to the ECR Repo
       - Deploy the container image

    e. Deploy oauth2-proxy along with the configuration generated in the Step 8

    f. Adds an Istio External Authorization Provider definition pointing to the Envoy Reverse Proxy

10. Deploy Tenant Application Microservices
     > :warning: Close the terminal window that you create the cluster in, and open a new terminal before starting this step otherwise you may get errors about your AWS_REGION not set.
     * Open a **_NEW_** terminal window and `cd` back into `aws-saas-factory-identity-and-routing-with-eks-and-istio` and run the following script:

     ```bash
     chmod +x deploy-tenant-services.sh
     ./deploy-tenant-services.sh
     ```

     This [script](./deploy-tenant-services.sh) creates the service dpeloyments for the two (2) sample tenants, along with Istio VirtualService constructs that define the routing rules.

11. Once finished running all the above steps, the bookinfo app can be accessed using the following steps.

    a. Since the sample tenants are built using the DNS domain example.com, domain name entries are made into the local desktop/laptop hosts file. For Linux/MacOS the file is /etc/hosts and on Windows it is C:\Windows\System32\drivers\etc\hosts.

    b. Wait for the Network Load Balancer instance status, in AWS Management Console, to change from Provisioning to Active.

    c. Run the following command in the Cloud9 shell
    ```bash
    chmod +x hosts-file-entry.sh
    ./hosts-file-entry.sh
    ```

    d. Append the output of the command into the local hosts file. It identifies the load balancer instance associated with the Istio Ingress Gateway, and looks up the public IP addresses assigned to it.

    e. To avoid TLS cert conflicts, configure the browser on desktop/laptop with a new profiles

    f. The browser used to test this deployment was Mozilla Firefox, in which a new profile can be created by pointing the browser to "about:profiles"

    g. Create a new profile, such as, "istio-saas"

    h. After creating the profile, click on the "Launch profile in new browser"

    i. In the browser, open two tabs, one for each of the following URLs:

    ```
       https://tenanta.example.com/bookinfo

       https://tenantb.example.com/bookinfo
    ```

    j. Because of self-signed TLS certificates, you may received a certificate related error or warning from the browser

    k. When the login prompt appears:

       In the browser windows with the "istio-saas" profile, login with:

    ```
       user1@tenanta.com

       user1@tenantb.com
    ```
       This should result in displaying the bookinfo page

12. Tenant Onboarding

    a. Add User Pools for new tenants
    
    ```bash
    chmod +x add-userpools.sh
    ./add-userpools.sh
    ```

    b. Re-configure Istio Ingress Gateway and Envoy Reverse Proxy
    
    ```bash
    chmod +x update-istio-config.sh
    ./update-istio-config.sh
    ```
    c. Deploy new tenant's microservices

    ```bash
    chmod +x update-tenant-services.sh
    ./update-tenant-services.sh
    ```
    d. Run the following command in the Cloud9 shell
    ```bash
    chmod +x update-hosts-file-entry.sh
    ./update-hosts-file-entry.sh
    ```

    e. Append the output of the command into the local hosts file. It identifies the load balancer instance associated with the Istio Ingress Gateway, and looks up the public IP addresses assigned to it.

    f. In the browser window with the "istio-saas" profile, open another tab for:

    ```
       https://tenantc.example.com/bookinfo
    ```

    g. Because of self-signed TLS certificates, you may received a certificate related error or warning from the browser

    h. When the login prompt appears, login with:

    ```
       user1@tenantc.com
    ```

       This should result in displaying the bookinfo page

## Cleanup

1. The deployed components can be cleaned up by running the following:

    ```bash
    chmod +x cleanup.sh
    ./cleanup.sh
    ```

    This [script](./cleanup.sh) will 

    a. Delete the Cognito User Pools and the assoicated Hosted UI domains

    b. Uninstall AWS Load Balancer Controller

    c. Delete the EKS Cluster

    d. Disable the KMS Master Key and removes the alias

    e. Delete EC2 Key-Pair

2. You can also delete

    a. The EC2 Instance Role `istio-ref-arch-admin`

    b. The Cloud9 Environment

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

