# LAB 2.2 - Fuzzing d'APIs amb Microsoft RESTler (DAST)

En aquest laboratori veurem com configurar l'eina [RESTler de Microsoft](https://github.com/microsoft/restler-fuzzer) per fer `smoke tests` i `fuzzing tests` sobre API's.
Aquesta pràctica és la continuació del laboratori [`2.1 - DAST ZAP`](../Lab2.1%20-%20DAST%20ZAP/README.md), es recomana fer-lo abans de començar aquest.

Requisits:

- [Docker](https://docs.docker.com/)
- [Git](https://git-scm.com/)

## Clone del projecte

Primer de tot, farem un clone del projecte que analitzarem en aquesta pràctica. Com podeu veure, és el mateix projecte que hem vist al [laboratori 1 de SAST](../Lab1%20-%20SAST/README.md)

```bash
git clone https://github.com/TheMatrix97/dotnet-6-crud-api.git
````


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

## Desplegament contenidor RESTler

Primer, desplegarem la imatge oficial de restler de Microsoft, aquesta conté l'eina ja preinstalada.

```bash
docker run --rm -it mcr.microsoft.com/restlerfuzzer/restler:v9.2.2 sh
```

Si tot ha anat bé, hauriem de disposar d'un terminal sh dins del contenidor que s'ha aixecat.
```txt
/ # ls -la
total 68
drwxr-xr-x    1 root     root          4096 Oct 13 10:04 .
drwxr-xr-x    1 root     root          4096 Oct 13 10:04 ..
-rwxr-xr-x    1 root     root             0 Oct 13 10:04 .dockerenv
drwxr-xr-x    1 root     root          4096 Jul 18 00:04 RESTler
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 bin
drwxr-xr-x    5 root     root           360 Oct 13 10:04 dev
drwxr-xr-x    1 root     root          4096 Oct 13 10:04 etc
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 home
drwxr-xr-x    1 root     root          4096 Jul 11 13:14 lib
drwxr-xr-x    5 root     root          4096 Jun 14 15:03 media
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 mnt
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 opt
dr-xr-xr-x  299 root     root             0 Oct 13 10:04 proc
drwx------    1 root     root          4096 Oct 13 10:05 root
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 run
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 sbin
drwxr-xr-x    2 root     root          4096 Jun 14 15:03 srv
dr-xr-xr-x   11 root     root             0 Oct 13 10:04 sys
drwxrwxrwt    2 root     root          4096 Jun 14 15:03 tmp
drwxr-xr-x    1 root     root          4096 Jul 18 00:03 usr
drwxr-xr-x   12 root     root          4096 Jun 14 15:03 var
/ #
```

Crearem una carpeta a l'arrel on guardarem els resultats de les execucions.

```bash
mkdir api && cd api
```

## Compilar recursos

A continuació, haurem de descarregar la definició OpenAPI de l'api que tenim desplegada

```bash
wget http://host.docker.internal:4000/swagger/v1/swagger.json
```

Seguidament, generarem els recursos per executar l'eina, a partir de la definició que hem descarregat al pas anterior

```bash
dotnet /RESTler/restler/Restler.dll compile --api_spec ./swagger.json
```

Si tot ha anat bé, ens hauria d'apareixer un directori anomenat `Compile`.

```text
/api # ls -la
total 24
drwxr-xr-x    4 root     root          4096 Oct 13 10:11 .
drwxr-xr-x    1 root     root          4096 Oct 13 10:08 ..
drwxr-xr-x    3 root     root          4096 Oct 13 10:11 Compile
drwxr-xr-x    2 root     root          4096 Oct 13 10:11 RestlerLogs
-rw-r--r--    1 root     root          4698 Oct 13 10:10 swagger.json
```

## Test (SmokeTest)

Ja podem executar la comanda de test per validar que tots els endpoints definits funcionen correctament. Per fer-ho, executarem la següent comanda dins del contenidor restler

```bash
dotnet /RESTler/restler/Restler.dll test --grammar_file ./Compile/grammar.py --dictionary_file ./Compile/dict.json --settings ./Compile/engine_settings.json --host host.docker.internal --target_port 4000 --no_ssl
```

Ens hauria de tornar la següent sortida:

```text
Starting task Test...
Using python: 'python3' (Python 3.9.5)
Request coverage (successful / total): 1 / 5
No bugs were found.
Task Test succeeded.
Collecting logs...
```

En aquest cas, podem veure que no ha trobat cap error 500 en els endpoints que ha provat, això si, només ha pogut validar satisfactoriament un endpoint.
Podem consultar les peticions que ha executat en els següents logs:

```bash
cat ./Test/RestlerResults/experiment<GUID>/logs/main.txt
cat ./Test/RestlerResults/experiment<GUID>/logs/speccov.json
```

Si us fixeu en els logs, podeu veure que això es degut a que l'eina està executant les peticions de `GET` abans de les de `POST`, a més, no té en compte que el parametre `Rol` es un enum i `email` ha de tenir un format especific. Aquestes limitacions són en part perque l'especificació de l'API està incompleta, i no reflexa aquestes limitacions ni dependencies.

La documentació oficial inclou una guia per intentar millorar el coverage de l'eina (<https://github.com/microsoft/restler-fuzzer/blob/main/docs/user-guide/ImprovingCoverage.md>)

A continuació, introduirem un error 500 a la nostra aplicació. Modificarem el fitxer `./Controllers/UsersControllers.cs`, i substituirem la funció `GetAll` per la següent:

```cs
    [HttpGet]
    public IActionResult GetAll()
    {
        var users = _userService.GetAll();
        //Force error throw
        throw new ApplicationException("Oopss, Something gone wrong!");
        return Ok(users);
    }
```

Seguidament, tornarem a generar la imatge Docker i recrearem el contenidor de l'API.

```bash
docker build -t crud_api:latest .
docker rm -f dotnet6-api && docker run -d --name dotnet6-api -p 4000:4000 crud_api:latest
```

Tornem a executar el `smoke test` des de dins del contenidor del restler

```bash
dotnet /RESTler/restler/Restler.dll test --grammar_file ./Compile/grammar.py --dictionary_file ./Compile/dict.json --settings ./Compile/engine_settings.json --host host.docker.internal --target_port 4000 --no_ssl
```

En aquesta execució hauriem de veure que ha detectat l'error 500

```txt
Starting task Test...
Using python: 'python3' (Python 3.9.5)
Request coverage (successful / total): 0 / 5
Bugs were found!
Bug buckets:

main_driver_500: 1
Task Test succeeded.
Collecting logs...
```

Consultem els logs per veure que efectivament ha fallat la query `GET /Users`

```txt
/api/Test/RestlerResults/experiment<GUID>/bug_buckets # cat bug_buckets.txt
main_driver_500: 1
Total Buckets: 1
-------------
main_driver_500 - Bug was reproduced - main_driver_500_1.replay.txt
Hash: main_driver_500_baf92d88d29aa014a313ed7df504ee870903bbcf
GET /Users HTTP/1.1\r\nAccept: application/json\r\nHost: host.docker.internal\r\nauthentication_token_tag\r\n
```

## Fuzzing

Aquesta eina inclou dos modes diferents de Fuzzing:

- **Fuzz-Lean**: Executa una única prova per cada endpoint i metode, fent servir un conjunt predefinit de verificadors per veure si es poden detectar bugs ràpidament
- **Fuzz**: En aquest mode, l'eina executarà tests de Fuzzing durant més temps, amb l'objectiu de trobar errors relacionats amb l'exhauriment de recursos.

Per aquest laboratori, executarem només l'eina de Fuzz-Lean com exemple.

```bash
dotnet /RESTler/restler/Restler.dll fuzz-lean --grammar_file ./Compile/grammar.py --dictionary_file ./Compile/dict.json --settings ./Compile/engine_set
tings.json --host host.docker.internal --target_port 4000 --no_ssl
```

```text
Starting task FuzzLean...
Using python: 'python3' (Python 3.9.5)
Request coverage (successful / total): 0 / 5
Bugs were found!
Bug buckets:

main_driver_500: 1
Task FuzzLean succeeded.
Collecting logs...
```

En aquest cas, podem veure el resum dels errors trobats al següent log:

```cat
cat /test/FuzzLean/ResponseBuckets/runSummary.json
```

i una mostra del les peticions executades amb les seves respostes al fitxer `errorBuckets.json`

```cat
cat /test/FuzzLean/ResponseBuckets/errorBuckets.json
```
