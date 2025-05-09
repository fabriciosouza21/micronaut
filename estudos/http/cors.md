**09-05-25**

# [Configure CORS in a Micronaut application](https://guides.micronaut.io/latest/micronaut-cors-maven-java.html)

## Pagina statica

`src/main/../../index.html`

``` html
<!DOCTYPE html>
<html>
        <head>
                <script>
                function request(method, url) {
                        const xhr = new XMLHttpRequest();
                        xhr.open(method, url);
                        xhr.onreadystatechange = function () {
                                document.getElementById("message").innerHTML = this.responseText;
                        };
                        xhr.send();
                }
                request("GET","http://localhost:8080/hello");
                </script>
        </head>
        <body>
        <h1 id="message">CORS not setup</h1>
        </body>
</html>
```

## Controller

``` java

package example.micronaut;

import io.micronaut.http.MediaType;
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;
import io.micronaut.http.annotation.Produces;

@Controller("/hello")
public class HelloController {
    @Get
    @Produces(MediaType.TEXT_PLAIN)
    public String index() {
        return "Hello World";
    }
}
```


## Enable cors

``` java
---
micronaut:
  server:
    cors:
      enabled: true
```


``` java
---
micronaut:
  server:
    cors:
      enabled: true

      configurations:
        ui:
          allowed-origins:
            - http://127.0.0.1:8000
```
