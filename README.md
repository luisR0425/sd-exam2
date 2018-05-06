# sd-exam2 <br />

### Universidad Icesi <br />

**Nombre:** Luis Fernando Rosales Cadena <br />
**Código:** A00315320 <br />
**URL repositorio:** https://github.com/luisR0425/sd-exam2 <br />

Se debe automatizar el despliegue de cuatro servicios, el fluentd para recopilación de logs, el elasticsearch como un motor de búsqueda, kibana para la visualización de los logs y estadísticas y un servicio web con 4 replicas y un balanceador de cargas. Para el servicio web se usó un contenedor llamado whoami que nos permitía visualizar el hostname de la máquina en la que corría en una página web. Lo que se hace es simplemente correr un script en go. <br />

``` go
package main

import (
  "os"
  "fmt"
  "net/http"
  "log"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    fmt.Fprintf(os.Stdout, "Listening on :%s\n", port)
    hostname, _ := os.Hostname()
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(os.Stdout, "I'm %s\n", hostname)
 	fmt.Fprintf(w, "I'm %s\n", hostname)
    })


    log.Fatal(http.ListenAndServe(":" + port, nil))
}

```

Antes de iniciar los servicios se debe crear un closter con docker swarm, para ello se usa el siguiente comando en la máquina que será el nodo manager. <br />

```
docker swarm init --advertise-addr 192.168.1.22
```

Y para el nodo worker en donde se desplegarán el servicio web con las 4 replicas se usa <br /

```
docker swarm join --token <token del nodo worker> 192.168.1.22:2377
```

Después se crean los archivos para aprovisionar la infraestructura para este examen se usaron los siguientes:

Dockerfile para aprovisionar fluentd.

```
# fluentd/Dockerfile
FROM fluent/fluentd:v0.12-debian
RUN echo "fluentd"
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.2"]
```

Docker-compose para crear toda la infraestructura.

```yml
version: "3"

services:

  whoami:
    image: tutum/hello-world
    networks:
      - webnet #Para la red
    ports:
      - "192.168.1.13:80:80" 
    logging:
      driver: "fluentd" # Logging Driver 
      options:
        fluentd-address: 192.168.1.22:24224  #Especifique una dirección de socket para conectarse al daemon Fluentd
        tag: docker.{{.Name}}    # TAG Es decir el hostname de cada contenedor
    deploy: #Para hacer el aprovisionamiento usando docker swarm
      restart_policy: #Configura cómo reiniciar los contenedores cuando se cierran
           condition: on-failure
           delay: 20s
           max_attempts: 3
           window: 120s
      mode: replicated
      replicas: 4 #indica las replicas que se quieren de ese contenedor
      resources:
        limits:
          cpus: "0.1" #restricción de cpu de 10%
          memory: 20M # Restricción de memoria de 20M
      placement:
        constraints:
          - node.role == worker # Para indicarle que es un nodo worker
      update_config:
        delay: 2s

  vizualizer:
      image: dockersamples/visualizer
      volumes:
         - /var/run/docker.sock:/var/run/docker.sock
      ports:
        - "8080:8080"
      networks:
        - webnet
      logging:
        driver: "fluentd" # Logging Driver 
        options:
         tag: visualizer   #TAG
      deploy:
          restart_policy:
             condition: on-failure
             delay: 20s
             max_attempts: 3
             window: 120s
          mode: replicated 
          replicas: 1
          update_config:
            delay: 2s
          placement:
             constraints: [node.role == manager]

  fluentd:
    image: fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224/tcp" # puertos que usa fluentd
      - "24224:24224/udp"
    networks:
      - webnet
    deploy:
      restart_policy:
           condition: on-failure
           delay: 20s
           max_attempts: 3
           window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s

  elasticsearch:
    image: elasticsearch
    ports:
      - "9200:9200"
    networks:
      - webnet
    environment: #variables de ambiente
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    logging:
        driver: "json-file" # Logging Driver 
        options:
          max-size: 10M
          max-file: 1  
    deploy:
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s
      resources:
        limits:
          memory: 1000M
    volumes:
      - my-vol:/usr/share/elasticsearch/data 

  kibana:
    image: kibana
    ports:
      - "5601:5601" 
    networks:
      - webnet
    logging:
        driver: "json-file"
        options:
           max-size: "10M"
           max-file: "1"
    deploy:
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s

volumes:
  my-vol: #volumen interno para elasticsearch

networks:
webnet: #crea la red
```

En el nodo worker se ejecutan el siguiente comando para crear la infraestructura.

```
docker stack deploy -c Dockerfile-compose.yml parcial2
```

Con ```Docker service ls``` se puede ver que todos los servicios se subieron correctamente 

![][1] 

Con ```Docker ps``` en el lado del nodo worker se observan los cuatro contenedores que en realidad son replicas de un mismo servicio web.

![][2]

Captura de servicio web.

![][4]

Con vizualizer podemos visualizar mejor los contenedores desde una página web.

![][3]

Por último las pruebas de funcionamiento de los servicios.

![][5]

Se puede ver los logs en formato json.

![][6]

### Diagrama de despliegue

![](https://i.imgur.com/DXCVaJI.png)

### Dificultades encontradas <br />
Debido a un proxy interno de docker que hay cuando se usa con macOS no se podía usar de nodo manager, por lo tanto, se usó una máquina virtual en vagrant con centos que sería el nodo manager y el pc mac sería el worker. Se debía asignar más memoria para el despliegue de efk por lo que se usó el comando ```sysctl -w vm.max_map_count=262144```.

[1]: images/captura1.png
[2]: images/captura2.png
[3]: images/captura3.png
[4]: images/captura4.png
[5]: images/captura5.png
[6]: images/captura6.png
