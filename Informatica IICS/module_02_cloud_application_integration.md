# Complete Beginner's Guide to Cloud Application Integration (CAI)

> **IICS Service:** Cloud Application Integration (CAI)
> **Difficulty:** Beginner
> **Estimated Reading Time:** 30 minutes
> **Prerequisites:** An IICS organization account (free trial is sufficient), basic understanding of what an API and a web service are
> **Learning Objectives:**
> - Explain what Cloud Application Integration is and how it differs from Cloud Data Integration
> - Define the three core CAI asset types: Application Integration Processes, Service Connectors, and Process Objects
> - Design a simple Application Integration Process that receives data via an API call and performs an action
> - Create a Service Connector to call an external REST API
> - Use a Process Object to store and pass structured data within a process
> - Identify real-world scenarios where CAI is the right choice over CDI

---

## 🧭 What Is Cloud Application Integration (CAI)?

**Cloud Application Integration (CAI)** (the IICS service that lets you build event-driven, real-time integrations between applications using APIs, events, and orchestration logic) is designed for a fundamentally different purpose than CDI.

Here's the simplest way to understand the difference:

| | CDI | CAI |
|---|-----|-----|
| **Metaphor** | A cargo truck that moves pallets of goods on a schedule | A postal worker who delivers one letter the instant it arrives |
| **Trigger** | Scheduled (e.g., "run every night at 2 AM") or manual | Event-driven (e.g., "when a new order is placed, do this immediately") |
| **Data Volume** | Thousands to millions of rows per run | One record or message at a time |
| **Primary Use** | Batch data warehouse loading, bulk replication | Real-time API orchestration, workflow automation |
| **Design Canvas** | Mapping Designer (data flow) | Process Designer (event/action flow) |

> 💡 **When to Use CAI vs. CDI**
> Use **CDI** when you need to move and transform large volumes of data in batches. Use **CAI** when you need to react to individual events in real time — for example, when a customer submits an order on your website and you need to instantly create a record in Salesforce, send a confirmation email, and update inventory in SAP.

---

## 🏗️ The Three Core CAI Asset Types

CAI has three primary building blocks. They work together like this:

```
External Event (e.g., API call, message)
        │
        ▼
┌────────────────────────┐
│ Application Integration │  ← The orchestration engine (the "brain")
│       Process           │
│                         │
│  ┌──────────────────┐   │
│  │  Process Object   │  ← Structured data container (the "clipboard")
│  └──────────────────┘   │
│                         │
│  ┌──────────────────┐   │
│  │ Service Connector │  ← Calls to external APIs (the "hands")
│  └──────────────────┘   │
└────────────────────────┘
        │
        ▼
External System (e.g., Salesforce, SAP, REST API)
```

| Asset Type | Role | Analogy |
|-----------|------|---------|
| **Application Integration Process** | Defines the workflow — what happens, in what order, under what conditions | The recipe that a chef follows step by step |
| **Service Connector** | Connects to and calls external APIs or web services | The chef's phone that lets them call the supplier for ingredients |
| **Process Object** | Defines the shape of data passed between steps in a process | The order ticket that the chef reads to know what to cook |

---

## 📋 Deep Dive: Process Objects — Defining Your Data Shape

We start with **Process Objects** because you'll usually create them first — they define the data structure that your process will work with.

A **Process Object** (a reusable data structure definition in CAI that describes the fields, data types, and hierarchy of data flowing through a process — similar to a schema or a class definition) is like a blueprint for a form. It doesn't contain data itself; it defines what the data looks like.

### Why Process Objects Exist

When an external system calls your integration process, it sends data — perhaps a JSON payload with customer name, email, and order total. Before your process can work with that data, IICS needs to know the structure: what fields exist, what types they are, which ones are required.

### Creating a Process Object

**Scenario:** You're building an integration that receives new customer orders. Each order has an order ID, customer name, order total, and a list of line items.

