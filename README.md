#   Control Plane with Let's Encrypt Certificates on AWS

This repository contains a guide and supporting files to deploy a Control Plane with Let's Encrypt Certificates on AWS.

1.  Generate Let's Encrypt Certificates

    ```
    sudo certbot \
    --server https://acme-v02.api.letsencrypt.org/directory \
    -d control.fionathebluepittie.com \
    -d *.control.fionathebluepittie.com \
    --manual --preferred-challenges dns-01 certonly
    ```

1.  "Pave the IaaS"

    *   Create a `terraform.tfvars` file using `terraform.tfvars-example` as a template

    *   Terraform
        ```
        cd terraforming-control-plane
        terraform init
        terraform plan -out=control-plane.tfplan
        terraform apply control-plane.tfplan
        ```

1.  Update DNS

    **TODO** terraform should do this

    https://docs.pivotal.io/pivotalcf/2-5/om/aws/prepare-env-terraform.html#dns

1.  Configure OpsManager

    ```
    export OM_TARGET=https://$(terraform output ops_manager_dns)
    export OM_USERNAME=admin
    export OM_PASSWORD=$(openssl rand -base64 10)
    export OM_SKIP_SSL_VALIDATION=true

    om configure-authentication \
    -u $OM_USERNAME \
    -p $OM_PASSWORD \
    -dp keepitsimple

    om update-ssl-certificate \
    --certificate-pem "$(cat certs/fullchain.pem)" \
    --private-key-pem "$(cat certs/privkey.pem)"

    unset OM_SKIP_SSL_VALIDATION
    ```

1.  Configure BOSH Director

    *   Update the `variables/bosh-director.yml` file, and create the `secrets/bosh-director.yml` file using `secrets/bosh-director.yml-example` as a template

    *   Configure the BOSH Director
        ```
        om configure-director \
        -c config/bosh-director.yml \
        --vars-file variables/bosh-director.yml \
        --vars-file secrets/bosh-director.yml
        ```

1.  Apply Changes

    ```
    om pending-changes
    om apply-changes
    ```

1.  Create Control Plane Databases

    *   SSH to OpsManager VM
        ```
        terraform output ops_manager_ssh_private_key > /tmp/opsmgrkey
        chmod 0400 /tmp/opsmgrkey
        ssh -i /tmp/opsmgrkey ubuntu@$(terraform output ops_manager_dns)
        ```

    *   Login to Postgres
        ```
        psql \
        --host <terraform output rds_address> \
        --port <terraform output rds_port> \
        --username <terraform output rds_username> \
        --dbname="postgres"

        <enter `terraform output rds_password`>
        ```

    *   Create the databases for Control Plane
        ```
        create database atc;
        create database credhub;
        create database uaa;
        ```

1.  Upload Control Plane Components BOSH Releases

    *   SSH to OpsManager VM
        ```
        ssh -i /tmp/opsmgrkey ubuntu@$(terraform output ops_manager_dns)
        ```

    *   Install Pivnet CLI
        ```
        sudo su -
        wget https://github.com/pivotal-cf/pivnet-cli/releases/download/v0.0.58/pivnet-linux-amd64-0.0.58
        chmod +x pivnet-linux-amd64-0.0.58
        cp pivnet-linux-amd64-0.0.58 /usr/bin/pivnet
        ```

    *   Download the Control Plane Components BOSH Releases
        ```
        pivnet login --api-token <YOUR_PIVNET_TOKEN>

        pivnet accept-eula -p p-control-plane-components -r 0.0.31

        pivnet download-product-files -p p-control-plane-components -r 0.0.31 -g uaa-release-69.0-250.17.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.31 -g garden-runc-release-1.18.2-250.17.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.31 -g credhub-release-1.9.9-250.17.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.31 -g concourse-release-4.2.3-250.17.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.31 -g postgres-release-36-250.17.tgz
        ```

    *   Download the Stemcell
        ```
        pivnet release-dependencies -p p-control-plane-components -r 0.0.31

        pivnet accept-eula -p stemcells-ubuntu-xenial -r 250.17

        pivnet download-product-files \
        -p stemcells-ubuntu-xenial \
        -r 250.17 \
        -g light-bosh-stemcell-250.17-aws-xen-hvm-ubuntu-xenial-go_agent.tgz
        ```

    *   Install [`om` CLI](https://github.com/pivotal-cf/om#installation)

    *   Upload Stemcell & Releases to BOSH

        ```
        eval "$(om bosh-env --ssh-private-key /tmp/opsmgrkey)"

        bosh upload-stemcell *bosh-stemcell*.tgz

        bosh upload-release uaa-release-*.tgz && \
        bosh upload-release garden-runc-release-*.tgz && \
        bosh upload-release credhub-release-*.tgz && \
        bosh upload-release concourse-release-*.tgz && \
        bosh upload-release postgres-release-*.tgz
        ```

1.  Deploy the Control Plane Components via Manifest

    *   SSH to OpsManager VM
        ```
        ssh -i /tmp/opsmgrkey ubuntu@$(terraform output ops_manager_dns)
        sudo su -
        cd /home/ubuntu
        ```

    *   Download the Manifest
        ```
        pivnet download-product-files \
        -p p-control-plane-components \
        -r 0.0.31 \
        -g control-plane-0.0.31-rc.1.yml \
        -d config/
        ```

    *   Convert Let's Encrypt Private Key to RSA
        ```
        openssl rsa -in private_key.pem -out private_key.pem.rsa.key
        ```

    *   Create Entries in BOSH's Credhub

        * `ca.pem` contains the contents of `certs/chain.pem`
        * `certificate.pem` contains the contents of `certs/cert.pem`
        * `private_key.pem` contains the contents of `certs/private_key.pem`
        ```
        eval "$(om bosh-env)"

        credhub set -n /p-bosh/control-plane/lets_encrypt_cert \
        -t certificate \
        -r "$(cat ca.pem)" \
        -c "$(cat cert.pem)" \
        -p "$(cat private_key.pem.rsa.key)"
        ```

    *   Exit OpsManager VM

    *   Deploy the Manifest
        ```
        eval "$(om bosh-env --ssh-private-key /tmp/opsmgrkey)"

        bosh update-cloud-config config/cloud-config.yml

        bosh deploy \
        -d control-plane config/control-plane*.yml \
        --vars-file=variables/control-plane.yml \
        --ops-file=operations/control-plane.yml
        ```

1.  Validate Deployment

    *   Get the Concourse Admin Password
        ```
        eval "$(om bosh-env)"
        credhub get -n $(credhub find | grep uaa_users_admin | awk '{print $3}')
        ```

    *   Login to Concourse https://plane.control.fionathebluepittie.com using `admin/<THE ADMIN PASSWORD>`