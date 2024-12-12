# AWS Project - Build a Full End-to-End Web Application with 7 Services | Step-by-Step Tutorial

This repo contains the code files used in this [YouTube video](https://youtu.be/K6v6t5z6AsU).

## TL;DR
We're creating a web application for a unicorn ride-sharing service called Wild Rydes (from the original [Amazon workshop](https://aws.amazon.com/serverless-workshops)).  The app uses IAM, Amplify, Cognito, Lambda, API Gateway and DynamoDB, with code stored in GitHub and incorporated into a CI/CD pipeline with Amplify.

The app will let you create an account and log in, then request a ride by clicking on a map (powered by ArcGIS).  The code can also be extended to build out more functionality.

## Cost
All services used are eligible for the [AWS Free Tier](https://aws.amazon.com/free/).  Outside of the Free Tier, there may be small charges associated with building the app (less than $1 USD), but charges will continue to incur if you leave the app running.  Please see the end of the YouTube video for instructions on how to delete all resources used in the video.

## The Application Code
The application code is here in this repository.

## The Lambda Function Code
Here is the code for the Lambda function, originally taken from the [AWS workshop](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-3/ ), and updated for Node 20.x:

```node
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { randomBytes } from 'crypto';

// Initialize DynamoDB client
const client = new DynamoDBClient({});
const ddb = DynamoDBDocumentClient.from(client);

// Unicorn fleet details
const fleet = [
    { Name: 'Angel', Color: 'White', Gender: 'Female' },
    { Name: 'Gil', Color: 'White', Gender: 'Male' },
    { Name: 'Rocinante', Color: 'Yellow', Gender: 'Female' },
];

// Lambda function handler
export const handler = async (event, context) => {
    try {
        // Validate authorization context
        if (!event.requestContext || !event.requestContext.authorizer) {
            return errorResponse('Authorization not configured', context.awsRequestId);
        }

        // Generate a unique Ride ID
        const rideId = toUrlString(randomBytes(16));
        console.log('Received event (', rideId, '): ', JSON.stringify(event));

        // Extract username and request body
        const username = event.requestContext.authorizer.claims['cognito:username'];
        const requestBody = JSON.parse(event.body);
        const pickupLocation = requestBody.PickupLocation;

        // Find a unicorn for the ride
        const unicorn = findUnicorn(pickupLocation);

        // Record the ride in DynamoDB
        await recordRide(rideId, username, unicorn);

        // Return success response
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
        console.error('Error processing the request:', err);
        return errorResponse(err.message, context.awsRequestId);
    }
};

// Helper function to find a random unicorn
function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for pickup location:', pickupLocation);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

// Helper function to record the ride in DynamoDB
async function recordRide(rideId, username, unicorn) {
    const params = {
        TableName: 'Rides', // Ensure this matches your DynamoDB table name
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    };

    console.log('Recording ride:', params);

    try {
        await ddb.send(new PutCommand(params));
        console.log('Ride successfully recorded in DynamoDB');
    } catch (error) {
        console.error('Error recording ride in DynamoDB:', error);
        throw new Error('Could not record ride');
    }
}

// Helper function to generate a URL-safe Base64 string
function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

// Helper function to return an error response
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