1. From the IICS home page, click **Application Integration** in the left navigation
2. Click **New** → **Process Object**
3. Name it `PO_CustomerOrder`
4. In the **Structure** panel, add fields:

| Field Name | Data Type | Description |
|-----------|-----------|-------------|
| `orderId` | String | Unique order identifier |
| `customerName` | String | Customer's full name |
| `orderTotal` | Decimal | Total order amount |
| `orderDate` | DateTime | Date and time of the order |

5. To add nested structures (like line items), click **Add Field** → set the type to **Object** or **Array of Objects** → define child fields (`lineItemId`, `productName`, `quantity`, `unitPrice`)
6. Click **Save**

> 💡 **Process Object vs. Database Table**
> A Process Object is NOT a table. It doesn't store data persistently. It's a **shape definition** — like saying "a customer order looks like this." Data conforming to this shape flows through your process at runtime and disappears when the process instance completes (unless you save it somewhere).

---

## 🔌 Deep Dive: Service Connectors — Reaching External Systems

A **Service Connector** (a CAI asset that wraps an external REST or SOAP web service, making its operations available as drag-and-drop steps inside your Application Integration Process) is how your process communicates with the outside world.

### Types of Service Connectors

| Type | When to Use | Example |
|------|------------|---------|
| **REST Service Connector** | Calling modern web APIs that use JSON over HTTP | Calling a Slack webhook, querying a REST API, posting to Salesforce |
| **SOAP Service Connector** | Calling legacy enterprise web services that use XML/WSDL | Calling an older SAP or Oracle EBS web service |

### Creating a REST Service Connector

**Scenario:** You want your process to call a third-party shipping API to calculate shipping rates.

1. Click **Application Integration** → **New** → **Service Connector**
2. Name it `SC_ShippingRateAPI`
3. Select **Type**: REST
4. In the **Connection** section:
   - **Base URL**: Enter the API's base URL (e.g., `https://api.shippingprovider.com/v2`)
   - **Authentication**: Select the method (None, Basic, OAuth 2.0, Custom Header)
   - If using **Basic Auth**, enter the username and password
   - If using **OAuth 2.0**, configure the token endpoint, client ID, and client secret
5. Click **Add Operation** to define the specific API calls:
   - **Operation Name**: `calculateRate`
   - **HTTP Method**: POST
   - **Path**: `/rates/calculate`
   - **Request Body**: Define or import the JSON schema (e.g., `{"weight": 5.0, "destination": "US"}`)
   - **Response Body**: Define or import the expected response schema
6. Click **Test** to verify the connection with sample data
7. Click **Save**

> ⚠️ **Security Note:** Never hardcode API keys or passwords directly into a Service Connector in a shared org. Use IICS **Secure Parameters** [VERIFY: confirm the exact IICS feature name for storing secrets in CAI] to store sensitive credentials separately from the asset definition.

---

## ⚙️ Deep Dive: Application Integration Processes — The Orchestration Engine

An **Application Integration Process** (a CAI asset that defines a sequence of steps — receiving events, making decisions, calling services, transforming data — executed in real time when triggered by an event or API call) is the centerpiece of CAI. This is where everything comes together.

### The Process Designer Canvas

The **Process Designer** (the browser-based visual workspace for building Application Integration Processes) uses a flowchart-style canvas. You define:

- **Start**: What triggers the process (an API call, a schedule, an event)
- **Steps**: The actions performed in sequence or in parallel
- **Decisions**: Conditional branching ("if order total > 1000, do X; otherwise do Y")
- **End**: Where the process completes and optionally returns a response

### Key Step Types for Beginners

| Step Type | What It Does | When to Use It |
|----------|-------------|----------------|
| **Service** | Calls a Service Connector operation | When you need to reach an external API |
| **Assignment** | Sets the value of a field or variable | When you need to map data between steps or compute a value |
| **Decision** | Branches the flow based on a condition | When different outcomes require different actions |
| **Notification** | Sends an email or other notification | When you need to alert someone about an event |
| **Subprocess** | Calls another Application Integration Process | When you want to reuse common logic across processes |
| **Logging** | Writes a message to the process log | For debugging and audit trails |

