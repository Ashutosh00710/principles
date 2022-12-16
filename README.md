# Principles for dealing with Logs

Before going to the principles you must know what we are trying to achieve and what are the goals of log improvements and standardization. Consider a microservice architecture, that has 3 services (let's say S1, S2, S3). There is a transaction which is going through all 3 services.

`[FRONT END (Mobile App)] --> [S1] --> [S2] --> [S3]`

For some reason, the user gets an error on his app. Now our task is to figure out where this issue is coming from. **Since codes are written by different people, we can't gurantee any sort of consistency in logs across services. It is difficult to query our logs if there are no standards. We must have a standard for logs so that we can query them easily.** The classical way of doing this is to have a correlation ID (or a request ID or a tracking ID) for every log. It will help us track logs across services, which will result in optimal issue tracking.

Other than tracking we must have a clear description in logs, so that we can have a better understanding on what is happening (or happened) in our app (or server).

Now I guess you have a clear understanding of what problems we are trying to solve. It's time for principles:

1. Understand the log levels clearly.
2. Do not log anything and everything. Select log statements precisely.
3. Must have a clear strategy around handled and unhandled errors. ERROR level logs must contain a stack trace.
4. The INFO level must have some sense of story or procedure (without exposing any sensitive details).
5. The DEBUG level must have all the necessary internal and sensitive details.
6. Must write appropriate INFO and DEBUG level logs. This will help other developers while debugging the code.
7. Every log (any level) must have a correlation ID (or request ID); this will help while tracing the logs across multiple instances of multiple services.

#### Log Levels

| Log Levels | Priority |                                                                     Description                                                                     | Enable in Production |
| :----------: | :--------: | :----------------------------------------------------------------------------------------------------------------------------------------------------: | :--------------------: |
|   ERROR   |  40,000  | Error events of considerable importance that will prevent <br />normal program execution, but might still allow the <br />application to continue running.<br /> |          ✅          |
|    WARN    |  30,000  |                   Potentially harmful situations of interest to end users <br />or system managers that indicate potential problems.<br />                   |          ✅          |
|    INFO    |  20,000  |      Informational messages that might make sense to end <br />users and system administrators, and highlight the <br />progress of the application.<br />      |          ❌          |
|   DEBUG   |  10,000  |                                       Highly detailed tracing messages. Produces the most voluminous output.<br />                                       |          ❌          |

⚠️ For ERROR, WARN and INFO ⚠️

Do not expose any sensitive information. Do not log anything other than Object Id or Recource Id

#### Log Structure

`[Log Level] [Timestamp] [Correlation Id] [ServiceName] [ClassName.MethodName] [Log String] [Log Metadata]`

* Correlation Id: This will be a uuid which the initiation of request will produce and this id will remains same throughout the request chain.
  **NOTE: This field is optional or not required for independent services.**
* Service Name: The name of the service.
* Log String: Generally a message to print, while writing these you can consider thinking of following points (NOT MANDATORY):

  * What happened (with appropriate detail; “Something happened” is not
    generally useful).
  * When did it happen (and when did it start and end, if relevant).
  * Where did it happen (on what host, what file system, which network
    interface, etc.).
  * Who was involved.
  * Where he, she or it came from.
  * Where do I get more information.?????
  * How certain should I be that the above is really what happened?
  * What is affected. (Hint !!)
  * What will happen next.
* Log Metadata: Any essential information related to **log string**

  * request params (For request recieved on controller)
  * IP from where the request is recieved
  * successful execution response of every subtask
  * can include error stacks etc.


#### Reparing Current Logging Strategy

While reparing logs you must consider below points:

* All log levels should be present in each API. While writing logs along with appropriate log level thinking process should be:

  * Write logs that can help you debugging the issue while running server locally or in dev environment.
  * Don't log anything and everything.
* Apply log levels as mentioned in the above table.
* Log Structure should be regular throughout the service (code).

#### Examples:

**Info Logs Example:** While reading the below example keep the quoted lines in mind,

> *&quot;Informational messages that might make sense to end users and system administrators, and highlight the progress of the application.&quot;*



```js
function purchaseItem(itemList) {
  try {
    console.info("Purchase Item");
    console.info("Picking items in the item list from the Shop");
    const items = pickItemsFromTheShop(itemList);

    console.info("Adding picked items to the cart");
    const cart = addToCart(items);

    console.info("Proceeding to checkout. Calculating total amount");
    const totalPrice = checkout(cart);

    console.info("Processing Payment");
    processPayment(UPI_ID, UPI_PIN, totalPrice);
    console.info("Payment processed successful");

    console.info("Items are successfully purchased");
  } catch (error) {}
}
```

Console

```stdout
  [INFO] Purchase Item
  [INFO] Picking items in the item list from the Shop
  [INFO] Adding picked items to the cart
  [INFO] Proceeding to checkout. Calculating total amount
  [INFO] Processing Payment
  [INFO] Payment processed successful
  [INFO] Items are successfully purchased
```

**Error Logs Example:** keep the quoted lines in mind,

> *&quot;Error events of considerable importance that will prevent normal program execution, but might still allow the application to continue running.&quot;*

```js
function processPayment(upiId, upiPin, amount, userId) {
  console.info("Process Payment");

  console.info("Verifying VPA");
  const { token, failedCount } = verifyVPA(upiId, upiPin);
  // handled error
  if (token == null) throw new Error("Incorrect UPI ID or PIN");
  console.info("Verifying VPA successful");

  console.info("Verifying Amount deduction");
  const canDeduct = verifyAmount(token, amount);
  // handled error
  if (!canDeduct) throw new Error("Insufficient balance");
  console.info("Verifying Amount deduction successful");

  console.info("Processing payment through payment gateway");
  // razorPayGateway can give an unhandled error
  const processPayment = razorPayGateway(upiId, upiPin, amount, userId);
  console.info("Processing payment through payment gateway successful");
}

function purchaseItem(itemList, userId) {
  try {
    console.info("Purchase Item for", userId);
    console.info("Picking items in the item list from the Shop");
    const items = pickItemsFromTheShop(itemList);

    console.info("Adding picked items to the cart");
    const cart = addToCart(items);

    console.info("Proceeding to checkout. Calculating total amount");
    const totalPrice = checkout(cart);

    console.info("Processing Payment");
    // ❌⬇️⬇️ Point of error ⬇️⬇️❌
    processPayment(UPI_ID, UPI_PIN, totalPrice, userId);
    console.info("Payment processed successful");

    console.info("Items are successfully purchased");
  } catch (error) {
    console.error("Purchase Failed for", userId);

    // 1. printing trace will let developer know the exact point of incident
    // 2. printing message will let developer know the context of the error
    console.error(error.message, error.trace);

    // handle the error the way you want
  }
}
```

Console

```stdout
  [INFO] Purchase Item for 1234
  [INFO] Picking items in the item list from the Shop
  [INFO] Adding picked items to the cart
  [INFO] Proceeding to checkout. Calculating total amount
  [INFO] Processing Payment
  [INFO] Process Payment
  [INFO] Verifying VPA
  [ERROR] Purchase Failed for 1234
  [ERROR] Incorrect UPI ID or PIN
  [ERROR] File Path: C:/lab/test-lib/index.js
  Class: TestClass
  Function: TestClass.sampleFunction
    7  |  console.info("Verifying VPA");
    8  |  const token = verifyVPA(upiId, upiPin);
  > 9  |  if (token == null) throw new Error("Incorrect UPI ID or PIN")
                             ^
    10 |  console.info("Verifying VPA successful");
    11 |  console.info("Verifying Amount deduction");


  Error: Purchase Failed for
      at purchaseItem (file:///C:/lab/test-lib/index.js:10:13)
      at processPayment (file:///C:/lab/test-lib/index.js:17:17)
      at ModuleJob.run (internal/modules/esm/module_job.js:183:25)
      at async Loader.import (internal/modules/esm/loader.js:178:24)
      at async Object.loadESM (internal/process/esm_loader.js:68:5)
```

**Warning Logs Example:** While reading the below example keep the quoted lines in mind,

> *&quot;Potentially harmful situations of interest to end users or system managers that indicate potential problems.&quot;*

```js
function processPayment(upiId, upiPin, amount, userId) {
  console.info("Process Payment");

  console.info("Verifying VPA");
  const { token, failedCount } = verifyVPA(upiId, upiPin);
  // handled error
  if (token == null) {
    if (failedCount == 3)
      console.warn(
        "3 incorrect attempts for",
        userId,
        "maybe someone who is not the owner of this account is trying to process the payment"
      );
    throw new Error("Incorrect UPI ID or PIN");
  }
  console.info("Verifying VPA successful");

  console.info("Verifying Amount deduction");
  const canDeduct = verifyAmount(token, amount);
  // handled error
  if (!canDeduct) throw new Error("Insufficient balance");
  console.info("Verifying Amount deduction successful");

  console.info("Processing payment through payment gateway");
  // razorPayGateway can give an unhandled error
  const processPayment = razorPayGateway(upiId, upiPin, amount, userId);
  console.info("Processing payment through payment gateway successful");
}
```

Console

```stdout
  [INFO] Processing Payment
  [INFO] Process Payment
  [INFO] Verifying VPA
  [WARNING] 3 incorrect attempts for 1234 maybe someone who is not the owner of this account is trying to process the payment
  [ERROR] Purchase Failed for 1234
  [ERROR] Incorrect UPI ID or PIN
  [ERROR] File Path: C:/lab/test-lib/index.js
  Class: TestClass
  Function: TestClass.sampleFunction
    7  |  console.info("Verifying VPA");
    8  |  const token = verifyVPA(upiId, upiPin);
  > 9  |  if (token == null) throw new Error("Incorrect UPI ID or PIN")
                             ^
    10 |  console.info("Verifying VPA successful");
    11 |  console.info("Verifying Amount deduction");


  Error: Purchase Failed for
      at purchaseItem (file:///C:/lab/test-lib/index.js:10:13)
      at processPayment (file:///C:/lab/test-lib/index.js:17:17)
      at ModuleJob.run (internal/modules/esm/module_job.js:183:25)
      at async Loader.import (internal/modules/esm/loader.js:178:24)
      at async Object.loadESM (internal/process/esm_loader.js:68:5)
```

**Debug Logs Example:** While reading the below example keep the quoted lines in mind,

> *&quot;Potentially harmful situations of interest to end users or system managers that indicate potential problems.&quot;*

```js
function processPayment(upiId, upiPin, amount, userId) {
  console.info("Process Payment");
  console.debug("Input params for Process Payment", upiId, upiPin, amount, userId);
  console.info("Verifying VPA");
  const { token, failedCount } = verifyVPA(upiId, upiPin);
  console.debug("account token:", token, "failed attempts:", failedCount);
  // handled error
  if (token == null) {
    if (failedCount == 3)
      console.warn(
        "3 incorrect attempts for",
        userId,
        "maybe someone who is not the owner of this account is trying to process the payment"
      );
    throw new Error("Incorrect UPI ID or PIN");
  }
  console.info("Verifying VPA successful");

  console.info("Verifying Amount deduction");
  const canDeduct = verifyAmount(token, amount);
  console.debug("Can deduct:", deduct);
  // handled error
  if (!canDeduct) throw new Error("Insufficient balance");
  console.info("Verifying Amount deduction successful");

  console.info("Processing payment through payment gateway");
  // razorPayGateway can give an unhandled error
  const processedPayment = razorPayGateway(upiId, upiPin, amount, userId);
  console.debug("Processed payment response from razor pay:", processedPayment);
  console.info("Processing payment through payment gateway successful");
}
```

Console

```stdout
  [INFO] Processing Payment
  [DEBUG] Input params for Process Payment abc@okaxis 098# 1000 1234
  [INFO] Process Payment
  [INFO] Verifying VPA
  [DEBUG] account token: 193287Y2874T2Y873273Y1287 failed attempts: 3
  [WARNING] 3 incorrect attempts for 1234 maybe someone who is not the owner of this account is trying to process the payment
  [ERROR] Purchase Failed for 1234
  [ERROR] Incorrect UPI ID or PIN
  [ERROR] File Path: C:/lab/test-lib/index.js
  Class: TestClass
  Function: TestClass.sampleFunction
    7  |  console.info("Verifying VPA");
    8  |  const token = verifyVPA(upiId, upiPin);
  > 9  |  if (token == null) throw new Error("Incorrect UPI ID or PIN")
                             ^
    10 |  console.info("Verifying VPA successful");
    11 |  console.info("Verifying Amount deduction");


  Error: Purchase Failed for
      at purchaseItem (file:///C:/lab/test-lib/index.js:10:13)
      at processPayment (file:///C:/lab/test-lib/index.js:17:17)
      at ModuleJob.run (internal/modules/esm/module_job.js:183:25)
      at async Loader.import (internal/modules/esm/loader.js:178:24)
      at async Object.loadESM (internal/process/esm_loader.js:68:5)
```
