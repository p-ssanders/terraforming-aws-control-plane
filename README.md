#   Control Plane on AWS

Reference Documentation: https://control-plane-docs.cfapps.io/

All commands assume `pwd` is `terraforming-control-plane`

1.  Terraform
    ```
    terraform init
    terraform plan -out=control-plane.tfplan
    terraform apply control-plane.tfplan
    ```

1.  Update DNS

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

1.  Create Control Plane Databases
    ```
    terraform output ops_manager_ssh_private_key > /tmp/opsmgrkey
    chmod 0400 /tmp/opsmgrkey
    ssh -i /tmp/opsmgrkey ubuntu@pcf.control.sam.pcftest.net

    psql --host <terraform output rds_address> --port <terraform output rds_port>  --username <terraform output rds_username> --dbname="postgres"
    <enter `terraform output rds_password`>
    create database atc;
    create database credhub;
    create database uaa;
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
