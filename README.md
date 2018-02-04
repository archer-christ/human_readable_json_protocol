# Human Readable Json Protocol
A way to convert Thrift Services and Functions into Human Readable JSON

> So, one problem I have heard is, “Hey, so we are publishing our Thrift APIs to third party vendors, and we need it to be easy to use.” Now, you either:
> 1. Expose the Thrift endpoints and coach your API consumers on Thrift and help them debug any issues.
> 2. You maintain a library that is wrapper around the different Thrift services for each language/platform you want to support.
> 3. Create a way to manually convert between human readable JSON to Thrift and vice versa, and maintain it for every single struct, service & function.

> If these options sound unappealing to you, you’re right — they can be a lot of work. But is there another way?

For a simple Thrift service like so:

```
exception SystemException {
  1: i32 errorCode,
  2: string message,
}

struct User {
  1: string id,
  2: string email,
  3: string name,
  4: i64 validatedAt
}

struct LoginResult {
  1: string authToken,
  2: User currentUser
}

service AuthenticationService {
  LoginResult login(1: string email, 2: string password) throws(1: SystemException err)
}
```

Basic Format of the request:
```json
{
  "method": "METHOD_NAME",
  "arguments": { ... }
}
```

Example:
```json
{
  "method": "login",
  "arguments": {
    "email": "devansh@wi.co",
    "password": "password"
  }
}
```

**_The arguments and the names have to match up exactly as specified in the Thrift definition files._**

Basic format of the response
```json
{
  "method": "METHOD_NAME",
  "result|exception": { ... }
}
```
**_Note that the JSON objects have keys whose names are the same as that of fields in the struct defined in Thrift_**

Example:
```json
{
  "method": "login",
  "result": {
    "success": {
      "authToken": "some_auth_token",
      "currentUser": {
        "id": "6a6c982b-62f9-46d2-aff9-bd3a1cdf43f9",
        "email": "user1@wi.co",
        "name": "user1",
        "validatedAt": 0
      }
    }
  }
}
```


**_If there was an error thrown which was defined in the method definition_**
```json
{
    "method": "login",
    "result": {
        "err": {
            "errorCode": 401,
            "message": "Invalid email or password"
        }
    }
}
```

For all other errors:
```json
{
    "method": "loginUser",
    "exception": {
        "message": "Unknown function loginUser",
        "type": 1
    }
}
```


## How to use it

1. You will have to make a small modification to the Thrift Source Code for JSON metadata generator in order for this to work. You can look at the diff of the change that is required over here: [JSON Generator Diff](diff_for_t_json_generator_cc.diff).

All this change does it to make sure when the class type is written out it displays as *package.class_name* instead of simply *class_name*.

You can also just use this Thrift Binary (v0.10.0): [thrift](thrift)

2. Generate JSON using Thrift with the `full_name` option:

```
./thrift -r -out PATH_TO_OUTPUT_DIRECTORY -gen json:full_name THRIFT_FILE
```

3. Parse all the JSON Files and concat them into an JSON Array

For example in Java:

```java
public static final JSONArray HUMAN_JSON_THRIFT_METADATA = readAllFiles();

private static JSONArray readAllFiles(String jsonMetadataPath) {
    File[] jsonFiles = new File[0];
    try {
        jsonFiles = new File(jsonMetadataPath).listFiles();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    JSONArray jsonArray = new JSONArray();

    for (File file : jsonFiles) {
        try (BufferedReader br = new BufferedReader(new FileReader(file.toPath().toString()))) {
            String s = "";
            while (br.ready()) {
                s += br.readLine() + "\n";
            }
            jsonArray.put(new JSONObject(s));
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    return jsonArray;
}
```

4. Create the HumanReadableJsonProtocol using the Factory, and the just use it like any other protocol like TBinaryProtocol, etc:

```java
String serviceName = "AuthenticationService"; // The exact name of the service
new HumanReadableJsonProtocol.Factory(HumanReadableJsonHelpers.HUMAN_JSON_THRIFT_METADATA, serviceName).getProtocol(transport);
```