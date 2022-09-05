# HTTP RPC server for IDF
This is a simple HTTP RPC server that works in espressif's IDF (esp8266/esp32). It uses the esp_http_server under the hood and provides 3 ways to interact with the server:

## Methods and Structs
```c
struct rpc_request
```
The `rpc_request` struct wraps the `httpd_req_t` type from IDF and provides some convenience methods for accessing HTTP request parameters.

```c
rpc_handler_t = esp_err_t (*)(rpc_request* req, char* resp, size_t len);
```

The `rpc_handler_t` type is used as a callback when a general function with arguments is called by the client. The `rpc_request` struct contains any arguments sent and the `resp` buffer can be used to write a response up to `len` bytes.

```c
template <typename T>
rpc_get_handler_t = T (*)();
```
The `rpc_get_handler_t` type is used when registering a `getter` i.e. a callback that return a specific data type.

```c
template <typename T>
rpc_set_handler_t = esp_err_t (*)(T val);
```
The `rpc_set_handler_t` type is used when registering a `setter` i.e. a callback that sets a specific variable.

```c
esp_err_t RpcServer::RegisterHandler(const char *uri, httpd_method_t method, rpc_handler_t handler)
```
This method is used to register a callback for a general function i.e. any function with any arguments.
```c
template <typename T>
esp_err_t RegisterGetHandler(const char *uri, rpc_get_handler_t<T> getter)
```
This method is used to register a handler for a `getter`.
```c
template <typename T>
esp_err_t RegisterSetHandler(const char *uri, rpc_set_handler_t<T> setter)
```
This method is used to register a handler for a `setter`.
```c
template <typename T>
esp_err_t RegisterPropertyHandler(const char *uri, rpc_get_handler_t<T> getter, rpc_set_handler_t<T> setter)
```
This method is shorthand for registering both a `getter` and a `setter` in the same function call.

## Example
```c
HttpRpcServer::HttpRpcServer()
:RpcServer(){
    InitMdns();
}


esp_err_t Handler(rpc_request* req, char* resp, size_t len){
    ushort param1;
    char param2[18];

    if (!req->TryGetParam("param1", param1))
    {
        return ESP_ERR_INVALID_ARG;
    }

    if (!req->TryGetParam("param2", &param2, 18))
    {
        return ESP_ERR_INVALID_ARG;
    }

    sprintf(resp, "%s: %d", param1);
    return ESP_OK;
}

esp_err_t HttpRpcServer::Start(){
    RpcServer::Start(10);

    const char* readProperty = "This is Read Only for the client";
    bool writeProperty = false;
    short readWriteProperty = 0;

    //Properties
    RegisterGetHandler<const char*>(
        "readProperty", 
        [](){ 
            return readProperty; 
        });

    RegisterSetHandler<bool>(
        "writeProperty", 
        [](val){
            writeProperty = val;
            return ESP_OK;
        }]);

    RegisterPropertyHandler<short>("readWriteProperty", 
    [](){       //Getter
        return readWriteProperty; 
        }, 
    [](val){    //Setter
        eadWriteProperty = val;
        return ESP_OK;
        });

    
    //Methods
    RegisterHandler("GetSomeInfo", HTTP_GET, Handler);

    return ESP_OK;
}
```