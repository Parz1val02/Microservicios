## Que son xd?
- Conjunto de componentes pequenos y autonomos que colaboran entre si para llevar a cabo una gran tarea, como una aplicacion o sistema, y brindar una solucion
- Evitan depender de saber exactamente la ip y puerto de un servicio, a diferencia de los web services. Se evitan problemas al no tener que lidiar con la dinamicidad de la asignacion de ips para los web services
#### Despliegue monolitico vs despliegue con microservicios
- En  un monolito se corre solo una aplicacion que gestiona todas las funciones que se realizan. Los elementos dependen de otros elementos para su despliegue
- Arquitecura desacoplada con microservicios
	- Cada servicio posee una funcion unica que debe realizar
	- Independientes entre si, no dependen de otros elementos para su despliegue
	- Registro y autodescubrimiento: ***servidor discovery*** (componente vital, diferencia una arquitectura de microservicios de una simple arquitectura que posee multiples web services) 
	- Autoescalado y agilidad de cada microservicio
	- Confiabilidad y tolerancia a fallos
	- Balanceo de cargas
	- Configuracion centralizada (las configuraciones para los microservicios de toda la aplicacion se pueden encontrar en una repo de git por ejemplo)
	- Desventajas: 
		- Retardo al consumir el servicio: al ser web services, el retardo en la respuesta es inevitable. Este retardo aumenta si los webservices se encuentran en diferentes instancias en diferentes regiones. Lo que no ocurre en un monolito, dado que pasar de la ejecucion de codigo en una funcion a otra normalmente solo toma un ciclo de reloj del servidor 
##### RESTful
- Considerar servicio web como RESTful, seguir el protocolo RFC 2616 para realizar un CRUD:
	- `GET: read`
	- `POST: create`
	- `PUT: update`
	- `DELETE: delete`
> Caso contrario es un servicio web REST
> Uso de entity, repository y RestController para brindar un servicio web REST
> Uso de entity, dao y RestTemplate para conumir un servicio web REST

##### RestTemplate vs. Feign
- *RestTemplate*: libreria nativa de Spring para consumir web services. Se indica la ruta exacta de la ubicacion del web service (ip:puerto): `http://localhost:8080/teleco`
	- Es posible migrar RestTemplate a usar name en vez de una url con un builder
- *Feign*: libreria creada por Netflix que posee una forma mas desacoplada de consumir web services utilizando name en vez de url
	- El nombre debe coincidir con el spring application name del servicio que se quiere consumir
 ---
## Componentes de una arquitectura de microservicios
- Servidor de registro o discovery: proporciona independencia del conocimiento previo del socket de cada servicio. Cada servicio se registra con este servidor para conocer su ubicacion en todo momento (ej. Eureka, Apache Zookeeper)
- Servidor perimetral: gateway que recibe todas las solicitudes externas y las redirecciona a los servicios correspondientes (ej. Zuul (anterior a la version 2.4), Spring Cloud Gateway (posterior a la version 2.4))
- Sistema de tolerancia a fallos: evita fallos en cascada (Ej. Hystrix (anterior a la version 2.4), (posterior a la version 2.4) Resilience4j)
- Balanceador: reparte carga entre servicios (ej. Ribbon)
- Servidor de configuracion: proporciona configuraciones centralizadas al sistema (ej. Spring Cloud Config)
- Gestion de logs: sistema pa gestionar logs y fallos por servicio (ej. Turbine)

### Eureka
- Servidor discovery para una arquitectura de microservicios
##### Eureka server
- Es importante levantar los servicios en orden. Primero el eureka, luego el gateway (se registra con eureka), finalmente los servicios genericos
- Puerto por defecto: 8761
	`server.port=8761`
- Se debe evitar el registro de eureka consigo mismo (esta funcionalidad puede ser util cuando se tienen multiples servidores discovery y se desea hacer balanceo de carga entre ellos)
	`eureka.client.register-with-eureka=false`
- No obtener la ubicacion de cada servicio de cache, es decir, cuando se vuelve a activar un servicio previamente registrado, Eureka puede obtener su ubicacion de la cache que ha guardado u obtener con discovery
	`eureka.client.fetch-registry=false`
##### Clientes Eureka
- Cada servicio se registra al prenderse como un eureka client y registra su ip y puerto. Al apagarse, no se desregistra
	`eureka.client.service-url.defaultzone=http://localhost:8761/eureka`
