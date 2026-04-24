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

1. Downtime: El servicio estará fuera de línea durante varios minutos.
2. Costo: El precio por hora aumenta significativamente.
3. Persistencia: Si no hay IP estática, la dirección pública podría cambiar.

10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?

![Resumen Postman grande.png](images/Resumen%20Postman%20grande.png)

Si, ya que al haber mayor cantidad de núcleos y mayor potencia, la función se vuelve más rápida de calcular.

11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

Al subir a 4 ejecuciones paralelas en una VM con pocos núcleos, el rendimiento empeoró porcentualmente por petición. Los procesos compiten por el tiempo de CPU, haciendo que el sistema se sature. La escalabilidad vertical tiene límites cuando la carga es concurrente, y este es un ejemplo.


### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
* ¿Cuál es el propósito del *Backend Pool*?
* ¿Cuál es el propósito del *Health Probe*?
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
* ¿Cuál es el propósito del *Network Security Group*?
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.




