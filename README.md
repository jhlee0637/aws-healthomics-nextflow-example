# aws-healthomics-nextflow-example
* Youtube video
* [Google Slides](https://docs.google.com/presentation/d/13Ew3Qx_GnlWHyE12p0sSXvBqXysa_5N131U1EmZB8h0/edit?usp=sharing)

> Please remind that you need to build your cloud environment in advance

### Declare Variables
```
region=$(ec2-metadata --availability-zone | sed 's/placement: \(.*\).$/\1/')

account_number=$(aws sts get-caller-identity --query 'Account' --output text)
```

### Set up AWS CDK (Cloud Development Kit)
```
cdk bootstrap aws://$account_number/$region
```

### Clone aws-healthomics-tutorials git
```
cd ~
git clone https://github.com/aws-samples/aws-healthomics-tutorials.git
```

### Install an Automation Application for HealthOmics Manifest Conversion
```
cd ~
git clone https://github.com/aws-samples/amazon-ecr-helper-for-aws-healthomics.git
cd amazon-ecr-helper-for-aws-healthomics
npm install
cdk deploy --all
```

### Clone Sarek & Generate the Docker Image Manifest file Example: `sarek`
```
cd ~
git clone https://github.com/nf-core/sarek.git

cp  ~/amazon-ecr-helper-for-aws-healthomics/lib/lambda/parse-image-uri/public_registry_properties.json \ namespace.config

cd ~
python3 aws-healthomics-tutorials/utils/scripts/inspect_nf.py \
--output-manifest-file sarek_dev_docker_images_manifest.json \
 -n namespace.config \
 --output-config-file omics.config \
 --region $region \
 ~/sarek/
```
1. check the 'sarek_dev_docker_images_manifest.json` file
2. [check the `omics.config` file](./sarek-config/omics.config)
### Execute the Manifest Conversion via AWS Step Functions
```
aws stepfunctions start-execution \
--state-machine-arn arn:aws:states:$region:$account_number:stateMachine:omx-container-puller \
--input file://sarek_dev_docker_images_manifest.json
```

### Update Pipeline Config
```
mv omics.config sarek/conf
echo "includeConfig 'conf/omics.config'" >> sarek/nextflow.config
```

### Workflow Staging Example: `sarek_ECR`
```
export yourbucket="my-test-bucket-527778419916" 

zip -r sarek_ECR.zip sarek/ -x "*/\.*" "*/\.*/**"

aws s3 cp sarek_ECR.zip s3://${yourbucket}/workshop/sarek_ECR.zip  
```

### Register Workflow Example: `sarek_ECR`
```
export workflow_name="sarek_ECR"
aws omics create-workflow \
  --name ${workflow_name} \
  --definition-uri s3://${yourbucket}/workshop/${workflow_name}.zip \
  --engine NEXTFLOW \
  #--parameter-template file://parameter-description.json \
```

### Check Generated Workflow
```
workflow_id=$(aws omics list-workflows --name ${workflow_name} --query 'items[0].id' --output text)
echo $workflow_id
```

### Uploading necessary files to S3 for the test run
1. [download necessary files for sarek test profile](./script/sarek-test-profile-data-download.sh)
2. modify the sample sheet file
3. upload them to your `S3` bucket

### Declare Variable Example

```
export workflow_name="sarek_ECR"
export workflow_id=$(aws omics list-workflows --name ${workflow_name} --query 'items[0].id' --output text)
export yourbucket="my-test-bucket-527778419916"
export your_account_id="527778419916"
export omics_role_name="OmicsUnifiedJobRole"

echo $workflow_id
echo $yourbucket
echo $your_account_id
echo $omics_role_name
```

### Create an AWS IAM role

```
aws iam create-role --role-name ${omics_role_name} --assume-role-policy-document file://trust_policy.json
```

### Attach a policy to a role
```
aws iam put-role-policy --role-name ${omics_role_name} --policy-name OmicsWorkflowV1 --policy-document file://omics_workflow_policy.json
```

### Run workflow Example
```
aws omics start-run \
  --name sarek_official_test\
  --role-arn arn:aws:iam::${your_account_id}:role/${omics_role_name}\
  --workflow-id ${workflow_id} \
  --parameters file://input.json \
  --output-uri s3://${yourbucket}/workflow-output/
```

