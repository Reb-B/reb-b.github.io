---
title: "Scale up IoT provisioning and deployment with AWS IoT Services - Part 2: Building and Testing Greengrass IoT components"
date: 2024-05-16 00:00:00 +0200
image: /assets/img/posts/iot-greengrass-components/iot-greengrass-components.png
description: |
  Here are some practical insights and guidance to build software with AWS Greengrass Components and the Greengrass Development Kit. With the IoT building blocks offered by the AWS Greengrass service, you can scale up your IoT projects and get them to market faster.
categories: [Platform Engineering]
tags: [aws, iot, greengrass, gdk, platform-engineering]
---

![IoT Things](/assets/img/posts/iot-greengrass-components/iot-greengrass-components.png)

We recently posted a guide to provisioning IoT devices at scale, using AWS Greengrass fleet provisioning. If you missed it, you can take a look at that article [here](/posts/iot-fleet-provisioning/). In this follow-up post, we'll cover how to build software with the help of [AWS IoT Greengrass Components](https://docs.aws.amazon.com/greengrass/v2/developerguide/greengrass-components.html), which is the first step to preparing for large scale IoT deployments with AWS. This post got kind of long, so part 3 focusing on deployment automation is coming up 😅

## AWS Greengrass Components

After provisioning your devices and connecting them to the cloud, you want to ensure that you can deploy features and updates without turning up on the doorstep of your customers - it's 'weird' and 'invasive' apparently 🤷‍♀️

This is where IoT Greengrass components can help you out. There are three types of components:

