# Event driven function for converting text to PDF

This event driven function converts a text file to PDF. Once you drop a `text` file into an Oracle Cloud Infrastructure Object Storage Bucket and configure the appropriate trigger rule, the function will convert it into PDF and store the converted file (with a `.pdf` extension) in an output bucket specified by the user 

- The function written in Go and uses [gofpdf](https://github.com/jung-kurt/gofpdf) for text to PDF conversion 
- Uses the [OCI Go SDK](https://github.com/oracle/oci-go-sdk) to execute Object Storage read and write operations
- A custom `Dockerfile` is used to build the function

## Pre-requisites

- Collect the following information for you OCI tenancy (you'll need these in subsequent steps) - Tenancy OCID, User OCID of a user in the tenancy, OCI private key, OCI public key passphrase, OCI region
- Copy your OCI private key to this folder. If you don't already have one, [please follow the documentation](https://docs.cloud.oracle.com/iaas/Content/API/Concepts/apisigningkey.htm#How)


## Create application

`fn create app text2pdf --annotation oracle.com/oci/subnetIds=<SUBNETS> --config TENANT_OCID=<TENANT_OCID> --config USER_OCID=<USER_OCID> --config FINGERPRINT=<FINGERPRINT> --config PASSPHRASE=<PASSPHRASE> --config REGION=<REGION> --config PRIVATE_KEY_NAME=<PRIVATE_KEY_NAME> --config OUTPUT_BUCKET=<OUTPUT_BUCKET>`

> the user credentials you provide here (using `USER_OCID`, `FINGERPRINT` etc. should have read and write access to the specified Object Storage bucket)

e.g.

`fn create app text2pdf --annotation oracle.com/oci/subnetIds='["ocid1.subnet.oc1.phx.aaaaaaaaghmsma7mpqhqdhbgnby25u2zo4wqlrrcskvu7jg56dryxt3hgvka"]' --config TENANT_OCID=ocid1.tenancy.oc1..aaaaaaaaydrjm77otncda2xn7qtv7l3hqnd3zxn2u6siwdhniibwfv4wwhta --config USER_OCID=ocid1.user.oc1..aaaaaaaavz5efq7jwjjipbvm536plgylg7rfr53obvtghpi2vbg3qyrnrtfa --config FINGERPRINT=41:82:5f:44:ca:a1:2e:58:d2:63:6a:af:52:d5:3d:04 --config PASSPHRASE=4242 --config REGION=us-phoenix-1 --config PRIVATE_KEY_NAME=oci_private_key.pem --config OUTPUT_BUCKET=output-bucket`

## Deploy the application

- `cd fn-text2pdf` 
- `fn -v deploy --app text2pdf --build-arg PRIVATE_KEY_NAME=<private_key_name>` e.g. `fn -v deploy --app text2pdf --build-arg PRIVATE_KEY_NAME=oci_private_key.pem`

## Obtain the function OCID

Let's find the function OCID (use the command below) and replace it in `actions.json` file

`fn inspect fn text2pdf convert | jq '.id' | sed -e 's/^"//' -e 's/"$//'`

## Create the rule using OCI CLI

Let's create the rule using OCI CLI

`oci --profile <oci-config-profile-name> cloud-events rule create --display-name <display-name> --is-enabled true --condition '{"eventType":"com.oraclecloud.objectstorage.object.create", "data": {"bucketName":"<bucket-name>"}}' --compartment-id <compartment-ocid> --actions file://<filename>.json`

Replace `<bucket-name>` with the (input) Object Storage bucket name where you will upload the text file

e.g.

`oci events rule create --display-name t2prule --is-enabled true --condition '{"eventType":"com.oraclecloud.objectstorage.createobject", "data": {"bucketName":"test"}}' --compartment-id ocid1.compartment.oc1..aaaaaaaaokbzj2jn3hf5kwdwqoxl2dq7u54p3tsmxrjd7s3uu7x23tkegiua --actions file://actions.json`


## Test

A sample text file (`lorem.txt`) has been provided to test the function. Upload this file to your object storage bucket and wait for the function to be triggered.

If successful, you should see a PDF (`lorem.pdf`) in the **output** Object Storage bucket spoecified by the user
