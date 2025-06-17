# EMR Cluster Provisioning with Hail Support on AWS

This project automates the deployment of Amazon EMR clusters optimized for genomic data analysis using Hail. It provides infrastructure-as-code templates for creating custom AMIs and provisioning EMR clusters with Spot and AWS Graviton instances for cost-effective processing of large-scale genetic data. The solution has been tested with EMR version 7.5.0 and Hail 0.2.134.

The solution consists of two main components:
1. A custom AMI builder that creates an Amazon Linux 2023 ARM64 image pre-configured with Hail dependencies
2. An EMR cluster provisioning system that leverages spot instances and supports ARM architectures for optimal price-performance ratio

## Repository Structure
```
.
├── simplified-ami-builder.yaml                     # CloudFormation template for building custom AMIs
├── install-emr.yaml                                # Main EMR cluster provisioning template
├── buildspec.yml                                   # AWS CodeBuild configuration for AMI creation

## Usage Instructions
### Prerequisites
- AWS CLI installed and configured with appropriate permissions
- Access to AWS services:
  - Amazon EMR
  - AWS CodeBuild
  - Amazon EC2
  - Amazon S3
  - AWS IAM
- An existing VPC with appropriate subnets
- An EC2 key pair for SSH access

### Installation

1. Build the Custom AMI:
```bash
# Deploy the AMI builder CloudFormation stack
aws cloudformation create-stack \
  --stack-name hail-ami-builder \
  --template-body file://simplified-ami-builder.yaml \
  --parameters \
    ParameterKey=AmiName,ParameterValue=hail-emr-ami \
    ParameterKey=HailVersion,ParameterValue=0.2.134 \
  --capabilities CAPABILITY_NAMED_IAM
```

2. Deploy the EMR Cluster:
Adjust the instance selection and diversify them across ARM64 families and sizes to minimise Spot interruptions.
```bash
# Deploy the EMR cluster using the custom AMI
aws cloudformation create-stack \
  --stack-name hail-emr-cluster \
  --template-body file://install-emr.yaml \
  --parameters \
    ParameterKey=EMRClusterName,ParameterValue=hail-cluster \
    ParameterKey=SpotBidPercentage,ParameterValue=100
```

### Quick Start
1. Create a custom AMI with Hail dependencies:
```bash
# Trigger the AMI build
aws codebuild start-build --project-name <stack-name>-ami-builder
```

2. Wait for the AMI build to complete and note the AMI ID from the Systems Manager Parameter Store.

3. Deploy an EMR cluster using the custom AMI:
```bash
# Deploy with default parameters
aws cloudformation create-stack \
  --stack-name my-hail-cluster \
  --template-body file://install-emr.yaml
```

4. Test the cluster with a hail job. Create hail-script.py with the following content:
```bash
import hail as hl
mt = hl.balding_nichols_model(n_populations=3,
n_samples=500,
n_variants=500_000,
n_partitions=32)
mt = mt.annotate_cols(drinks_coffee = hl.rand_bool(0.33))
gwas = hl.linear_regression_rows(y=mt.drinks_coffee,
x=mt.GT.n_alt_alleles(),
covariates=[1.0])
gwas.order_by(gwas.p_value).show(25)
```

5. Login to the cluster primary node as Hadoop user and run the spark job.

```bash
export HADOOP_CONF_DIR=/etc/hadoop/conf
sudo update-alternatives --set java /usr/lib/jvm/java-11-amazon-corretto.aarch64/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/java-11-amazon-corretto.aarch64/bin/javac
HAIL_HOME=$(pip3 show hail | grep Location | awk -F' ' '{print $2 "/hail"}')
spark-submit \
  --master yarn \
  --jars $HAIL_HOME/backend/hail-all-spark.jar \
  --conf spark.driver.extraClassPath=$HAIL_HOME/backend/hail-all-spark.jar \
  --conf spark.executor.extraClassPath=./hail-all-spark.jar \
  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
  --conf spark.kryo.registrator=is.hail.kryo.HailKryoRegistrator \
  hail-script.py
  ```