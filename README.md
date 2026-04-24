### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Parte 1 - Escalabilidad vertical

### Creación de VM

![Creacion VM.png](images/Creacion%20VM.png)

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?

![Componentes creados.png](images/Componentes%20creados.png)

2. ¿Brevemente describa para qué sirve cada recurso?

Virtual Machine (VERTICAL-SCALABILITY): Es el servidor principal (cómputo). Proporciona la CPU, RAM y sistema operativo para ejecutar tu FibonacciApp.

Public IP Address (VERTICAL-SCALABILITY-ip): La dirección única en internet para que puedas acceder a la VM desde tu casa vía SSH o HTTP.

Virtual Network (VERTICAL-SCALABILITY-vnet): Define el segmento de red privada donde vive tu VM, permitiendo el aislamiento y la comunicación interna.

Network Interface (vertical-scalability4): Es la tarjeta de red virtual que conecta la VM con la red (VNet) y le asigna las IPs.

Network Security Group (VERTICAL-SCALABILITY-nsg): Actúa como un firewall. Define las reglas de seguridad para permitir o denegar tráfico (como el puerto 8080 o el 22 de SSH).

Resource Group (SCALABILITY_LAB): No es un recurso técnico per se, sino un contenedor lógico para agrupar y gestionar todos los elementos anteriores como una sola unidad.

3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?

La aplicación se cae porque se está ejecutando como proceso de la conexión, al terminar la conexión, se terminan todos los procesos que se están ejecutando. Por defecto, Azure bloquea todo el tráfico por seguridad. El puerto del servicio debe abrirse manualmente con la regla de seguridad para que las peticiones lleguen a la aplicación.

4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.

Los tiempos de cada una de las pruebas estuvieron entre 9 segundos para la prueba más simple, es decir, 1.000.000 y 11 segundos para la prueba de 1.090.000.

La serie de Fibonacci calculada de forma recursiva simple tiene una complejidad muy grande, por lo que po cada número se va a demorar más en calcularse.

5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.

![CPU Endpoint.png](images/CPU%20Endpoint.png)

Por cada número adicional, el esfuerzo del CPU se duplica.

6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.

![Resumen Postman pequeña.png](images/Resumen%20Postman%20peque%C3%B1a.png)

Los fallos se deben a la concurrencia que se simuló. En cuanto al tiempo, también se llega a elevar mucho, pasando de 9 segundos a 17 o 18 segundos.

7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?

B1ls: Es la más pequeña de Azure. Está diseñada para servidores web mínimos o micro-contenedores. No tiene mucha capacidad de "acumular" créditos de CPU.

B2ms: Permite ráfagas (Bursting). Cuando se lanza Newman, la VM "gasta" créditos para dar potencia extra momentánea. El B2ms tiene mucho más margen para manejar la concurrencia que el B1ls.

8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?

A corto plazo sí, porque reduce el tiempo de cálculo. Sin embargo, tiene un límite (techo de hardware) y es costosa. Fibonacci pasa a tener tiempos de 6 segundos.

9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?

Downtime: El servicio estará fuera de línea durante varios minutos.
Costo: El precio por hora aumenta significativamente.
Persistencia: Si no hay IP estática, la dirección pública podría cambiar.

10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?

![Resumen Postman grande.png](images/Resumen%20Postman%20grande.png)

Si, ya que al haber mayor cantidad de núcleos y mayor potencia, la función se vuelve más rápida de calcular.

11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

Al subir a 4 ejecuciones paralelas en una VM con pocos núcleos, el rendimiento empeoró porcentualmente por petición. Los procesos compiten por el tiempo de CPU, haciendo que el sistema se sature. La escalabilidad vertical tiene límites cuando la carga es concurrente, y este es un ejemplo.


### Parte 2 - Escalabilidad horizontal

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?

Azure Load Balancer: Trabaja en la Capa 4 (TCP/UDP). Es ideal para tráfico de red puro.

Application Gateway: Trabaja en la Capa 7 (HTTP/HTTPS). Entiende URLs y cookies (ideal para aplicaciones web).

El Stock Keeping Unit define las capacidades y el precio del recurso. Por último, se necesita una IP pública porque es el punto único de entrada para los usuarios. En lugar de que el usuario intente conectarse a la IP de la VM1 o VM2, se conecta a la IP del Balanceador, y este decide a qué VM enviarlo. Sin IP pública, el balanceador sería invisible desde internet.

* ¿Cuál es el propósito del *Backend Pool*?

Contiene las direcciones IP de las VMs que están listas para recibir tráfico.

* ¿Cuál es el propósito del *Health Probe*?

Revisa constantemente si las VMs están vivas. Si una VM deja de responder, el balanceador deja de enviarle tráfico automáticamente.

* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.

Define por qué puerto entra el tráfico (ej. 80) y a qué puerto del backend debe enviarse (ej. 8080).

Tipos: None (petición balanceada al azar), Client IP (misma IP, misma VM), o Client IP and Protocol.

Importancia: Es vital si la aplicación guarda datos en la memoria local de la VM.

Efecto en escalabilidad: Si los usuarios son obligados a quedarse en una VM específica, se puede sobrecargar una máquina mientras las otras están libres, rompiendo el equilibrio del sistema.

* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?

Virtual Network (VNet): Tu parcela privada en la nube de Azure. Es una red lógica aislada.

Subnet: Una subdivisión de la VNet. Sirve para organizar recursos (ej. una subred para servidores web y otra para bases de datos).

Address Space: El rango total de IPs de la VNet.

Address Range: El rango específico asignado a una Subnet.

* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?

Son centros de datos físicos separados dentro de una misma región. Se seleccionan 3 zonas diferentes para tema de disponibilidad, es decir, si la máquina 1 falla, la máquina 2 se levanta y sigue la ejecución. Que sea zone-redundant significa que la misma IP está en las diferentes zonas para que pueda ser dirigido a ellas.

* ¿Cuál es el propósito del *Network Security Group*?

Su propósito es la seguridad perimetral. Es una tabla de reglas que permite o deniega el paso.

* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.