- [AWS Provided](https://docs.aws.amazon.com/greengrass/v2/developerguide/public-components.html)
- [AWS Community Provided](https://docs.aws.amazon.com/greengrass/v2/developerguide/greengrass-software-catalog.html)
- [Custom Components](https://docs.aws.amazon.com/greengrass/v2/developerguide/develop-greengrass-components.html)

## AWS and Community Provided Components

With AWS IoT Greengrass and community provided components you get a set of pre-built modules that can be used to build IoT applications. They are modular and designed to be deployed to multiple devices at once. The idea is that you can build your own components and use them in conjunction with the pre-built ones. These components tend to cover typical tasks that crop up when working with IoT devices, like connecting to AWS Services, running machine learning models, or managing device data.

An example of an AWS provided component is the [Greengrass Nucleus](https://docs.aws.amazon.com/greengrass/v2/developerguide/greengrass-nucleus-component.html) service. This is the central component of the AWS IoT Greengrass service and is necessary to run Greengrass on your IoT devices. It is essentially a management system for Greengrass components and is responsible for starting and stopping software, and handling configurations for you.

An optional AWS provided component, is the [Greengrass DiskSpooler](https://docs.aws.amazon.com/greengrass/v2/developerguide/disk-spooler-component.html) that allows you to save messages to a device when it is offline, and send them to the cloud again when the device is back online. We've been using this in a recent project to ensure that we don't lose any important data if a device loses its internet connection. This obviously saves us the work of having to build this functionality ourselves.

You can search for AWS Provided components in the console, under `AWS IoT Core` > `AWS IoT Greengrass` > `Components` > `AWS Provided`, and find community provided components on Github. Take a look [here](https://github.com/aws-greengrass/aws-greengrass-software-catalog) if you're interested in getting to know the software catalogue available.

## How do I build my own components?

AWS and community provided components are great for common use cases, but you will need to build your own components if you have specific requirements. To build your own custom components, you need the [Greengrass Development Kit (GDK-CLI)](https://docs.aws.amazon.com/greengrass/v2/developerguide/greengrass-development-kit-cli.html). This allows you to create, build, and publish software to your devices. Building your own custom components means initially developing your software locally. Here's one way you can do this:

In your project directory, you can create a folder for test components, and then create a new component with the `gdk component init` command.

```bash
mkdir test-components && cd test-components

gdk component init \
--language python \
--template HelloWorld \
--name my-component
```

We've used python in this example, so you'll have a `main.py` file in your new component directory. This is where you would put your new code. Add a `requirements.txt` file if you need to install any dependencies. There is also a `gdk-config.json` file that you can use to define the component's configuration. Important here would be to update the version of the component, and the name of the component. You can add `NEXT_PATCH` to the version number to automatically increment the patch version when you build and publish the component.

```json
{
  "component": {
    "com.example.PythonHelloWorld": {
      "author": "<PLACEHOLDER_AUTHOR>",
      "version": "NEXT_PATCH",
      "build": {
        "build_system": "zip",
        "options": {
          "zip_name": ""
        }
      },
      "publish": {
        "bucket": "<PLACEHOLDER_BUCKET>",
        "region": "<PLACEHOLDER_REGION>"
      }
    }
  },
  "gdk_version": "1.3.0"
}
```

In addition to the `gdk-config.json`, you will see a `recipe.yaml` file. This is the [recipe for your application](https://docs.aws.amazon.com/greengrass/v2/developerguide/component-recipe-reference.html) (see below). The recipe specifies the component's configuration parameters, dependencies and platform compatibility. There is also a `lifecycle` that defines what the component should do once it is built and deployed to your device, this might include an installation phase or what it should run. In this case, it runs the code in our `main.py` file with the configuration parameter `Message` and sends a hello world message via MQTT to IoT Core.

```yaml
RecipeFormatVersion: "2020-01-25"
ComponentName: "{COMPONENT_NAME}"
ComponentVersion: "{COMPONENT_VERSION}"
ComponentDescription: "This is simple Hello World component written in Python."
ComponentPublisher: "{COMPONENT_AUTHOR}"
ComponentConfiguration:
  DefaultConfiguration:
    Message: "World"
Manifests:
  - Platform:
      os: all
    Artifacts:
      - URI: "s3://BUCKET_NAME/COMPONENT_NAME/COMPONENT_VERSION/com.example.PythonHelloWorld.zip"
        Unarchive: ZIP
    Lifecycle:
      Run: "python3 -u {artifacts:decompressedPath}/com.example.PythonHelloWorld/main.py {configuration:/Message}"
```

Once you have added any extra code you might want to in the `main.py` file, and set a name and version in the deployment template, you can build it with the command:

`gdk component build`

The component artifact is now available to us locally in the `greengrass-build` directory. The artifact is just a zip file that contains the component's code and dependencies and will be uploaded to an S3 bucket with the specified version number when it is published. But we won't publish it just yet, because we need to test it first.

## Test your component locally

Now that you have your component built locally, you can deploy it to a single test device and check if it is behaving as you expect.

Preqrequisites for your device:

- Greengrass Core installed
- Greengrass CLI installed
- Provisioned and connected to the cloud
- Python installed (assuming you are using a Python component)
- Local test-components directory available on the device

You can find out more about how to set up your device [here](https://docs.aws.amazon.com/greengrass/v2/developerguide/getting-started.html)

---

**Note**: We are currently running services on our [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/) and locally in [Docker](https://docs.docker.com/get-docker/), so we have a [docker-compose.yml](https://docs.docker.com/compose/compose-application-model/) that includes mounting the test-component directory. This results in very little overhead or manual steps to test components on our part. However, if you are running your services on a Raspberry Pi or other device without docker, then you will need to copy the local components directory to the device or automate this with a script.

---

Once that is all set up, you can continue with the following steps:

- connect via ssh to your device
- deploy your component to the device using the GreenGrass CLI
- check the status of the deployment

```bash
# connect via ssh to your device
ssh <user>@<device-ip>

# The deploy command will output a deployment Id
 /greengrass/v2/bin/greengrass-cli deployment create \
    --recipeDir /test-components/<COMPONENT-NAME>/greengrass-build/recipes \
    --artifactDir /test-components/<COMPONENT-NAME>/greengrass-build/artifacts \
    --merge "<COMPONENT-NAME>=<COMPONENT-VERSION>"

# Check deployment status
/greengrass/v2/bin/greengrass-cli deployment status -i <DEPLOYMENT-ID>
```

Now you should have a component running on your device 🙂 If you have any problems, you can find logs in the `/greengrass/v2/logs` directory on your device.

## Conclusion

The AWS Greengrass service is a useful tool for building IoT applications. With Greengrass components and the Greengrass Development Kit, you have a lot of options to craft whatever service you want to provide your users and you can do this with the knowledge that scalability has been taken into account from the outset. If you've got a component working and you want to deploy it to multiple devices, come back to check out part 3 of this series of posts! We'll cover how to automate deployment with Github actions and the AWS Greengrass cli 🎉
