# AWS Lambda + Pandas Layer (Docker Method) + Query Filter API

## URL-https://3r5sorsh752ro7rtliweu5bmwi0qorot.lambda-url.ap-south-1.on.aws/

## 1. Objective

The goal of this task was to successfully run Pandas inside AWS Lambda, which is not available by default, by building a custom layer using Docker.
We then expose a working Lambda function through a Function URL so it behaves like an API endpoint.
Finally, we add query parameter filtering and clean error handling.

The entire workflow below represents the final approach that worked reliably end-to-end.

## 2. Why Docker for Lambda Layers?

Lambda requires packages compiled for Amazon Linux 2, not macOS or Windows.
Docker ensures we build Pandas and NumPy in the same environment Lambda runs in.
This avoids import errors like:

Unable to import required dependencies: numpy


Using Docker gives a predictable and clean build every time.

## 3. Building the Pandas Layer (Docker Method)
Step 1 — Create a project folder
mkdir lambda-pandas-docker
cd lambda-pandas-docker

Step 2 — Create requirements.txt
echo "pandas==2.3.3" > requirements.txt
echo "numpy==2.2.6" >> requirements.txt

Step 3 — Start Lambda-compatible Docker container
docker run --platform linux/amd64 -it \
  --entrypoint /bin/bash \
  -v "$PWD":/var/task \
  public.ecr.aws/lambda/python:3.10

Step 4 — Install dependencies inside the container
pip install -r requirements.txt -t python


You should now see a python/ folder containing Pandas, NumPy, and dependencies.

Step 5 — Exit the container
exit

Step 6 — Zip the layer
zip -r pandas-layer-docker.zip python


This ZIP file is your working Lambda layer.

## 4. Upload Layer to S3

Open S3 Console

Upload pandas-layer-docker.zip

Note the S3 URI for reference

## 5. Create Lambda Layer

AWS Console → Lambda → Layers → Create layer

Name: pandas-layer-docker

Upload the ZIP file (choose “Upload a file from Amazon S3” if needed)

Runtime: Python 3.10

Create

## 6. Create Lambda Function

Name: pandas-api-docker

Runtime: Python 3.10

## 7. Attach the Layer to the Function

Lambda → Configuration → Layers → Add layer
Choose: Custom Layers → pandas-layer-docker

## 8. Lambda Function Code 

import json
import pandas as pd

def lambda_handler(event, context):
    try:
        df = pd.DataFrame({
            "name": ["aman", "lambda", "docker"],
            "score": [10, 20, 30]
        })

        params = event.get("queryStringParameters") or {}
        name_query = params.get("name")

        if name_query:
            df = df[df["name"].str.lower() == name_query.lower()]

        rows = df.to_dict(orient="records")

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "success": True,
                "message": "Pandas layer (Docker) is working!",
                "filtered": bool(name_query),
                "rows": rows
            })
        }

    except Exception as e:
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "success": False,
                "error": str(e)
            })
        }

## 9. Create API Endpoint (Function URL)

Enable Function URL:

Auth: NONE (public)

Copy the generated URL
Example:

https://xxxxxx.lambda-url.ap-south-1.on.aws/

## Output Examples

### Below are three output screenshots


A) Output without query param 

https://3r5sorsh752ro7rtliweu5bmwi0qorot.lambda-url.ap-south-1.on.aws/

![Output Examples](./out11.png)


B) Output with query param (?name=aman)
https://3r5sorsh752ro7rtliweu5bmwi0qorot.lambda-url.ap-south-1.on.aws/?name=aman

![Output Examples](./out12.png)



C) Output with unmatched query param (?name=xyz) (out13)
https://3r5sorsh752ro7rtliweu5bmwi0qorot.lambda-url.ap-south-1.on.aws/?name=xyz

![Output Examples](./out13.png)