### Step-by-Step: Building a Simple Order Processing Process

**Scenario:** When a new customer order arrives via API call, the process should:
1. Receive the order data
2. Check if the order total exceeds $500
3. If yes — call the shipping API for expedited shipping rates
4. If no — assign standard shipping
5. Log the outcome and respond with a confirmation

**Create the Process:**
1. Click **Application Integration** → **New** → **Process**
2. Name it `AP_ProcessCustomerOrder`
3. Click **Create** — the Process Designer canvas opens

**Configure the Start Event:**
4. Click the **Start** element on the canvas
5. Under **Type**, select **API** (this means the process is triggered by an HTTP request)
6. Define the **Input**: select your `PO_CustomerOrder` Process Object as the input schema
7. Define the **Output**: create a simple response structure with fields `status` (String) and `message` (String)

**Add a Decision Step:**
8. Drag a **Decision** step from the palette onto the canvas after the Start
9. Name it `CheckOrderTotal`
10. Define the condition: `$inputData/orderTotal > 500`
11. This creates two branches: **True** and **False**

**Add a Service Step (True Branch):**
12. On the **True** branch, drag a **Service** step
13. Name it `CallShippingAPI`
14. Select the `SC_ShippingRateAPI` Service Connector → operation `calculateRate`
15. Map the input fields: weight from the order data, destination from a default or order field

**Add an Assignment Step (False Branch):**
16. On the **False** branch, drag an **Assignment** step
17. Name it `AssignStandardShipping`
18. Set a field like `shippingMethod = 'Standard Ground'`

**Add a Logging Step:**
19. After both branches merge, add a **Logging** step
20. Log a message like: `"Order " + $inputData/orderId + " processed with shipping method: " + $shippingMethod`

**Configure the End Event:**
21. Connect the flow to the **End** element
22. Map the output fields: `status = 'Confirmed'`, `message = 'Order processed successfully'`

**Publish and Test:**
23. Click **Save**
24. Click **Publish** — this deploys the process and generates a REST API endpoint
25. Note the generated URL — you can call this from any HTTP client (Postman, curl, another application)
26. Test by sending a POST request with a sample JSON order

---

## 🌐 How CAI Exposes Your Process as an API

One of CAI's most powerful features is automatic API generation. When you publish an Application Integration Process with an API-type start event, IICS automatically:

- Creates a REST API endpoint
- Generates the URL (typically: `https://<your-pod>.informaticacloud.com/active-bpel/rt/<process-name>`)
- Applies your input/output schemas from the Process Object
- Handles authentication based on your org's settings

External systems can call this endpoint to trigger your process — no additional API gateway configuration needed (though production deployments often add one for rate limiting and monitoring).

---

## ⚠️ Common Mistakes & Troubleshooting

| Mistake | Why It Happens | How to Fix It |
|---------|---------------|---------------|
| Process publishes but API call returns 401 Unauthorized | The caller is not passing valid IICS credentials or session token in the request headers | Include your IICS session token or configure API-level authentication. Check the API documentation generated at publish time |
| Service Connector test succeeds but fails at runtime | The Service Connector was tested from the IICS cloud, but at runtime the Secure Agent's network can't reach the external API | Verify network connectivity from the Secure Agent machine to the external API endpoint. Check firewalls and proxy settings |
| Decision step always takes the same branch | The condition expression has a syntax error or references the wrong field path | Double-check the XPath or field path used in the condition. Use the **Logging** step before the Decision to print the actual value being evaluated |
| Process Object fields appear empty at runtime | The input data doesn't match the expected Process Object structure (e.g., mismatched field names, wrong JSON nesting) | Compare the incoming JSON payload against the Process Object definition. Field names are case-sensitive |
| Process runs forever and never reaches the End step | An infinite loop exists (e.g., a Subprocess calls itself) or a Service call hangs without a timeout | Add timeout settings to Service steps. Review the flow for circular Subprocess references |

