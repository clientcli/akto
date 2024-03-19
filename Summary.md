# AKTO REPO INSTRUCTIONS

## 1. Components

- Opensource
    - **Dashboard**: serves UI and traffic processing. It uses ***Runtime analyzer*** as a library and interacts with ***Mongo***.
    - **Testing module**: polls ***Mongo*** for pending tests. If found, it starts testing APIs based on the test configuration. Then, it saves the reports in ***Mongo***.
    - **Puppeteer server**: automate auth token generation, triggered by Dashboard.
    - **MongoDB**: save data as data lake.

- Extensions
    - **Runtime analyzer** (Enterprise only): consumes all the traffic from ***Kafka*** to find out endpoints, auth policies, changes in APIs and populates them in the ***Mongo*** database.
    - **Context Analyzer** (Enterprise only): consumes data from ***Kafka*** for ML perspective to reduce false positives (FP) in test results. Then, context inferrence will be saved in ***Mongo***.
    - **Traffic mirroring module**: assemble packets in an order and extract HTTP req-resp pairs from the stream. Then push to ***Kafka***.

- Container service's information

  | Container              | Container Port | System Port |
  |------------------------|----------------|-------------|
  | mongo                  | 27017          | 27017       |
  | api-security-dashboard | 8080           | 9090        |
  | api-security-testing   |                |             |
  | puppeteer-replay       | 3000           | 3001        |

## 2. Docker Deploy

### 2.1. Prerequisites

- Docker

### 2.2. Docker Compose Deploy
Run the following commands to install open source

```
$ git clone https://github.com/akto-api-security/akto.git
$ cd akto/
$ docker-compose up -d
```

Then, all services are running as

<img src="/images/docker-compose-up.png" alt="docker-compose-up" width="200"/>

You should open the dashboard at http://localhost:9090. If we want to access from the outside as other computer, please make sure this port was opened for all IP. In that case, we need to change ``localhost`` to the IP of deployment place.

### 2.3. Dashboard Usage

Click on the Signup button to get started for the first time. If the account have already signed up, sign in to the account.

![sign-in-dashboard](/images/dashboard-login.png)

## 3. Function Analysis

### 3.1. Runtime analyzer
`$ apps/api-runtime/src/main/java/com/akto/runtime/Main.java`

- This function subcribe to "akto.api.logs" topic by default and make a polling (wait 10 seconds for each record).

Example API

  ```
    POST https://xyz.abc.com/api/v1/users?userId=500&creationFlow=true
    Request Payload

    {
        "userData": {
            "name": "user1"
            "status": "normalUser"
            "age": 20
        },
        "profileData": {
            "isActive": true,
            "createdAt": 1254345343
        }
    }

    Response Payload
    {
        "id": 500,
        "created": true,
        "username": "user1", 
    }

    Request Headers

    Content-Type: application/json
    Authorization: <Bearer-Token>
    Host: https://xyz.abc.com

    Response Headers

        access-control-allow-origin: "*"
        date: "Wed, 05 Jul 2023 09:53:32 GMT"
        content-length: "14871"
        server: "uvicorn"
        access-control-allow-credentials: "true"
        content-type: "application/json"
  ```

- The response data will be categorized based on `accountId`
- Perform parsing data using function `parseKafkaMessage`
    - HttpRequestParams

    ```
        - String method    // POST
        - String url
        - String type      // HTTP/1.1
        - Map<String, List<String>> headers 
        - String payload
        - int apiCollectionId
    ```

    - HttpResponseParams

    ```
        - Enum source       // HAR, PCAP, MIRRORING, SDK, OTHER, POSTMAN (API Connectors)
        - String accountId
        - String type       // HTTP/1.1
        - int statusCode    // 200
        - String status     // Message
        - Map<String, List<String>> headers 
        - String payload
        - int time
        - HttpRequestParams requestParams
        - boolean isPending
        - String orig
        - String sourceIP
    ```

- Updates occur at least once every hours to ensure that the application has up-to-date information
    ```
        if ((Context.now() - accountInfo.lastEstimatedCountTime) > 60*60) {
            accountInfo.lastEstimatedCountTime = Context.now();
            accountInfo.estimatedCount = SingleTypeInfoDao.instance.getMCollection().estimatedDocumentCount();
            accountInfo.setAccountSettings(AccountSettingsDao.instance.findOne(AccountSettingsDao.generateFilter()));
            loggerMaker.infoAndAddToDb("STI Estimated count: " + accountInfo.estimatedCount, LogDb.RUNTIME);
        }
    ```
- Send data to the central Kafka topic "akto.central" to save into database

