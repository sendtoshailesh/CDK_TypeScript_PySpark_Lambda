### Using CDK with TypeScript for PySpark on Lambda
This repor demostrate how to use AWS CDK with TypeScript to deploy PySpark Lambda functions instead of AWS SAM CLI templates. This approach allows you to maintain your standard coding paradigm using CDK and TypeScript while implementing Lambda functions with PySpark capabilities.

## Implementing PySpark Lambda with CDK

Set up a PySpark Lambda function using CDK with TypeScript:

``` typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

export class PySparkLambdaStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create a role for the Lambda function
    const lambdaRole = new iam.Role(this, 'PySparkLambdaRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
        iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonS3ReadOnlyAccess')
      ]
    });

    // Create a Lambda layer for PySpark dependencies
    const pysparkLayer = new lambda.LayerVersion(this, 'PySparkLayer', {
      code: lambda.Code.fromAsset('layers/pyspark'),
      compatibleRuntimes: [lambda.Runtime.PYTHON_3_9],
      description: 'PySpark and its dependencies',
    });

    // Create the Lambda function
    const pysparkFunction = new lambda.Function(this, 'PySparkFunction', {
      runtime: lambda.Runtime.PYTHON_3_9,
      code: lambda.Code.fromAsset('lambda'),
      handler: 'pyspark_handler.handler',
      memorySize: 3008, // PySpark needs more memory
      timeout: cdk.Duration.minutes(5),
      role: lambdaRole,
      layers: [pysparkLayer],
      environment: {
        PYTHONPATH: '/opt:/opt/python:/var/task',
        PYSPARK_PYTHON: '/var/lang/bin/python3'
      }
    });

    // Output the function ARN
    new cdk.CfnOutput(this, 'PySparkLambdaArn', {
      value: pysparkFunction.functionArn,
      description: 'The ARN of the PySpark Lambda function',
    });
  }
}

```

### Creating the PySpark Layer
For the PySpark layer, you'll need to prepare a directory structure with all the required dependencies:

```
Create a layers/pyspark/python directory structure
Install PySpark and its dependencies in that directory:
```

### Sample PySpark Lambda Handler

``` python
import os
from pyspark.sql import SparkSession

def handler(event, context):
    # Initialize Spark session
    spark = SparkSession.builder \
        .appName("PySparkLambda") \
        .config("spark.executor.memory", "512m") \
        .config("spark.driver.memory", "512m") \
        .config("spark.python.worker.memory", "512m") \
        .master("local[*]") \
        .getOrCreate()
    
    # Create a simple DataFrame
    data = [("Alice", 34), ("Bob", 45), ("Charlie", 29)]
    df = spark.createDataFrame(data, ["Name", "Age"])
    
    # Perform some operations
    result = df.select("Name", "Age").collect()
    
    # Stop the Spark session
    spark.stop()
    
    # Return the results
    return {
        'statusCode': 200,
        'body': str(result)
    }

```

For more complex scenarios, you can extend your CDK code to include:

``` typescript
// Add VPC configuration if needed
const pysparkFunction = new lambda.Function(this, 'PySparkFunction', {
  // ... other properties
  vpc: vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  
  // Increase ephemeral storage for PySpark processing
  ephemeralStorageSize: cdk.Size.mebibytes(10240),
  
  // Add architecture specification
  architecture: lambda.Architecture.X86_64,
});

// Add event sources if needed
pysparkFunction.addEventSource(new SomeEventSource(...));

```

### To deploy your CDK stack:

npm run build   # Compile TypeScript to JavaScript
cdk deploy      # Deploy the stack to AWS


refer more here:

https://docs.aws.amazon.com/lambda/latest/dg/typescript-package.html#aws-cdk-ts

