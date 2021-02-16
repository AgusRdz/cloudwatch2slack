# Creating a CloudWatch Log custom integration for Slack using AWS Lambda

Why didn't the 🍊 win the race? It ran out of juice.

There exists Slack integrations for everything but sometimes you need something more specific so this time I'm going to show you a way to create your own customizable integration to be notified when your application logs something in CloudWatch.

### A custom integration, what is it?

> Incoming webhooks are a simple way to share information from external sources with your workspace. [Slack Help Center]([Incoming webhooks for Slack](https://slack.com/intl/en-mx/help/articles/115005265063-Incoming-webhooks-for-Slack))

The API to create an integration it is very complete, of course it depends on the things you want to create but it covering enough to solve all needs.
In other hand, is a way to have total control under your information and the format used to display it in your Slack workspace.

### What is a Lambda function?

Basically it is a code snippet that only runs on demand and the price is cheaper than other Slack integrations that you have to pay even if they are not used. Using this function you can reduce your costs a bit by eliminating unnecessary integrations since if you only need to send error alerts, the use of this function should be very little, because of course we do not want our applications in production to be full of errors.

### Let's go to log

1. First you need to access to [AWS Management Console](https://console.aws.amazon.com/)
2. Create a new [IAM role](https://console.aws.amazon.com/iam/home#/roles), it will be used to access to CloudWatchLogs events.

   ![1](/images/1.png)
   ![2](/images/2.png)
   ![3](/images/3.png)

3. Now you need to create 2 logs groups in [CloudWatch](https://us-east-2.console.aws.amazon.com/cloudwatch/home), `/aws/lambda/Log2Slack` and `production-api` or whatever name you want to test. Note: _Retetion setting is used to assign a log lifetime_.
   ![4](/images/4.png)
   ![5](/images/5.png)
4. In order to create the lambda function you need to go to [Lambda](https://us-east-2.console.aws.amazon.com/lambda/home) and click on Create button then assign a name, select the runtime language (Node.js for this example) and assign the role previously created.
   ![6](/images/6.png)
   ![7](/images/7.png)
5. Add a trigger to listen the CloudWatch events.
   ![8](/images/8.png)
   ![9](/images/9.png)
6. Before to add the lambda code you need to create a custom integration using this url `https://{your-slack-workspace}.slack.com/apps/manage/custom-integrations` then clicking on `Incoming Webhooks`
7. Now click on `Add to Slack`.
   ![10](/images/10.png)

- Select a channel to post the alerts
- Click on `Add Incoming Webhook integration`
- Copy the value of `Webhook URL`
  ![11](/images/11.png)
  Note: _In **Integration Settings** you can assign a name and a logo to provide a nicer look _

8. It has been a long road so far, but now you are very close to completing the integration, click on the name of your function to open the code editor and then double click on the `index.js` file and replace the current content with the code below.
   ![12](/images/12.png)

````js
const zlib = require("zlib");
const https = require("https");
const SLACK_ENDPOINT =
  "/services/T1N6FE97Y/B01NK2BR2CR/TrCYV2mkCIRaaxopcXYF3jyc"; // don't use this endpoint, I removed it after publish this post
const SLACK_BOT = "Cloudwatch";

function doRequest(content) {
  // formatting the message according Slack API
  const payload = {
    username: SLACK_BOT,
    blocks: [
      {
        type: "header",
        text: {
          type: "plain_text",
          text: "Whoops, looks like something went wrong 😞🤕",
          emoji: true,
        },
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: "<!here> the API is running into an issue",
          },
        ],
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: "*Environment: * Production",
          },
        ],
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: "*Message:* _" + content.message + "_",
          },
        ],
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: "*Stacktrace:*",
          },
        ],
      },
      {
        type: "section",
        text: {
          type: "mrkdwn",
          text:
            "```" +
            JSON.stringify(content.original ? content.original : content) +
            "```",
        },
      },
      {
        type: "divider",
      },
    ],
  };

  const payloadStr = JSON.stringify(payload);
  const options = {
    hostname: "hooks.slack.com",
    port: 443,
    path: SLACK_ENDPOINT,
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Content-Length": Buffer.byteLength(payloadStr),
    },
  };

  const postReq = https.request(options, function (res) {
    const chunks = [];
    res.setEncoding("utf8");
    res.on("data", function (chunk) {
      return chunks.push(chunk);
    });
    res.on("end", function () {
      if (res.statusCode < 400) {
        console.log("sent!!!");
      } else if (res.statusCode < 500) {
        console.error(
          "Error posting message to Slack API: " +
            res.statusCode +
            " - " +
            res.statusMessage
        );
      } else {
        console.error(
          "Server error when processing message: " +
            res.statusCode +
            " - " +
            res.statusMessage
        );
      }
    });
    return res;
  });
  postReq.write(payloadStr);
  postReq.end();
}

function main(event, context) {
  context.callbackWaitsForEmptyEventLoop = true;
  // always returns the last event
  const payload = Buffer.from(event.awslogs.data, "base64");
  const log = JSON.parse(zlib.gunzipSync(payload).toString("utf8"));
  // the log is an object that contains an array of events called `logEvents` and we need access it bypassing the index 0
  doRequest(log.logEvents[0]);
  const response = {
    statusCode: 200,
    body: JSON.stringify("Event sent to Slack!"),
  };
  return response;
}

exports.handler = main;
````

9. Click on `Deploy` button to complete the function.
10. Testing, testing and testing. There are two ways to test the function:

- Clicking on `Test` button and selecting the `Amazon Cloudwatch Logs` template.
  ![14](/images/14.png)

- Creating a `Log Stream` directly into the `Log Group` , then enter to log stream and triggering a log event manually.
  ![15](/images/15.png)
  ![16](/images/16.png)
  ![17](/images/17.png)

This is the final result after a Cloudwatch log is sent to Slack 🎉
![18](/images/18.png)

Slack resources:

- [Block Kit](https://app.slack.com/block-kit-builder)
- [Mentions](https://api.slack.com/reference/surfaces/formatting#mentioning-users)
- [Special Mentions](https://api.slack.com/reference/surfaces/formatting#special-mentions)

### Final thoughts

Create your own integration to log the relevant information about your applications is very easy and if you want you can customize each type of log level to show as `success`, `info`, `warning` or `error` providing an easy way to fix the issues without wasting time checking the logs directly in Cloudwatch.

At ClickIt we can provide solutions to reduce your costs or improve your workflow through the use of the most modern and useful tools.
