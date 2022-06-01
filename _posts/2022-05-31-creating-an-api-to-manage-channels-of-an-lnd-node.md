## Creating An API To Manage Channels of an LND Node
In our previous article, we created the basic setup to connect our LND node with a node application hosted in a docker container. In this article, we will create API endpoints to interact with our node and list, close and create channels with other nodes in our lightning simulation network.

### List Channels
We can list channels with the `getChannels` method provided by `ln-service`. All methods provided by `ln-service` take an `lnd` object as a parameter. See my [previous article](https://lndev.net/2022/05/05/connecting-a-node-application-to-a-lightning-lnd-node) on how to set up the connection to the LND node. Now, let's create a simple `GET` endpoint that will return a list of all channels of our LND node:
```javascript
router.get("/get-channels", async (req, res) => {
    try {
        const channels = await lnService.getChannels({ lnd });
        res.json({
            data: channels,
        });
    } catch(e) {
        res.status(500).send();
    }
});
```

When making a `GET` request to `http://localhost:8080/api/get-channels` through [Postman](https://postman.com), we receive a response with an array of objects each of which describes one channel. The [documentation](https://github.com/alexbosworth/ln-service#getchannels) of `ln-service` includes a description for each of the fields of a channel object. We will make use of these fields for closing a channel.
![Postman get-channels](/assets/img/2022-05-31/postman-get-channels.png)

### Closing Channels
To close a channel, we can call the `closeChannel` method and pass either the `id` or the `transaction_id` __and__ the `transaction_vout`. Let's define a `DELETE` endpoint `/close-channel` in our API:
```javascript
router.delete("/close-channel", async (req, res) => {
    const { id, transaction_id, transaction_vout } = req.body;

    if(!(id || (transaction_id && transaction_vout))) {
        res.status(400).send({
            error: ['Missing parameter! "id" OR "transaction_id" AND "trasaction_vout" are required.']
        });
    }

    try {
        const result = await lnService.closeChannel({ lnd, id, transaction_id, transaction_vout });
        res.status(200).json(result);
    } catch(e) {
        res.status(500).send();
    }
});
```

Now, we can grab the `id` of the response from our previous call to `get-channel` and pass it to the body of a `DELETE` request to `http://localhost:8080/api/close-channel`.  
When we update the channels from nodes in Polar, the channel will turn orange and show the status `Waiting to Close`. The reason is that in our simulation network, no further block has been mined to record the transaction. Once we mine a block and update the channels again, the channel will disappear.

![Postman close-channel](/assets/img/2022-05-31/postman-close-channel.png)

### Creating a channel
Let's have a look at creating a channel with another node in our network. The method `openChannel` takes two parameters: 
  - `partner_public_key` – the public key of the channel partner's node
  - `local_tokens` — the channel capacity in satoshis, which is funded by the channel opener  
  
We can add a `POST` endpoint with the same parameters to our API to enable opening channels. 
```javascript
router.post("/open-channel", async (req, res) => {
    const { partner_public_key, local_tokens } = req.body;

    if(!(partner_public_key && local_tokens)) {
        res.status(400).send({
            error: ['Missing paramter! "partner_public_key" and "local_tokens" are required.']
        });
    }

    try {
        const result = await lnService.openChannel({ lnd, partner_public_key, local_tokens });
        res.status(200).json(result);
    } catch(e) {
        res.status(500).send(e);
    }
});
```  
  
Now, we can make a `POST` request to our endpoint `http://localhost:8080/api/open-channel` and pass the public key of the node `bob` and the desired capacity for the channel.
![Postman open-channel](/assets/img/2022-05-31/postman-open-channel.png)

### Wrapping up
We are now able to list, close and open channels through our API. We can confirm that the channel was opened in Polar. Once we have mined a block, the channel will appear and Polar will show a connection between the two nodes. Clicking on the connection between the nodes will open the details of the channel in Polar. Next, we will have a look at sending payments.
![Polar channel with bob](/assets/img/2022-05-31/polar-channel.png)

