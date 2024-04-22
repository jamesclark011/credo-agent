# Vera ID Credo Setup

This repo contains a sample setup for a Credo cloud agent that can interact with Paradym.

## Setup

To keep the setup simple, we will use the published Credo Rest API. This API is a simple REST API that can be used to interact with a Credo cloud agent.

Start by copying the `.env.local.example` to `.env.local`.

```sh
cp .env.local.example .env.local
```

Next, run an ngrok tunnel to expose the local server to the internet. [Follow the instructions](https://ngrok.com/download) if you haven't installed ngrok yet.

```sh
ngrok http 3001
```

Next, set the `CREDO_REST_ENDPOINT` in `.env.local` to the **https** URL of your ngrok tunnel.

## Running

Once you have set up the `.env.local` file, you can start the server by the following command. If you don't have docker installed, you can install it [here](https://docs.docker.com/engine/install/).

```sh
docker compose up -d
```

You can now go to http://localhost:3000 and see the docs for the API.

If you want to view logs from the agent, you can run the following command:

```sh
docker compose logs -f
```

## Webhook events

If you'd like to see events emitted by the Credo cloud agent, you can set up a webhook URL that will receive events.

For testing you can use a service like https://webhook.site, which will give you a unique URL that you can use to receive events.

Update the `CREDO_REST_WEBHOOK_URL` in the `.env.local` file to the URL provided by webhook.site, and start the server again. You should now see events being sent to the webhook URL when you interact with the Credo cloud agent.

## Guides

This sections contains several guides on how to interact with the Credo cloud agent.

You can use the Swagger UI at http://localhost:3000, use a tool like Postman, or integrate with the API using code.

## Creating a tenant

1. Make a POST request to `/tenants`, note the `id` of the tenant, this should be provided in all subsequent requests for this tenant using the `x-tenant-id` header (you can easily configure this in the Swagger UI).

## Create a connection and send a basic with Paradym

1. Save the below "Connect and message" workflow in Paradym, click "Publish" and then "Run. Fill in a message and click "Execute".

```yaml
name: Connect and message

trigger:
  type: api

input:
  type: object
  properties:
    message:
      type: string

  required:
    - message

actions:
  - id: createConnection
    name: didcomm/createConnection@v1

  - id: sendBasicMessage
    name: didcomm/sendBasicMessage@v1
    attributes:
      connectionId: $.actions.createConnection.output.connection.connectionId
      message: $.input.message
```

2. Copy the `invitationUrl` from the output of the `createConnection` action.
3. Make a POST request to `/didcomm/out-of-band/receive-invitation` with the following payload (make sure to replace the invitation url):

```json
{
  "invitation": "https://paradym.id/invitation/c2568592-345f-4e0c-80dc-f82b2a3cd984"
}
```

4. If you now look at the logs (using `docker compose logs -f`), or you have configured a webhook url at webhook.site (see how to above), you should see the connection being created and the basic message being received. That's all you needed to create a connection and send a message with Paradym and a custom Credo cloud agent.
5. We can also send a reply from the Credo cloud agent to Paradym. First, set up the following workflow in Paradym, to react to incoming messages. Make sure to publish it.

```yaml
name: Reply to basic message

trigger:
  type: didcommBasicMessageReceived

input:
  type: object

actions:
  - id: sendBasicMessage
    name: didcomm/sendBasicMessage@v1
    attributes:
      connectionId: $.input.basicMessage.connectionId
      message: I'll reply whenever you send me something! :)
```

6. Now, look at the logs, or the webhook.site events at the id of the connection that was just created.
7. Make a POST to `/didcomm/basic-messages/send` with the following payload (make sure to replace the connection id):

```json
{
  "connectionId": "c2568592-345f-4e0c-80dc-f82b2a3cd984",
  "content": "Hello, Paradym!"
}
```

8. If you now look in the Paradym dashboard at the executions of the "Reply to basic message" workflow, you should see that the message was received and a reply was sent.

## Receiving a credential from Paradym

To issue a credential from Paradym, we first need to set up a credential template, and then we can issue with it. To keep it simple, we'll combine all steps in one workflow.

1. Create a new workflow, open the "Templates" tab, and select "Claim Paradym demo credential". This includes everything needed to start issuing a credential. Save the workflow, click "Publish" and then "Run". Fill in the fields, and then click "Execute".
2. Copy the `invitationUrl` from the output of the `createConnection` action, and make a POST request to `/didcomm/out-of-band/receive-invitation` with the following payload (make sure to replace the invitation url):

```json
{
  "invitation": "https://paradym.id/invitation/c2568592-345f-4e0c-80dc-f82b2a3cd984"
}
```

3. After receiving the invitation, the agents will make a connection and Paradym will automatically send a credential offer afterwards. You can look at the logs, or the webhook.site events to see the credential offer being received. For now we'll make a GET request to `/didcomm/credentials`, which should now return exactly one result that has a state of `offer-received`. Note down the `id` of the record.
4. Now we're going to accept the credential offer. Make a POST request to `/didcomm/credentials/{credentialExchangeId}/accept-offer` with the `id` of the credential exchange record. The request can be an empty JSON object.
5. You should now have the credential in your wallet! You can make a GET request to `/didcomm/credentials/{credentialExchangeId}/format-data` to see the full credential exchange format metadata, or you can make a GET request to `/didcomm/credentials/{credentialExchangeId}` to see the credential exchange record, which should now have state `done`. The workflow in Paradym should also have reached the "Completed" state.

## Presenting a credential to Paradym

> NOTE: the Credo cloud agent currently has no endpoint to fetch all credentials matching a presentation request, so you can choose which credential to share in a proof request. This will be added soon and allows for more control.

These steps will guide you through presenting a credential to Paradym from the Credo cloud Agent, it expects the above credential issuance flow to have been completed. In this case we're going to do a connection-less verification (i.e. we're not going to create a connection first), and we're going to embed the verification request directly into the out of band invitation.

1. Create a new workflow based on the following template. The `didcomm/createPresentationRequestInvitation` will create an invitation that includes a presentation request. Save the workflow, click "Publish" and then "Run".

```yaml

```

2. Copy the `invitationUrl` from the output of the `createConnection` action, and make a POST request to `/didcomm/out-of-band/receive-invitation` with the following payload (make sure to replace the invitation url):

```json
{
  "invitation": "https://paradym.id/invitation/c2568592-345f-4e0c-80dc-f82b2a3cd984"
}
```

3. After receiving the invitation, you should be able to see the received presentation request. You can look at the logs, or the webhook.site events to see the presentation request being received. For now we'll make a GET request to `/didcomm/proofs`, which should now return exactly one result that has a state of `request-received`. Note down the `id` of the record.
4. Now we're going to accept the proof request. Make a POST request to `/didcomm/proofs/{proofExchangeId}/accept-request` with the `id` of the proof exchange record. The request can be an empty JSON object.
5. You should now have shared the presentation with the Paradym agent! You can make a GET request to `/didcomm/proofs/{proofExchangeId}/format-data` to see the full proof exchange format metadata, or you can make a GET request to `/didcomm/proofs/{proofExchangeId}` to see the proof exchange record, which should now have state `done`. Finally, the workflow within Paradym should have reached the "Completed" state, and the `createPresentationRequestInvitation` action should now show the presentation with the requested data (`Title`) in the output.