### 3.2. Testing module
- Main task: Write custom tests and run to find vulnerabilities in APIs.
- YAML Test: defines a set of instructions for testing APIs to find security vulnerabilities. Akto provides more than 200 pre-built test cases in [here](https://github.com/akto-api-security/tests-library/tree/master) for `OWASP top 10` and `HackerOne Top 10`
    - ``6 Main Blocks of YAML Test``
  
      | Blocks                | Components                                                                           | Description                                                                              |
      |-----------------------|--------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
      | ID                    | id                                                                                   | Identifier for a particular test YAML file                                               |
      | Info                  | Name, Description, Details, Impact, Category, SubCategory, Severity, Tags, Reference | Metadata about the test                                                                  |
      | API Selection Filters | Operators                                                                            | Conditions for choosing whether APIs are eligible or not                                 |
      | Execute               | params, updated_params, Combining Conditions using Boolean Operators                 | a set of operations that can be performed on API calls that match the selection criteria |
      | Auth (Optional)       | authenticated, authorizations                                                        | security criteria                                                                        |
      | Validate              | Operators                                                                            | validation criteria                                                                      | 

    - ``Operators``: are the key to separate multiple conditions
  
      | Types            | Description                                                                                                                               |
      |------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
      | Parent Operators | `response_code`, `method`, `url`, `request_payload`, `response_payload`, `request_headers`, `response_headers`, `query_param`             |
      | Data Operators   | `regex`, `eq`, `neq`, `gt`, `gte`, `lt`,`lte`<br/> `not_contains`, `not_contains_either`, `contains_jwt`, `contains_all`, `contains_either` |
      | Collection Operators | `for_one` |
      | Combining Conditions using Boolean Operators | `or`, `and` |

- Write a custom test
  - Example Yaml with all possible operators
  ```
      id: Vulnerable_Test                # ID field serves as a unique
      info:
        name: ""                           # specifies the name or title of the test
        description: ""                    # provides a detailed explanation of the test. describes objectives, methodologies, and scope of the test
        details: ""                        # allows inclusion of additional information and context about the test
        impact: ""                         # describes the potential risks or consequences associated with the identified vulnerabilities
        category:                          # classifies the test into a specific category or domain
            name: ""                          
            shortName: ""
            displayName: ""
        subCategory: ""                    # this key also specifies the name or title of the test. should always contain the same value as id key
        severity: ""                       # indicates the severity level assigned to the identified vulnerabilities
        tags: ""                           # provides descriptive labels or keywords associated with the test
        references: ""                     # contains a list of relevant resources, documentation, or external links related to the test
      auth:
        authenticated: true                # makes sure that only authentiated api's get considered for a test.
      api_selection_filters:  
        response_code:                     # Filters API calls that return a response code between 200 and 300 (inclusive).      
            gte: 200                        
            lte: 300 
        url:       
            contains_all:                  # Filters API calls that contain the word "user" in the URL.
            - user
            extract: urlVar                # extracts the url value into a variable named urlVar
        method:    
            contains_either:               # Filters API calls that use either the POST, PATCH, or PUT HTTP methods
            - POST
            - PATCH
            - PUT
        request_payload:                   # Filters API calls whose request payload contains a key-value pair where the key matches the regex ".*age*." and the value is between 15 and 40 (inclusive)
            for_one:      
            key:
                regex: .*age*. 
                extract: ageVar            # extracts the matching key value into a variable named ageVar
            value:
                gt: 15  
                lt: 40  
        response_payload:                  # Filters API calls whose response payload does not contain the string "user2."
            not_contains: user2
        request_headers:                   # Filters API calls whose request header contains a whose value has a JWT token in it.
            for_one:
            key:
                contains_jwt: true          
        response_headers:                  # Filters API calls whose response header contains a key that exactly matches "server" and a value that matches the regex "nginx/1.8.0."
            for_one:
            key:
                eq: server
            value:
                regex: nginx/1.8.0
      execute: 
        type: single
        requests:
            - req:
            - modify_url: https://xyz.abc.com/api/v2/users    # Changes the URL of the API call to "[https://xyz.abc.com/api/v2/users."](https://xyz.abc.com/api/v2/users.%22)
            - modify_method: PATCH         # Changes the HTTP method of the API call to PATCH.
            - add_body_param:              # Adds a key-value pair "k1: v1" to the request body.
                k1: v1
            - modify_body_param:           # Changes the value of the "status" key in the request body to "admin."
                status: admin
            - delete_body_param: age       # Deletes the "age" key-value pair from the request body.
            - add_header:                  # Adds a "h1: v2" key-value pair to the request header.
                h1: v2
            - modify_header:               # Changes the value of the "host" key in the request header to "[https://xyz.evil.com](https://xyz.evil.com/)."
                host: https://xyz.evil.com
            - delete_header: authorization # Deletes the "authorization" key-value pair from the request header.
            - add_query_param:             # Adds a "q1: v3" key-value pair to the query string.
                q1: v3
            - modify_query_param:          # Changes the value of the "userId" key in the query string to "501."
                userId: 501
            - delete_query_param: creationFlow # Deletes the "creationFlow" key-value pair from the query string.
            - replace_body: '{"user": "newUser", "status": "admin"}'
            - remove_auth_header: true     # Replaces the entire request body with the JSON object 
            - follow_redirect: true        # Follows any HTTP redirects returned by the API call.
      validate:
        response_code:                     # Validates that the response code of the API call is 201.
            eq: 201
        response_payload:                  # Validates that the response payload is not empty and contains atleast one key named "success". Also it checks whether test response payload and sample response payload content are not similar(difference should be higher than 50%)
            length:
            gt: 0
            percentage_match:
            lt: 50
            contains_either:
            for_one:
                key: success
  ```

- Code Findings:






