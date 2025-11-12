# Building Reliable Embedded Cloud Integrations with MQTT

By FerretCode

## The Problem

In applications that require multiple embedded devices, synchronization, management, and communication between them can be a challenge. In addition, the need for coordination and data aggregation grows when building end-to-end applications that combine both embedded devices and cloud services. Reliability is also a concern in such applications, since dropping data or failing to reach a device is undesirable.

## How MQTT can be a Good Fit

MQTT is a messaging protocol for embedded applications that follows a pub-sub architecture that's designed for IoT applications and low-bandwidth/power applications. Clients (the devices in this case) generally publish messages to a central broker under different "topics," which are then routed to other clients subscribed to that topic. Moreover, MQTT is bidirectional, meaning devices can both listen and subscribe to interested topics.

MQTT is also a lightweight protocol which is simple to implement in code and requires little processing power, meaning it's suitable for resource-constrained environments. It's also scalable and efficient--payloads are often small, and both the protocol and many brokers alike are designed to handle lots of connected devices. Finally, MQTT brokers can be configured with several authentication methods like username/password or client certificate auth.

Furthermore, one of MQTTs draws is reliability. The protocol supports several reliability features that are designed to ensure messages are delivered even over unreliable networks.

The distributed nature of MQTT, its pub-sub architecture, and its focus on resource efficiency and reliability make it a good fit for multi-device and end-to-end embedded applications.

## MQTT's Reliability Features in Detail

One of MQTT's main reliability mechanisms is the use of Quality of Service (QoS) levels and functionality. MQTT exposes three QoS levels which all allow a tradeoff between reliability and overhead:

- QoS Level 0:
    - Guarantees "at most once" transmission
    - Fire & Forget
    - No `ACK/NACK`
    - Delivery is not guaranteed
    - Best in scenarios where readings or data are not critical, or in high-frequency data collection where occasional losses are acceptable
- QoS Level 1:
    - Guarantees "at least once" transmission
    - Guarantees the message arrives, but duplicates may occur
    - Sender resends the message until a `PUBACK` acknowledgement message is received
    - Best used when transmitting important data where delivery is crucial but duplicates are acceptable
- QoS Level 2:
    - Guarantees "exactly once" transmission
    - Best guarantee of reliability, since it ensures a message is always delivered exactly once
    - Uses a four-step handshake to confirm delivery
    - Best used for mission critical data or commands where loss OR duplication is unacceptable

These QoS levels are the first step in building a reliable message delivery flow. Additionally, MQTT also has other reliability mechanisms; one of which is persistent sessions.

When using persistent sessions, a client will connect to the broker, which stores the client's current subscriptions, and queues missed QoS 1 and 2 messages while the client is offline. When that same client reconnects, the broker will deliver the queued messages, ensuring no data is lost during temporary disconnections.

Additionally, the use of the "Keep Alive" functionality helps detect abnormal disconnections. Each client periodically sends a `PINGREQ` packet to the broker to indicate that it's online. If the broker doesn't receive a message or `PINGREQ` within a specified interval, it can close the connection, and then use the final reliability mechanism:

Last Will and Testament (LWT) is a mechanism where clients can specify a "will" message during connection. If the client doesn't gracefully disconnect (e.g., due to a network failure), the broker will automatically publish that predefined message to a specified topic, allowing other clients to be aware of the disconnection.

When using all of these reliability mechanisms that MQTT provides, it allows for high reliability systems that support varying network conditions and device priorities. 

# Example Device <-> Cloud Integration using RP2040, MQTT, EMQX, C, and Go

Several months ago, I built a prototype embedded device based on the RPi Pico W using the C SDK that talked to a cloud backend and displayed information on the frontend using MQTT.

This project used an RFID tag reader for warehouse inventory tracking and management. One of the device's functions was to assist the dashboard in mapping new RFID tags to items. The dashboard would create a new item in its database, and over MQTT send the device basic information about the item. The device would then display the item on an OLED and allow the operator to scan an RFID tag that would be associated with that item upon warehouse ingress or egress.

The first step in building the communication flow between the device and the cloud is to initialize an MQTT client and create the connection to the broker. For this project, since I was using the RPi Pico W, I was able to use the LwIP TCP/IP stack, which on top of basic TCP/IP support implements application layer protocols, including MQTT. The general broker connection flow on the Pico looks like:

1. Initialize WiFi on the Pico's CYW43 wireless chip 
2. Connect to a network 
3. Create the MQTT client by initializing some state and creating the LwIP client object 
4. Run a DNS lookup on the EMQX instance hostname 
5. Attempt a connection with the broker

In practice, the high-level initialization function looked like:

```c
MQTT_CLIENT_T* init_mqtt(char* ssid, char* pw, char* broker_hostname, u16_t broker_port, int timeout)
{
    int wifi_init_result = cyw43_arch_init();
    if (wifi_init_result) {
        printf("Failed to initialize MQTT. Init result: %d\n", wifi_init_result);
        return NULL;
    }

    while (!cyw43_is_initialized(&cyw43_state)) {
        sleep_ms(1);
    }

    cyw43_arch_enable_sta_mode();

    sleep_ms(2000);

    printf("Connecting to WiFi with credentials ssid=%s, pw=%s\n", ssid, pw);
    int connection_result = cyw43_arch_wifi_connect_timeout_ms(
        ssid, pw, CYW43_AUTH_WPA2_AES_PSK, timeout);
    if (connection_result) {
        printf("Failed to connect to WiFi. Connection result: %d\n", connection_result);

        switch (connection_result) {
        case PICO_ERROR_BADAUTH:
            printf("Error authenticating to WiFi network\n");
            break;
        case PICO_ERROR_TIMEOUT:
            printf("Timeout authenticating to WiFi network\n");
            break;
        case PICO_ERROR_CONNECT_FAILED:
            printf("Other error connecting to WiFi network\n");
            break;
        }

        return NULL;
    } else {
        printf("Connected to WiFi\n");
    }

    MQTT_CLIENT_T* state = mqtt_client_init(broker_port);

    int res = run_dns_lookup(state, broker_hostname);
    if (res == DNS_ERR) {
        state->err = DNS_ERR;
        return state;
    }

    state->mqtt_client = mqtt_client_new();
    state->counter = 0;

    if (state->mqtt_client == NULL) {
        printf("Failed to create new MQTT client\n");
        state->err = MQTT_ERR;
        return state;
    }

    res = mqtt_connect(state);

    if (res != ERR_OK) {
        printf("Error connecting to MQTT broker\n");
        state->err = res;
        return state;
    }

    return state;
}
```

### Note: the LwIP MQTT implementation does not support persistent sessions, so this project just relied on the device being connected to the broker when we needed to register new items.

Once a connection is made with the broker, the next step was to subscribe to the "incoming items" topic such that we could listen for new items to be registered:

```c
void subscribe_incoming_items(MQTT_CLIENT_T* state, Task** head)
{
    mqtt_set_inpub_callback(
        state->mqtt_client,
        NULL,
        process_incoming_item_data,
        head);

    if (mqtt_client_is_connected(state->mqtt_client) == 1) {
        mqtt_subscribe(
            state->mqtt_client,
            SUB_TOPIC,
            1,
            subscribed_topic,
            NULL);
    } else {
        printf("MQTT client is not connected.\n");
    }
}

void subscribed_topic(void* arg, err_t err)
{
    printf("Subscription callback triggered\n");
    if (err != ERR_OK) {
        printf("Error subscribing. Err: %d\n", err);
    } else {
        printf("Successfully subscribed.\n");
    }
}
```

With the subscription created, new messages sent by the dashboard would trigger this callback, which would parse a JSON payload and push it onto a linked-list-based item queue for display and processing:

```c
void process_incoming_item_data(void* arg, const u8_t* data, u16_t total_len, u8_t flags)
{
    if (flags & MQTT_DATA_FLAG_LAST) {
        printf("New item to register!\n");
        printf("Received Length: %d\n", total_len);
        printf("MQTT Payload: %.*s\n", total_len, (const char*)data);

        if (!arg) {
            printf("Failed to process item\n");
            return;
        }

        Task** head = (Task**)arg;

        ITEM_T* item = (ITEM_T*)malloc(sizeof(ITEM_T));

        cJSON* json = cJSON_Parse((const char*)data);
        if (json == NULL) {
            const char* err_ptr = cJSON_GetErrorPtr();
            if (err_ptr != NULL) {
                printf("Error parsing JSON payload: %s\n", err_ptr);
            }
            cJSON_Delete(json);
            return;
        }

        cJSON* item_id = cJSON_GetObjectItemCaseSensitive(json, "item_id");
        cJSON* tag_quantity = cJSON_GetObjectItemCaseSensitive(json, "tag_quantity");

        if (!cJSON_IsNumber(item_id) || !cJSON_IsNumber(tag_quantity)) {
            printf("Invalid JSON payload format\n");
            cJSON_Delete(item_id);
            cJSON_Delete(tag_quantity);
            cJSON_Delete(json);
            return;
        }

        cJSON* item_name = cJSON_GetObjectItemCaseSensitive(json, "item_name");

        if (!cJSON_IsString(item_name)) {
            printf("Invalid JSON payload format\n");
            cJSON_Delete(item_id);
            cJSON_Delete(tag_quantity);
            cJSON_Delete(json);
            cJSON_Delete(item_name);
            return;
        }

        item->item_id = cJSON_GetNumberValue(item_id);
        item->tag_quantity = cJSON_GetNumberValue(tag_quantity);
        item->item_name = cJSON_GetStringValue(item_name);

        enqueue_task(head, item);
    }
}
```