- Cada cliente de eureka manda un heartbeat cada 30 segundos para anunciar su correcto funcionamiento. Cuando Eureka deja de recibir 3 heartbeats seguidos de algun servicio, asume que se ha caido el servicio
- Nombre para poder registrarlo junto con su socket. Cada servicio debe contar con un nombre que el servidor Eureka pueda guardar, incluido el mismo Eureka
	`spring.application.name=nombre`
- Permite un balanceo de carga basico, round robin. Al tener mas de un servicio registrado con el mismo nombre, se ejecuta este balanceo de carga. 
	- Identificador unico para que Eureka puede diferenciar a servicios con el mismo nombre
		`eureka.instance.instance-id=${spring.cloud.client.hostname}:${spring.application.name}:${random.int}`
	- Al caerse uno de los servicios, se debe esperar alrededor de minuto y medio para que Eureka pueda darse cuenta y el balanceo de carga no genere errores
- El ecosistema no se ve limitado a solo el uso de java. La ventaja de la arquitectura de microservicios es la indepdencia en cuanto al lenguaje de programacion usado

### Spring Cloud Gateway
- Un API gateway provee una puerta de enlace centralizada entre el exterior y los servicios
- Permite tambien enrutamiento dinamico a los microservicios, balanceo de carga, filtros para gestionar cabeceras, payloads, etc., con el objetivo proveer seguridad (como un firewall, rechazo de requests que cumplan ciertas condiciones)
- Es importante registrar el gateway con el servidor de discovery
- Configuracion de servicio desde el API gateway para una ruta (routes funciona como una lista, por eso se usa un indice):
	- `spring.cloud.gateway.routes[0].id=nombre`: nombre del servicio
	- `spring.cloud.gateway.routes[0].uri=lb://nombre`: ruta al servicio (lb stand for load balance)
	- `spring.cloud.gateway.routes[0].predicates[0]=Path=/api/**`: predicates cambia la ruta del servicio al agregar **/api** al inicio. \*\* simboliza el resto de la ruta ya predefinida para el servicio (predicates tambien funciona como una lista)    
	- `spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1`: Saca el primer prefijo desde la raiz para usarlo como ruta final. En el ejemplo, el gateway saca **/api** de la ruta que le llega al gateway y usa el resto como ruta final para buscar el servicio (filters tambien funciona como una lista)

### Resilience4j
- En la arquitectura de microservicios, se puede llegar a generar una situacion en la cual se cae un servicio del que dependen muchos otros servicios y se genera un fallo en cascada. Para casos como este, se implementa un cortocircuito. Este es un patron de diseno que. al superarse un umbral de requests fallidos hacia un servicio, el circuito se abre y la siguiente solicitud que llega hacia el servicio ya no se realiza. Simplemente se avisa que este servicio en cuestion se encuentra caido. Un flujo alternatico que responde sin necesidad de preguntar
- tldr; el cortocircuito evita que spring realice solicitudes innecesarias hacia un servicio caido
- El cortocircuito (CircuitBreaker) se implementa en el servicio que consume
##### Estados del circuito
- Cerrado: el microservicio funciona nomal (estado por defecto)
- Cuando se supera el umbral de falla tras 100 solicitudes:
	- Abierto: Ya no se realizan las solicitudes al microservicio caido. Se envian respuestas alternas
	- Semi-abierto: Despues de haber estado 60s (tiempo por defecto) en el estado abierto, automaticamente el circuito pasa a este estado. Se realizan 10 solicitudes y, en caso no se supere el umbral de falla, vuelve al estado cerrado. Caso contrario, vuelve al estado abierto
- *Parametros de configuracion por defecto*:
	- slidingWindowSize: 100
		- Se realizan 100 Peticiones a un microservicio y se registran un porcentaje de fallas. Si esto supera el umbral, se abre el circuito.
	- failureRateThreshold: 50 (porcentaje)
		- Es el umbral. Si de las 100 solicitudes, fallan 50, se abre el circuito.
	- waitDurationInOpenState: 60000 ms 
		- En este tiempo no se realiza más solicitudes a un microservicio pues está abierto.
	- permittedNumberOfCallsInHalfOpenState: 10
		- En semiabierto se van a realizar 10 solicitudes para ver si regresa a cerrado o a abierto.
	- slowCallRateThreshold: 100
		- Si se tienen 100 llamadas lentas entonces entra a estado abierto. 
		- ¿Qué es una “llamada lenta”? → slowCallDurationThreshold
	- slowCallDurationThreshold: 60000 ms
		- Si tarda más de 1 minuto se considera una llamada lenta.