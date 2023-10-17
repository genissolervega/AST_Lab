# LAB 2.1 - Anàlisi d'APIs amb OWASP ZAP (DAST)

En aquest laboratori veurem com analitzar dinàmicament APIs amb l'eina OWASP ZAP, aplicant les relges del [OWASP Top 10](https://owasp.org/www-project-top-ten/)

Requisits:

- [Docker](https://docs.docker.com/)
- [Git](https://git-scm.com/)

## Clone del projecte

Primer de tot, farem un clone del projecte que analitzarem en aquesta pràctica. Com podeu veure, és el mateix projecte que hem vist al [laboratori 1 de SAST](../Lab1%20-%20SAST/README.md)

```bash
git clone https://github.com/TheMatrix97/dotnet-6-crud-api.git
````

## Inspecció del codi

Com podeu observar, es tracta d'una API REST creada amb `.NET 6.0`. Un `CRUD` bàsic d'usuaris amb els següents endpoints:

```text
POST   /users
GET    /users
GET    /users/{id}
PUT    /users/{id}
DELETE /users/{id}
```

Aquesta aplicació està basada en el projecte de l'usuari `cornflourblue` a GitHub. (<https://github.com/cornflourblue/dotnet-6-crud-api>). També podeu accedir al l'entrada del bloc de l'usuari on explica el funcionament de l'aplicació (<https://jasonwatmore.com/post/2022/03/15/net-6-crud-api-example-and-tutorial>).

## Execució de l'aplicació

Generem la imatge `Docker` a partir del `Dockerfile`

```bash
cd dotnet-6-crud-api
docker build -t crud_api:latest .
```

Crearem i executarem un contenidor basat en l'imatge que hem generat anteriorment, mapejant l'aplicació al port 4000.

```bash
docker run -d --name dotnet6-api -p 4000:4000 crud_api:latest
```

Si tot ha funcionat bé, hauriem de trobar l'aplicació aixecada al port `4000`. Si executem un `GET` al endpoint `/users`, ens hauria de tornar un array buit.

```bash
curl --location 'http://localhost:4000/users'

[]
```

Si estàs a windows i no disposes de l'eina [cURL](https://curl.se/), pots fer servir la interfície web de Swagger (http://localhost:4000/swagger/index.html) per generar les peticions HTTP.

Per poder afegir una persona, haurem de fer un POST a l'endpoint `users` amb aquesta informació:

```json
{
    "title": "Mr",
    "firstName": "Juan",
    "lastName": "Palomo",
    "role": "User",
    "email": "juan.palomo@gmail.com",
    "password": "secret",
    "confirmPassword": "secret"
}
```

```bash
curl --location 'http://localhost:4000/users' \
--header 'Content-Type: application/json' \
--data-raw '{
    "title": "Mr",
    "firstName": "Juan",
    "lastName": "Palomo",
    "role": "User",
    "email": "juan.palomo@gmail.com",
    "password": "secret",
    "confirmPassword": "secret"
}'
````

Aquesta petició ens hauria de tornar el seguent missatge

```json
{
    "message": "User created"
}
```

**Compte!** Aquesta petició pot tornar `error 500` en alguns casos, es tracta d'un comportament esperat. Si es així torna a executar la petició.


## Escaneig automatic amb OWASP ZAP

Ara que tenim l'API en marxa, executarem l'escaneig automatic de OWASP ZAP fent servir la definició de la mateixa en OpenAPI (http://localhost:4000/swagger/v1/swagger.json).

```bash
docker run --rm -t owasp/zap2docker-stable zap-api-scan.py -t http://host.docker.internal:4000/swagger/v1/swagger.json -f openapi
```

Aquesta comanda ens hauria de tornar el següent resultat

```text
....
PASS: Loosely Scoped Cookie [90033]
PASS: Cloud Metadata Potentially Exposed [90034]
PASS: Server Side Template Injection [90035]
PASS: Server Side Template Injection (Blind) [90036]
WARN-NEW: Unexpected Content-Type was returned [100001] x 3
        http://host.docker.internal:4000/swagger/index.html (200 OK)
        http://host.docker.internal:4000/swagger?class.module.classLoader.DefaultAssertionStatus=nonsense (200 OK)
        http://host.docker.internal:4000/swagger/ (200 OK)
WARN-NEW: X-Content-Type-Options Header Missing [10021] x 2
        http://host.docker.internal:4000/swagger/v1/swagger.json (200 OK)
        http://host.docker.internal:4000/Users (200 OK)
FAIL-NEW: 0     FAIL-INPROG: 0  WARN-NEW: 2     WARN-INPROG: 0  INFO: 0 IGNORE: 0       PASS: 110
```

Com podeu veure, l'aplicació no està tan malament, només genera 2 Warnings. 

A continuació, modificarem l'aplicació per generar més warnings. Intencionalment, afegirem un error 500 en el endpoint `GET /users` i inclourem informació del servidor a la capçalera.

Només haurem de modificar el fitxer `./Controllers/UsersControllers.cs`, i substituirem el mètode GetAll per aquest:

```cs
    [HttpGet]
    public IActionResult GetAll()
    {
        var users = _userService.GetAll();
        //Afegim Headers
        HttpContext.Response.Headers.Add("X-AspNet-Version", "2.0.0");
		HttpContext.Response.Headers.Add("X-Backend-Server", "localhost");
        //Provoquem error 500
		throw new ApplicationException("Oopss, Something gone wrong!");
        return Ok(users);
    }
```

Parem el servei de la API

```bash
docker rm -f dotnet6-api
```

Tornem a compilar la imatge docker i executem el contenidor de nou

```bash
docker build -t crud_api:latest .
docker run -d --name dotnet6-api -p 4000:4000 crud_api:latest
```

**Si ara tornem a executar el scanner, hauriem de veure alguns errors adicionals... Quins?**

```bash
docker run --rm -t owasp/zap2docker-stable zap-api-scan.py -t http://host.docker.internal:4000/swagger/v1/swagger.json -f openapi
```