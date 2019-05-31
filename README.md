#   Control Plane on AWS

Reference Documentation: https://control-plane-docs.cfapps.io/

All commands assume `pwd` is `terraforming-control-plane` unless directed otherwise.

1.  Terraform

    *   Create a `terraform.tfvars` file using `terraform.tfvars-example` as a template

    *   Terraform
        ```
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
    om configure-authentication -u $OM_USERNAME -p $OM_PASSWORD -dp keepitsimple
    om update-ssl-certificate --certificate-pem "$(cat ../certs/fullchain.pem)" --private-key-pem "$(cat ../certs/privkey.pem)"
    unset OM_SKIP_SSL_VALIDATION
    ```

1.  Configure BOSH Director

    ```
    om configure-director -c ../config/bosh-director.yml --vars-file ../variables/bosh-director.yml --vars-file ../secrets/bosh-director.yml
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
        ssh -i /tmp/opsmgrkey ubuntu@pcf.control.sam.pcftest.net
        ```

    *   Login to Postgres
        ```
        psql --host <terraform output rds_address> --port <terraform output rds_port>  --username <terraform output rds_username> --dbname="postgres"
        <enter `terraform output rds_password`>
        ```

    *   Create the databases for Control Plane
        ```
        create database atc;
        create database credhub;
        create database uaa;
        ```

1.  Upload Control Plane Components BOSH Releases

    *   _(optional)_ SSH to OpsManager VM for better network performance

    *   Get Pivnet CLI
        ```
        sudo su -
        wget https://github.com/pivotal-cf/pivnet-cli/releases/download/v0.0.58/pivnet-linux-amd64-0.0.58
        chmod +x pivnet-linux-amd64-0.0.58
        cp pivnet-linux-amd64-0.0.58 /usr/bin/pivnet
        ```

    *   Download the Control Plane Components BOSH Releases
        ```
        pivnet login --api-token <YOUR_PIVNET_TOKEN>

        pivnet accept-eula -p p-control-plane-components -r 0.0.34

        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g uaa-release-69.0-315.34.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g garden-runc-release-1.19.1-315.34.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g credhub-release-1.9.9-315.34.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g concourse-release-4.2.4-315.34.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g postgres-release-37-315.34.tgz && \
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g bosh-dns-aliases-release-0.0.3-315.34.tgz
        ```

    *   Download the Stemcell
        ```
        pivnet release-dependencies -p p-control-plane-components -r 0.0.34
        pivnet accept-eula -p stemcells-ubuntu-xenial -r 315.34
        pivnet download-product-files -p stemcells-ubuntu-xenial -r 315.34 -g light-bosh-stemcell-315.34-aws-xen-hvm-ubuntu-xenial-go_agent.tgz
        ```

    *   Upload Stemcell & Releases to BOSH

        ```
        eval "$(om bosh-env --ssh-private-key /tmp/opsmgrkey)"

        bosh upload-stemcell *bosh-stemcell*.tgz

        bosh upload-release uaa-release-*.tgz && \
        bosh upload-release garden-runc-release-*.tgz && \
        bosh upload-release credhub-release-*.tgz && \
        bosh upload-release concourse-release-*.tgz && \
        bosh upload-release postgres-release-*.tgz && \
        bosh upload-release bosh-dns-aliases-release-*.tgz
        ```

1.  Deploy the Control Plane Components via Manifest

    *   Download the Manifest
        ```
        pivnet download-product-files -p p-control-plane-components -r 0.0.34 -g control-plane-0.0.34-rc.3.yml -d ../config/
        ```

    *   Deploy the Manifest
        * Note: many of the secrets can be obtained from BOSH's Credhub via `credhub find -n <DEPLOYMENT NAME>`

        ```
        eval "$(om bosh-env --ssh-private-key /tmp/opsmgrkey)"

        bosh deploy -d control-plane ../config/control-plane*.yml --vars-file=../variables/control-plane.yml --vars-file=../secrets/control-plane.yml --ops-file=../operations/control-plane.yml
        ```

1.  Validate Deployment

    *   Get the Concourse Admin Password
        ```
        eval "$(om bosh-env --ssh-private-key /tmp/opsmgrkey)"
        credhub get -n $(credhub find | grep uaa_users_admin | awk '{print $3}')
        ```

    *   Login to Concourse https://plane.control.sam.pcftest.net using `admin/<THE ADMIN PASSWORD>`