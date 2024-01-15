> HTTP Request/Response 구조에 대해 알아보자

### Requset Message
    - {HTTP METHOD = GET} {Request Target = /abc/123} {HTTP Version = HTTP/1.1}
    - HEADERS → host, accept, connection, origin, cookie 등
    - BODY → query string. request body 등

### Response Message
    - {HTTP Version = HTTP/1.1} {Status Code=200} {Status Text = OK}
    - HEADERS → content type, content-length 등
    - BODY → response 값