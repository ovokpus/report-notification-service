# report-notification-service
Build a Resilient, Asynchronous Notification System with NodeJS, Cloud Run and Pub/Sub

## Business Use Cases

* **Reliable Medical Test Delivery:** Medical labs can POST test results directly to Pet Theory’s ingestion endpoint, ensuring no results are lost even if downstream systems are unavailable ([cloud.google.com][1]).
* **Asynchronous Notification Pipeline:** Decouple receipt of lab data from client notifications—enabling parallel processing, backoff/retries, and independent scaling of Email and SMS services ([cloud.google.com][2]).
* **Scalable Serverless Architecture:** Handle unpredictable load (e.g. lab report spikes) without provisioning servers or capacity planning; pay only for actual usage ([datadoghq.com][3]).
* **Extensible Event-Driven Platform:** Easily add new consumers (e.g. Slack alerts, data analytics) by subscribing to the same Pub/Sub topic—without touching the ingestion code ([cloud.google.com][4]).

## Architecture Overview

At a high level, the system comprises three single-purpose services:

1. **Lab Report Service** – HTTP POST receiver that publishes each report to a Pub/Sub topic.
2. **Email Service** – Cloud Run worker that decodes Pub/Sub push messages and sends emails.
3. **SMS Service** – Cloud Run worker that decodes Pub/Sub push messages and sends SMS.

By using Pub/Sub as the “event bus,” each component scales independently, enjoys built-in retry semantics, and isolates failures—key advantages of asynchronous microservices ([sysaid.com][5]) ([cloud.google.com][6]).

### Key Concepts

* **Pub/Sub Topics & Subscriptions:** A topic is a named conduit to which producers send messages; subscriptions allow multiple consumers to pull or receive via push ([cloud.google.com][1]).
* **Cloud Run:** A fully managed, serverless container platform that scales to zero and back automatically, perfect for event-driven workers ([cloud.google.com][7]) ([cloud.google.com][8]).
* **Push vs Pull Delivery:** Email and SMS services use **push** subscriptions (Pub/Sub invokes an HTTPS endpoint), while Lab Report Service **pulls** nothing—it only publishes ([cloud.google.com][4]).
* **Fault Isolation & Retries:** If a consumer returns a 500 error, Pub/Sub retries delivery (up to a configurable limit), ensuring no data loss without custom retry logic ([cloud.google.com][9]).

## Prerequisites

* A Google Cloud project with billing enabled.
* **APIs enabled:** Pub/Sub, Cloud Run, Cloud Build, IAM.
* **gcloud CLI** installed and authenticated.
* **Node.js v18** runtime installed locally (for Cloud Shell).

## 1. Create the Pub/Sub Topic

```bash
gcloud pubsub topics create new-lab-report
```

This topic serves as the event bus for all incoming lab reports ([cloud.google.com][1]).

## 2. Build & Deploy the Lab Report Service

### 2.1 Scaffold & Dependencies

```bash
git clone https://github.com/rosera/pet-theory.git
cd pet-theory/lab05/lab-service
npm install express body-parser @google-cloud/pubsub
```

### 2.2 `package.json`

Add in `"scripts"`:

```jsonc
"scripts": {
  "start": "node index.js",
  …
},
```

So Cloud Run knows to invoke `index.js`.

### 2.3 `index.js`

```js
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
app.use(require('body-parser').json());
const port = process.env.PORT || 8080;

app.post('/', async (req, res) => {
  try {
    await pubsub.topic('new-lab-report')
      .publish(Buffer.from(JSON.stringify(req.body)));
    res.status(204).send();
  } catch (e) {
    console.error(e);
    res.status(500).send(e);
  }
});

app.listen(port, () => console.log(`Listening on port ${port}`));
```

### 2.4 Containerize & Deploy

**Dockerfile**

```dockerfile
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --only=production
COPY . .
CMD ["npm","start"]
```

**deploy.sh**

```bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service
gcloud run deploy lab-report-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
  --platform managed --region us-east1 \
  --allow-unauthenticated --max-instances=1
```

```bash
chmod +x deploy.sh && ./deploy.sh
```

Cloud Run will provide a URL for testing. ([cloud.google.com][10])

### 2.5 Smoke Test

```bash
export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service \
  --platform managed --region us-east1 \
  --format="value(status.address.url)")
./post-reports.sh
```

You should see three `204` entries in the Lab Report Service logs.

## 3. Build & Deploy the Email Service

### 3.1 Scaffold & Dependencies

```bash
cd ~/pet-theory/lab05/email-service
npm install express body-parser
```

### 3.2 `package.json`

```jsonc
"scripts": { "start": "node index.js", … },
```

### 3.3 `index.js`

```js
const express = require('express'), bodyParser = require('body-parser');
const app = express();
app.use(bodyParser.json());
app.post('/', async (req, res) => {
  const report = JSON.parse(Buffer.from(req.body.message.data,'base64'));
  try {
    console.log(`Email Service: Report ${report.id} trying…`);
    sendEmail(); // stub
    console.log(`Email Service: Report ${report.id} success`);
    res.status(204).send();
  } catch {
    console.error(`Email Service failure`);
    res.status(500).send();
  }
});
function sendEmail(){ console.log('Sending email'); }
app.listen(process.env.PORT||8080);
```

