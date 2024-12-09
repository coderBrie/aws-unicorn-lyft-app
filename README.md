# AWS Project - Build a Full End-to-End Web Application with 7 Services | Step-by-Step Tutorial

We're creating a web application for a unicorn ride-sharing service called Wild Rydes (from the original [Amazon workshop](https://aws.amazon.com/serverless-workshops)).  The app uses IAM, Amplify, Cognito, Lambda, API Gateway and DynamoDB, with code stored in GitHub and incorporated into a CI/CD pipeline with Amplify.

Here is the link  to live the app:
https://main.d1kz8duxcgvpqz.amplifyapp.com/

![Image Alt](https://github.com/coderBrie/aws-unicorn-lyft-app/blob/cb69d6a461f18a68a0d1cd9cdbe47832a05c26ff/Web-App-Architecture-Diagram.jpg)

## How to use the app
Furst
The app will let you create an account by entering your name and email address. 

Next
The app will send you an authentication code to your email. 

On the app: Renter your email address and the authentication code from your email. 

Click verify to Verify your account

Next
Sign in with your email and  the password you created

After log in: request a ride by clicking anywhere on the map (powered by ArcGIS) then click Set pickup to request ride 

and a unicorn with gallap over to your location. 


## Manifesto
In this tuorial I wanted to focus on the big picture: building a cost optimizing, highly available, secure application that can scale to support millions of users, with opportunities for future enhancements like ride tracking or group ride capabilities. 
 
I believe in rapid iteration and deployment, taking action to ensure timely delivery of valuable features. 

I take full ownership for the app's performance, monitoring it continuously and implementing improvements as needed.
 
Here are some of the benefits to this apps cloud architecture:

## Scalability
Lambda Scaling: I leveraged AWS Lambda's auto-scaling capabilities to ensure that the backend automatically adjusts to handle fluctuations in user requests.

DynamoDB Auto Scaling: DynamoDB's auto-scaling feature is enabled to dynamically adjust read and write capacity, ensuring efficient handling of varying traffic patterns.

Future Scalabilty:
CloudFront for Global Distribution: Integrate Amazon CloudFront to distribute content globally and reduce latency for end users.

API Gateway Throttling: Throttling limits are set on API Gateway to handle traffic surges while protecting the backend from overload.


## High Availability
Multi-AZ DynamoDB: DynamoDB inherently provides multi-AZ data replication, ensuring high availability and data durability.

Lambda Regional Deployment: To minimize downtime, I plan to deploy Lambda functions in multiple regions, enabling regional failover if required.

Amplify Hosting: AWS Amplify hosting ensures automatic scaling and self-healing for the frontend.
If the company goes global we can use route 53 and add disastery recovery.



## App Security
IAM Policies: The app uses IAM roles and policies to enforce least privilege access, protecting sensitive resources.

Cognito Authentication: Cognito handles user sign-up, sign-in, and access controls, ensuring user authentication is secure.

API Gateway Authorizers: I’ve implemented authorizers to validate incoming API requests and restrict access to unauthorized users.

Environment Variables: Sensitive information such as database credentials is stored securely in Lambda environment variables.

## Future Security Enhancements 
Encryption: I will ensure that all data stored in DynamoDB is encrypted at rest using AWS-managed KMS keys.

Multi-Factor Authentication (MFA): I will implement MFA in Cognito for additional security during user authentication.

Private Networking: I plan to use VPC endpoints for Lambda and DynamoDB to ensure data does not leave the AWS private network. See Example VPC Infrastructure below:
![Image Alt](https://github.com/coderBrie/aws-unicorn-lyft-app/blob/cb69d6a461f18a68a0d1cd9cdbe47832a05c26ff/Web-App-Architecture-Diagram.jpg)


## Cost
All services used are eligible for the [AWS Free Tier](https://aws.amazon.com/free/).  Outside of the Free Tier, there may be small charges associated with building the app (less than $1 USD), but charges will continue to incur if you leave the app running. 

## Cost Optimization
Free Tier Utilization: Wherever possible, I have utilized AWS Free Tier services during development and testing phases to minimize costs.
Scaling Strategies: The app dynamically scales based on demand, aws pay as you go pricing avoids unnecessary over-provisioning of resources.


Future Cost Optimization:
CloudWatch Monitoring: Use Amazon CloudWatch to monitor resource utilization and set alerts, ensuring resources are scaled only when necessary.

Query Optimization: DynamoDB queries are optimized to minimize read/write operations, helping to reduce operational costs

## The Application Code
The application code is here in this repository.

## The Lambda Function Code
Here is the code for the Lambda function, originally taken from the [AWS workshop](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-3/ ), and updated for Node 20.x:

```node
import { randomBytes } from 'crypto';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const ddb = DynamoDBDocumentClient.from(client);

const fleet = [
    { Name: 'Angel', Color: 'White', Gender: 'Female' },
    { Name: 'Gil', Color: 'White', Gender: 'Male' },
    { Name: 'Rocinante', Color: 'Yellow', Gender: 'Female' },
];

export const handler = async (event, context) => {
    if (!event.requestContext.authorizer) {
        return errorResponse('Authorization not configured', context.awsRequestId);
    }

    const rideId = toUrlString(randomBytes(16));
    console.log('Received event (', rideId, '): ', event);

    const username = event.requestContext.authorizer.claims['cognito:username'];
    const requestBody = JSON.parse(event.body);
    const pickupLocation = requestBody.PickupLocation;

    const unicorn = findUnicorn(pickupLocation);

    try {
        await recordRide(rideId, username, unicorn);
        return {
            statusCode: 201,
            body: JSON.stringify({
                RideId: rideId,
                Unicorn: unicorn,
                Eta: '30 seconds',
                Rider: username,
            }),
            headers: {
                'Access-Control-Allow-Origin': '*',
            },
        };
    } catch (err) {
        console.error(err);
        return errorResponse(err.message, context.awsRequestId);
    }
};

function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

async function recordRide(rideId, username, unicorn) {
    const params = {
        TableName: 'Rides',
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    };
    await ddb.send(new PutCommand(params));
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId) {
    return {
        statusCode: 500,
        body: JSON.stringify({
            Error: errorMessage,
            Reference: awsRequestId,
        }),
        headers: {
            'Access-Control-Allow-Origin': '*',
        },
    };
}
```

## The Lambda Function Test Function
Here is the code used to test the Lambda function:

```json
{
    "path": "/ride",
    "httpMethod": "POST",
    "headers": {
        "Accept": "*/*",
        "Authorization": "eyJraWQiOiJLTzRVMWZs",
        "content-type": "application/json; charset=UTF-8"
    },
    "queryStringParameters": null,
    "pathParameters": null,
    "requestContext": {
        "authorizer": {
            "claims": {
                "cognito:username": "the_username"
            }
        }
    },
    "body": "{\"PickupLocation\":{\"Latitude\":47.6174755835663,\"Longitude\":-122.28837066650185}}"
}
```

