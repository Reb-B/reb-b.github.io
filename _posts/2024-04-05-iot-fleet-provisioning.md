---
title: "Scale up IoT provisioning and deployment with AWS IoT Services - Part 1: Fleet provisioning"
date: 2024-04-05 00:00:00 +0200
image: /assets/img/posts/iot-fleet-provisioning/iot-things.png
description: |
  We offer practical insights and guidance to provision software to multiple IoT devices at once. With the IoT services offered by AWS, you can scale up your IoT projects and get them to market faster.
categories: [IoT]
tags: [aws, iot, greengrass, fleet-provisioning, cdk]
---

![IoT Things](/assets/img/posts/iot-fleet-provisioning/iot-things.png)

## The Problem

The availability and price of devices like the Raspberry Pi and Arduino have made it possible for more and more developers and tech enthusiasts to build their own IoT projects. Maybe you've even connected your device to the cloud and are sending data to [AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html). But what if you want to scale up your project and deploy it to multiple devices? How do you provision and deploy software to hundreds or even thousands of devices in a production environment? and how do you ensure little to no technical intervention is required from your users? This is an altogether different beast.

## AWS IoT Services

There are two main services from AWS IoT that aim to solve this problem:

- [IoT Fleet Provisioning](https://docs.aws.amazon.com/iot/latest/developerguide/iot-provision.html)
- [IoT Greengrass Components](https://docs.aws.amazon.com/greengrass/v2/developerguide/greengrass-components.html)

Both of these services are part of [AWS IoT Greengrass](https://docs.aws.amazon.com/greengrass/v2/developerguide/what-is-iot-greengrass.html), which extends AWS IoT Core and allows you to deploy and run your software locally on your IoT devices.

### Fleet Provisioning

To connect securely to AWS IoT Core, one way or another, your device needs to have a device certificate, a private key, and a root CA certificate. There are several ways to get these certificates onto your devices, but fleet provisioning is recommended for large-scale deployments.

There are two variants of Fleet Provisioning:

- provisioning by claim
- provisioning by trusted user (or trusted third party)

Provisioning by claim involves embedding claim certificates in IoT devices, which have already been registered with AWS IoT Core and then exchanging them for unique device certificates.

Provisioning by trusted user involves a trusted user creating a claim certificate and then transferring it to the device. The device then exchanges this claim certificate for its unique certificate and private key from AWS IoT Core. In this case, it is not necessary to first register claim certificates in IoT core, fleet provisioning handles this aspect of the process for you.

As you can see both of these methods require some degree of intervention on the side of the producer. This is because the claim certificates need to be provided at some point, however it can be automated for the most part.

## Provisioning by Trusted User

We recently automated the process of provisioning by trusted user for a client. A large part of the automation was the brainchild of Karina, Valerie and Jan, so just props to them for coming up with a working solution. The approach was influenced by the code found [here](https://github.com/aws-samples/aws-iot-robot). The client had a large number of devices (RaspberryPi) that they wanted to provision and deploy to and did not want to waste time on manual setup tasks. They wanted users to be shipped their devices, switch them on, and be able to rapidly connect to the cloud.

### Architecture

![Fleet provisioning architecture](/assets/img/posts/iot-fleet-provisioning/fleetprovisioning.drawio.png)

In the architecture diagram, you can see how we set this up. The software for our devices runs in [docker](https://www.docker.com/), so we have a docker compose file that defines the services that run on the raspberryPi. The frontend service is a simple web app that allows users to interact with the device. The second service is AWS IoT Greengrass which provides local compute and messaging capabilities and more importantly the provisioning capabilities we need to connect to the cloud.

### How it Works

On startup, users are directed to a registration and login page where they can begin the setup process. This user registration and login process is handled by [AWS Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) (step 1), which is a user authentication service. On registration, users receive an email from AWS Cognito to confirm that they can now log in with their new credentials. Once the user is logged in, they are presented with the option to 'register a device'.

To handle the device registration process, we have an [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) that is connected to two [lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html). The first function checks to see if the user has already registered that device (step 2). It does this by looking up the device's unique identifying number in a [DynamoDB table](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html). If the device is found, the frontend will display a response to the user, telling them that their device is already registered.

If the device is not already registered, the second lambda function is called (step 3). This function does the following:

- Get the endpoints for the IoT credential and data services
- Call the IoT client to create a temporary claim certificate and private key that is valid for 5 minutes
- Create a unique identifier for the device (Thing Name)
- Save the unique identifier to a DynamoDB table
- Set the setup status of the the device to 'pending' until it is successfully provisioned.
- Return the data to the frontend

```typescript
return {
  statusCode: 201,
  body: JSON.stringify({
    certificatePem: claimResponse.certificatePem,
    certificateKey: claimResponse.keyPair!.PrivateKey,
    dataEndpoint: dataEndpointResponse.endpointAddress,
    credentialEndpoint: credentialProviderResponse.endpointAddress,
    thingName: thingName,
  }),
};
```

Once we have the data from the lambda function returned to our frontend, we use it to adapt the Greengrass configuration file, and to save our temporary certificate data to a folder on our IoT device (step 4). The Greengrass configuration file tells Greengrass where to find the credential and data IoT services, the location of our claim certificate and private key, and the unique identifier for the device. This information is required to connect to the cloud. Here is an example of the configuration file:

```typescript
const generateConfig = (props: {
  dataEndpoint: string;
  credentialEndpoint: string;
  thingName: string;
  ownerId: string;
}) => {
  return `
---
services:
  aws.greengrass.Nucleus:
    version: "2.12.1"
    configuration:
      awsRegion: "eu-central-1"
      iotRoleAlias: "GreengrassCoreTokenExchangeRoleAlias"
      iotDataEndpoint: "${props.dataEndpoint}"
      iotCredEndpoint: "${props.credentialEndpoint}"
      interpolateComponentConfiguration: true
      logging:
        level: "DEBUG"
  aws.greengrass.FleetProvisioningByClaim:
    configuration:
      rootPath: /greengrass/v2
      awsRegion: "eu-central-1"
      iotDataEndpoint: "${props.dataEndpoint}"
      iotCredentialEndpoint: "${props.credentialEndpoint}"
      iotRoleAlias: "GreengrassCoreTokenExchangeRoleAlias"
      provisioningTemplate: "GreengrassFleetProvisioningTemplate"
      claimCertificatePath: "/greengrass-setup/certificate.pem"
      claimCertificatePrivateKeyPath: "/greengrass-setup/certificate.key"
      rootCaPath: "/greengrass-setup/AmazonRootCA1.pem"
      templateParameters:
        ThingName: "${props.thingName}"
        OwnerId: "${props.ownerId}"
        ThingGroupName: "GroupName"`;
};
```

Before we provision our device we need to first confirm that the registered device and its parameters are available in our database. We do this by running a so-called [pre-provisioning hook](https://docs.aws.amazon.com/iot/latest/developerguide/pre-provisioning-hook.html) (step 5). This is pretty much like it sounds; just a script that runs before the device is provisioned. It checks the status of the device in the DynamoDB table and if the status is 'pending' and the parameters are there, it will proceed with the provisioning process. The pre-provisioning hook is added to a fleet provisioning template. Here's how that looks when defined in CDK:

```typescript
const preProvisioningHookFunc = new NodejsFunction(
  this,
  "preProvisioningHook",
  {
    environment: {
      TABLE_NAME: props.dynamoTable.tableName,
    },
  }
);
props.dynamoTable.grantReadData(preProvisioningHookFunc);
preProvisioningHookFunc.grantInvoke(new ServicePrincipal("iot.amazonaws.com"));

// Create fleet provisioning template for greengrass core devices
const greengrassFleetProvisioningTemplate = new iot.CfnProvisioningTemplate(
  this,
  "GreengrassFleetProvisioningTemplate ",
  {
    provisioningRoleArn: greengrassFleetProvisioningRole.roleArn,
    templateBody: JSON.stringify(FleetProvisioningTemplate),
    description: "Provisioning template for fleet",
    enabled: true,
    templateName: "GreengrassFleetProvisioningTemplate",
    preProvisioningHook: {
      targetArn: preProvisioningHookFunc.functionArn,
    },
  }
);
```

If the pre-provisioning hook finds the information that we need in our database, the provisioning process begins. This is carried out in accordance with the parameters we have defined in a fleet provisioning Template. A fleet provisioning template includes many of the same parameters as the Greengrass configuration file, but it also includes additional parameters such as the IoT permissions required for the Greengrass device to perform subscribe and publish actions on various topics you allow. Here is an example of a fleet provisioning template:

```typescript
export const FleetProvisioningTemplate = {
  Parameters: {
    ThingName: {
      Type: "String",
    },
    ThingGroupName: {
      Type: "String",
    },
    "AWS::IoT::Certificate::Id": {
      Type: "String",
    },
    OwnerId: {
      Type: "String",
    },
  },
  Resources: {
    MyThing: {
      OverrideSettings: {
        AttributePayload: "REPLACE",
        ThingGroups: "REPLACE",
        ThingTypeName: "REPLACE",
      },
      Properties: {
        AttributePayload: {
          owner: {
            Ref: "OwnerId",
          },
        },
        ThingGroups: [
          {
            Ref: "ThingGroupName",
          },
        ],
        ThingName: {
          Ref: "ThingName",
        },
      },
      Type: "AWS::IoT::Thing",
    },
    ThingPolicy: {
      Properties: {
        PolicyName: "GreengrassV2IoTThingPolicy",
      },
      Type: "AWS::IoT::Policy",
    },
    ThingCertificate: {
      Properties: {
        CertificateId: {
          Ref: "AWS::IoT::Certificate::Id",
        },
        Status: "Active",
      },
      Type: "AWS::IoT::Certificate",
    },
  },
};
```

Once the device has been provisioned, we will see a IoT Thing created in the IoT Core console. The device is now ready to send and receive messages from the cloud 🎉

### Conclusion

We found that the IoT Core and Greengrass documentation, although extensive, was a little difficult to navigate. There were a myriad of options that may or may not have been relevant to our use case. I hope this post offers some clarification, as to how to set up a process with AWS services, that allows users to register their IoT devices, and connect them to the cloud with minimal intervention.