### 3.4 Containerize & Deploy

Use a similar **Dockerfile** and **deploy.sh** (tagged `email-service`). Then:

````bash
chmod +x deploy.sh && ./deploy.sh
``` :contentReference[oaicite:13]{index=13}

### 3.5 Pub/Sub Push Subscription  
1. **Service Account**  
   ```bash
   gcloud iam service-accounts create pubsub-cloud-run-invoker \
     --display-name="PubSub Invoker"
````

2. **Grant Invoker Role**

   ```bash
   gcloud run services add-iam-policy-binding email-service \
     --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
     --role=roles/run.invoker --region us-east1 --platform managed
   ```
3. **Create Subscription**

   ````bash
   EMAIL_SERVICE_URL=$(gcloud run services describe email-service \
     --platform managed --region us-east1 --format="value(status.address.url)")
   gcloud pubsub subscriptions create email-service-sub \
     --topic=new-lab-report \
     --push-endpoint=$EMAIL_SERVICE_URL \
     --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
   ``` :contentReference[oaicite:14]{index=14}
   ````

### 3.6 Validation

Re-run `post-reports.sh` and observe `email-service` logs for successful deliveries.

## 4. Build & Deploy the SMS Service

Repeat the Email Service steps in `lab05/sms-service`, swapping `sendEmail()` for:

```js
function sendSms(){ console.log('Sending SMS'); }
```

Tag the container `sms-service`, grant the same invoker SA, and create `sms-service-sub`. Validate via logs. ([digitalocean.com][11])

## 5. Resiliency Testing

1. **Simulate Failure:** Edit `email-service/index.js`’s `sendEmail()` to `throw 'Email server is down'`.
2. **Redeploy & Test:**

   ```bash
   ./deploy.sh
   ~/pet-theory/lab05/lab-service/post-reports.sh
   ```

   You’ll see 500 errors in Email Service logs and no message loss—Pub/Sub will retry ([atlassian.com][12]).
3. **Restore & Redeploy:** Remove the `throw`, redeploy, and confirm the backlogged messages are processed with `204` responses.

### Takeaways

* **Loose Coupling:** Services communicate via events, not direct HTTP calls, enhancing flexibility and scalability.
* **Built-In Retry Logic:** Pub/Sub handles redelivery on failure—no custom code needed.
* **Extensibility:** New subscribers (e.g. analytics, logging) can join by creating additional push subscriptions.
* **Serverless Resilience:** Cloud Run’s autoscaling and Pub/Sub’s durable messaging ensure high availability even under failure conditions ([medium.com][13]).

---


[1]: https://cloud.google.com/pubsub/docs/overview?utm_source=chatgpt.com "What is Pub/Sub? - Google Cloud"
[2]: https://cloud.google.com/pubsub/docs/pubsub_overview?utm_source=chatgpt.com "What is Pub/Sub and Pub/Sub Lite? - Google Cloud"
[3]: https://www.datadoghq.com/knowledge-center/serverless-architecture/?utm_source=chatgpt.com "Serverless Architecture: What It Is & How It Works | Datadog"
[4]: https://cloud.google.com/pubsub?utm_source=chatgpt.com "Pub/Sub for Application & Data Integration | Google Cloud"
[5]: https://www.sysaid.com/blog/sysaid-tech/microservices-architecture-asynchronouscommunication-better?utm_source=chatgpt.com "Microservices Architecture: Asynchronous Communication is Better"
[6]: https://cloud.google.com/architecture/scalable-and-resilient-apps?utm_source=chatgpt.com "Patterns for scalable and resilient apps | Cloud Architecture Center"
[7]: https://cloud.google.com/run?utm_source=chatgpt.com "Cloud Run | Google Cloud"
[8]: https://cloud.google.com/serverless?utm_source=chatgpt.com "Serverless | Google Cloud"
[9]: https://cloud.google.com/pubsub/docs/choosing-pubsub-or-cloud-tasks?utm_source=chatgpt.com "Choosing Pub/Sub or Cloud Tasks - Google Cloud"
[10]: https://cloud.google.com/blog/topics/developers-practitioners/cloud-run-story-serverless-containers?utm_source=chatgpt.com "Cloud Run: What no one tells you about Serverless (and how it's done)"
[11]: https://www.digitalocean.com/blog/top-use-cases-for-serverless-computing?utm_source=chatgpt.com "Top Use Cases for Serverless | DigitalOcean"
[12]: https://www.atlassian.com/microservices/cloud-computing/advantages-of-microservices?utm_source=chatgpt.com "5 Advantages of Microservices [+ Disadvantages] - Atlassian"
[13]: https://medium.com/google-cloud/architecting-for-resilience-on-google-cloud-platform-a4a361b764ae?utm_source=chatgpt.com "Architecting for Resilience on Google Cloud Platform - Medium"