Finally, as items were pulled from the queue they would be displayed onto an OLED for the operator to read, then scan the associated tag, and finally press a button that tells the cloud backend what the tag's ID was:

```c
void publish_item_registered(MQTT_CLIENT_T* state, ITEM_T* item)
{
    // encode the item as JSON
    cJSON* root = cJSON_CreateObject();
    cJSON_AddNumberToObject(root, "item_id", item->item_id);
    cJSON_AddNumberToObject(root, "tag_quantity", item->tag_quantity);

    char hexTag[5];

    bytes_to_hex(item->tag, sizeof(item->tag), hexTag);

    cJSON_AddStringToObject(root, "tag", hexTag);

    char* json_str = cJSON_Print(root);
    cJSON_Delete(root);

    mqtt_publish(state->mqtt_client, PUB_TOPIC, json_str, strlen(json_str), 1, 0, NULL, NULL);

    free(item);
    free(json_str);
}
```

On the backend, the flow looked something like this:

1. When requested by an operator, create a new item in the database and publish it to the "item registration" topic
2. Listen for a response from the "item registered" topic and update the associated item in the database.

Here is what the item publishing looked like on the backend:

```go
func CreateItem(w http.ResponseWriter, r *http.Request, ctx types.RequestContext) error {
	var item repositories.CreateItemParams

	err := r.ParseForm()
	if err != nil {
		return err
	}

	item.Name = sql.NullString{r.PostFormValue("name"), true}
	item.Category = sql.NullString{r.PostFormValue("category"), true}
	item.Sku = sql.NullString{r.PostFormValue("sku"), true}

	tx, err := ctx.DB.Begin()
	if err != nil {
		return err
	}
	defer tx.Rollback()

	qtx := ctx.Repositories.WithTx(tx)

	newItem, err := qtx.CreateItem(ctx.Ctx, item)
	if err != nil {
		return err
	}

	if err = tx.Commit(); err != nil {
		return err
	}

	messagePayload := types.ItemRegistrationPaylod{
		ItemId:      newItem.ID,
		ItemName:    newItem.Name.String,
		TagQuantity: 100, // TODO: make this variable
	}

	stringified, err := json.Marshal(messagePayload)
	if err != nil {
		return err
	}

	token := ctx.MQTTConn.Publish("item_registration", 1, false, stringified)
	if token.Wait() && token.Error() != nil {
		return err
	}

	http.Redirect(w, r, "/dashboard/items", http.StatusFound)

	return nil
}
```

And here is what the item updating looked like:

```go
func (h *MQTTHandler) CreateTag(client mqtt.Client, msg mqtt.Message) {
	createTagRequest := types.CreateTagRequest{}

	if err := json.Unmarshal(msg.Payload(), &createTagRequest); err != nil {
		h.Logger.Error("error decoding payload", "err", err)
		return
	}

	h.Logger.Info("new item finished registration", "request", createTagRequest)

	tx, err := h.DB.Begin()
	if err != nil {
		h.Logger.Error("error opening database transaction", "err", err)
		return
	}
	defer tx.Rollback()

	qtx := h.Repositories.WithTx(tx)

	_, err = qtx.CreateTag(*h.Ctx, repositories.CreateTagParams{
		Uid:      sql.NullString{createTagRequest.Uid, true},
		Item:     sql.NullInt64{createTagRequest.ItemId, true},
		Quantity: sql.NullInt64{createTagRequest.Quantity, true},
	})
	if err != nil {
		h.Logger.Error("error creating tag", "err", err)
		return
	}

	if err = tx.Commit(); err != nil {
		h.Logger.Error("error committing database transaction", "err", err)
		return
	}
}
```

# Why MQTT Was Useful in This Project (and why it can be in yours too) 

MQTT was a good choice in this project--its pub-sub architecture and distributed nature made the application scalable up to many devices. After item registration, operators also need to be able to use the scanner to view the inventory of items in the warehouse. With many operators (and therefore devices) that may be working at once, having a protocol that was robust enough not to drop packets in an environment where network connectivity may be limited and could support many communications at once was imperative. 

For this project, I used QoS level 1, and the communication was reliable for what I needed it to be. It was a shame that LwIP couldn't support persistent sessions, but when connection was stable, packets were always delivered reliably.

Additionally, MQTT was a good fit for this project since it allowed the backend to update in real-time with fresh data from multiple sensor boards, and allowed device management remotely via the frontend.

For similar reasons, MQTT may be useful in your project--if you need an application layer protocol that is reliable, distributed, and allows for communication, management, and synchronization, MQTT is a good fit.
