# Day 1 – End-to-End Setup & Development Summary

## Goal for Day 1
Build the integration workflow:
Blob Added in ADLS → Event Grid → Azure Function → Service Bus Queue  
Successfully implemented, tested, and verified the entire flow.

# 1. Created Required Azure Resources

## 1.1 Storage Account (ADLS Gen2)
•	Name: banksourcedata  
•	Role: Source location where folders/files (atm, customers, upi) arrive.  
•	Enabled hierarchical namespace (ADLS Gen2).  
•	Created container: raw  

##  1.2 Function App
•	Name: Bank-Func-app  
•	Runtime: Python  
•	Hosting: Consumption plan  
•	Purpose: To receive Event Grid events and push blob metadata to Service Bus.  

##  1.3 Service Bus Namespace
•	Name: Service-Bus-Bank  
•	Pricing: Basic Tier  
•	Purpose: Queue messages from Function App.  

##  1.4 Service Bus Queue
•	Name: queue_ingestion  
•	Role: Temporary buffer for blob processing events.  


# 2. Configure Event Subscriptions (Event Grid)

##  2.1 Enable Event Grid on Storage Account
•	Navigated to: Storage Account → Events  
•	Created a System Topic automatically:  
	o	Name: Event-Trigger-New-File-Topic  
	o	Topic Type: Microsoft.Storage.StorageAccounts  

##  2.2 Event Subscription
•	Name: Event-Trigger-New-File  
•	Trigger Types:  
	o	BlobCreated  
	o	BlobDeleted  
•	Endpoint Type: Azure Function  
•	Selected function: EgBlobToQueue  

This ensures that whenever a new blob is uploaded, Event Grid immediately triggers our Function App.


# 3. Built the EventGrid-Triggered Azure Function

##  3.1 Code in __init__.py

```python
import json
import logging
import azure.functions as func
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import os

def main(event: func.EventGridEvent):

    logging.info("Event Grid Trigger Fired for New File Upload")

    event_data = event.get_json()
    url = event_data["url"]                         # file URL
    file_name = url.split("/")[-1]                 # extract filename

    logging.info(f"File Uploaded: {file_name}")

    # Basic Validation
    if not file_name.endswith(".csv"):
        logging.error("Invalid file format. Only CSV allowed.")
        return

    # Detect source type based on filename
    source_type = ""
    if "atm" in file_name.lower():
        source_type = "ATM"
    elif "upi" in file_name.lower():
        source_type = "UPI"
    elif "customer" in file_name.lower():
        source_type = "CUSTOMER"
    else:
        source_type = "UNKNOWN"

    # Prepare message body
    message_body = {
        "file_url": url,
        "file_name": file_name,
        "source_type": source_type
    }

    logging.info(f"Sending message to Service Bus: {message_body}")

    # Send to Service Bus Queue
    sb_client = ServiceBusClient.from_connection_string(
        conn_str=os.environ["SERVICE_BUS_CONNECTION_STRING"]
    )

    queue_name = os.environ["SERVICE_BUS_QUEUE_NAME"]

    with sb_client:
        sender = sb_client.get_queue_sender(queue_name=queue_name)
        with sender:
            sb_message = ServiceBusMessage(json.dumps(message_body))
            sender.send_messages(sb_message)

    logging.info("Message sent to Service Bus Queue successfully.")
```

The function performs exactly THREE tasks:
1.	Receive Event Grid event
2.	Extract blob URL
3.	Push message to file-queue


# 4. Configured Application Settings
Inside func-app-bank → Configuration:
Added:

<img width="974" height="635" alt="image" src="https://github.com/user-attachments/assets/fdcfdf58-f905-4bcf-b6d1-cb9199faf7fd" />


# 5. Tested End-to-End Workflow

## Step 1: Uploaded sample files in ADLS
Locations tested:
•	/raw/atm
•	/raw/customers
•	/raw/upi

## Step 2: Verified Events
In Storage → Events:
•	Event Grid showed Published, Delivered, Matched events
•	Successful delivery count increased

## Step 3: Function App Logs
Under Bank-Func-app→ Logs:
•	Confirmed Function triggered multiple times
•	No errors
•	Host running fine

## Step 4: Service Bus Verification
In Service-Bus-Bank→ queue_ingestion:
•	Successful Requests count increased
•	Message Count = count increased
•	No dead-letter
•	Queue successfully received + auto-cleared messages

All parts worked correctly.


# 6. Final Architecture Validated (Day 1 Output)
ADLS Gen2  →  Event Grid  →  Function App  →  Service Bus Queue
Everything is connected and functional.