---

## 📝 Key Takeaways

- **CAI is for real-time, event-driven integration** — it reacts to individual events instantly, unlike CDI which processes data in bulk batches
- **The three building blocks work together:** Process Objects define data shape, Service Connectors reach external systems, and Application Integration Processes orchestrate the workflow
- **Publishing a process automatically creates a REST API endpoint** — external systems can call your integration without any additional API infrastructure
- **Process Objects are blueprints, not storage** — they define what data looks like but don't persist data themselves
- **Always test Service Connectors independently before using them in a process** — network and authentication issues are easier to diagnose in isolation

---

## 🧪 Practice Exercise

**Scenario:** Your company wants a real-time integration that receives a "new employee" notification via API call and logs the employee's name and department.

**Your Task:**
1. Create a **Process Object** named `PO_NewEmployee` with fields: `employeeId` (String), `firstName` (String), `lastName` (String), `department` (String), `startDate` (DateTime)
2. Create an **Application Integration Process** named `AP_NewEmployeeNotification`:
   - Start event type: **API**
   - Input: your `PO_NewEmployee` Process Object
   - Add a **Decision** step: if `department = 'Engineering'`, go to one branch; otherwise, go to another
   - On the Engineering branch: add a **Logging** step that logs `"Engineering hire: " + firstName + " " + lastName`
   - On the other branch: add a **Logging** step that logs `"New hire: " + firstName + " " + lastName + " in " + department`
   - End the process with output: `status = 'Received'`
3. **Publish** the process
4. Test by sending a POST request (using a tool like Postman or curl) with a sample JSON payload
5. Check the **Process Monitor** for your process instance — verify the correct branch was taken and the log message is correct

**Expected Outcome:** The process instance shows "Completed" status. The log message reflects the correct branch based on the department you sent.

---

## 📚 Glossary

| Term | Definition |
|------|-----------|
| **Cloud Application Integration (CAI)** | The IICS service for building real-time, event-driven integrations between applications using APIs and orchestration logic |
| **Application Integration Process** | A CAI asset defining a sequence of steps (receiving events, calling services, making decisions) executed in real time |
| **Service Connector** | A CAI asset that wraps an external REST or SOAP web service, making it callable from within a process |
| **Process Object** | A reusable data structure definition that describes the shape of data flowing through a process |
| **Process Designer** | The browser-based visual canvas for building Application Integration Processes |
| **REST** | Representational State Transfer — a modern architectural style for web APIs that typically uses JSON over HTTP |
| **SOAP** | Simple Object Access Protocol — an older, XML-based protocol for web services, common in enterprise systems |
| **Publish** | The action of deploying an Application Integration Process, making it live and generating its API endpoint |
| **Decision Step** | A Process step that creates conditional branches based on a Boolean expression |
| **Assignment Step** | A Process step that sets or changes the value of a field or variable |
| **Subprocess** | A Process step that calls another Application Integration Process, enabling reuse of common logic |
| **Event-Driven** | An integration pattern where processing is triggered by the occurrence of an event (API call, message arrival) rather than a schedule |
| **API Endpoint** | A specific URL that external systems call to trigger a published Application Integration Process |

---

## 🔗 What to Learn Next

- **Advanced Process Patterns in CAI:** Error handling with fault handlers, parallel execution, timer events, and compensation logic
- **Connecting CAI with CDI:** Learn how an Application Integration Process can trigger a CDI Mapping Task, enabling real-time event capture followed by batch data loading
- **Building Service Connectors for Common Platforms:** Step-by-step guides for connecting to Salesforce, SAP, Workday, and Slack APIs using Service Connectors
