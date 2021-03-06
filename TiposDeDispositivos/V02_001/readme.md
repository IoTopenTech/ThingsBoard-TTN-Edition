# Tipo de dispositivo V02_001
## Descripción general
El tipo de dispositivo V02_001 tiene una finalidad eminentemente didáctica para mostrar las posibilidades de la plataforma my IoT open Tech.

Se trata de un dispositivo LoRaWAN con las siguientes características:

* Envía un mensaje con el estado de un sensor de efecto hall (sensor de puerta abierta) cada vez que éste cambia de estado.
* Envía periódicamente una señal heartbeat indicando el estado de la puerta, el nivel de batería y el estado del LED.
* Sólo admite OTAA.
* Puede ser pre-aprovisionado (un proveedor se encarga de dar de alta el dispositivo en su cuenta de The Things Network antes de vendérselo al cliente), reclamado (el cliente sólo tiene que introducir el identificador del dispositivo para empezar a utilizarlo en My IoT open Tech, pero en The Things Network el dispositivo sigue siendo propiedad del proveedor), y tomado en posesión (una vez reclamado, el usuario puede tomar la posesión completa del dispositivo vinculándolo a su propia cuenta de The Things Network).
* El identificador del dispositivo puede leerse como código morse a través del LED.
* Puede resetearse para volver al estado pre-reclamado, por si el usuario quisiera revincularlo a otra aplicación de The Things Network, o pasar la propiedad a un tercero.

## Hardware
El tipo de dispositivo V02_001 está construído a partir de la plataforma del nodo V2.2 de la comunidad The Things Network Madrid:

https://github.com/IoTopenTech/Nodo_TTN_MAD_V2_2/wiki

Está basado en un Arduino Pro mini a 8MHz y 3.3V, y un RFM95W.

En el tipo de nodo V02_001 sólo se han incorporado las extensiones Hall y LED.

## Software
El código esta disponible en la carpeta software y para programarlo en el dispositivo se requiere un conversor USB / Serial con lógica de 3.3V.

Al arrancar, el dispositivo consulta si el botón está pulsado y, en caso afirmativo, empieza a hacer parpadear el LED cada segundo mientras espera a que el usuario suelte el botón.

Si el usuario suelta el botón antes de 5 segundos, empezará a emitir en código morse mediante el LED la clave para reclamar el dispositivo. Con otra pulsación del botón se saldrá del modo morse, y el dispositivo empezará a funcionar normalmente.

Si el usuario tarda más de 5 segundos en soltar el botón, el dispositivo se reseteará volviendo al estado pre-reclamado, es decir, regresará a las credenciales OTAA de The Things Network que tenía originalmente (antes de que el usuario final lo reclamase y/o tomase posesión de él).

Si el botón no está pulsado al arrancar, el dispositivo entra en el modo de funcionamiento normal, que consiste en estar dormido (para minimizar el consumo de energía), y sólo despertar cuando se produzca un cambio de estado del sensor de efecto hall o para enviar señales de tipo heartbeat periódicas (por defecto, aproximadamente cada minuto).

## Carga de pago para uplinks

El dispositivo utiliza el formato de carga de pago Cayenne LPP (por ejemplo:  01020160020000030000060100) , con los siguientes canales:

Canal | Tipo | Valor
----- | ---- | -----
0x01 | Entrada analógica [0x02] | Tensión de la batería en cV (centivoltios)
0x02 | Entrada digital [0x00] | Estado del sensor hall: 0 cerrado y 1 abierto
0x03  |Entrada digital [0x00] | Estado del LED: 0 apagado y 1 encendido.  Obsérvese que se distingue el canal para indicar el estado del LED (éste) del canal para cambiar el estado del LED (0x06; ver siguiente)
0x06 | Salida digital [0x01] | Canal para cambiar el estado del LED. myDevices necesita que sea de tipo 0x01 para mostrar el botón, pero el downlink que envía es de tipo 0x00?

## Carga de pago para downlinks

El dispositivo sólo atiende los downlinks que lleguen por el puerto 99, y usa un formato similar al de Cayenne LPP:

Canal | Tipo | Valor
----- | ---- | -----
0x06  |Entrada digital [0x00] | Cambiar el estado del LED: 100 (0x64) encender; cualquier otro valor apagar
0x07|Entrada digital [0x00] | Cambiar el periodo de envío de heartbeats (expresado en minutos; máximo 60)
0x46 | 0x46 (no estándar LPP) | Cambio de credenciales OTAA

El cambio de credenciales implica el envío de 3 downlinks, uno para cada credencial (DEV EUI, APP EUI y APP KEY). No pueden enviarse en un sólo downlink porque se superaría la longitud de carga de pago de LoRaWAN para los spread factors más lentos.

Cada downlink comenzará con 0x46 0x46 (caracteres "FF" ASCII expresados en hexadecimal).

A continuación se utilizará el siguiente par de bytes para indicar el tipo de credencial:

Byte 3 y 4 | Tipo de credencial
---------- | ------------------
0x30 0x31 ("01" ASCII2HEX) | DEV EUI
0x30 0x32 ("02" ASCII2HEX) | APP EUI
0x30 0x33 ("03" ASCII2HEX) | APP KEY

A continuación se utilizará un byte para indicar el tipo de activación (O de OTAA o A de ABP); en este caso sólo se admite OTAA, por lo que el 5º byte será 0x4F.

Por último se incluirá la credencial correspondiente en el mismo formato que la presenta la consola de The Things Network por defecto (sin cambiar el endianess).

# Parámetos de configuración
El tipo de dispositivo V02_001 ofrece los siguientes parámetros de configuración en la interfaz de My IoT open Tech:

* Número de minutos entre heartbeats.
* DevEUI, para poder recibir telemetrías a través de un dispositivo de control (tipo SYSTEM).
* Coordenadas para mostrarlo sobre la imagen de un activo de tipo IMAGE01 (corresponden al porcentaje respecto a la esquina superior izquierda; sólo se admiten valores positivos).
* Alarmas: En todas las alarmas el usuario puede elegir por qué vía desea que le sean notificadas (email, IFTTT, Telegram...)
  * Cambio de estado: Se puede activar al abrir el sensor hall o al cerrarlo.
  * Nivel bajo de batería: Permite configurar un umbral mínimo.
  * Inactividad: Permite establecer el número de segundos que debe transcurrir sin actividad por parte del dispositivo para que se genere la alarma (obviamente, convendrá que sea un valor mayor que el del periodo de hearbeat).
  
## Notificaciones de alarmas
### Cambio de estado
Las notificaciones para la alarma de cambio de estado tienen la siguiente estructura:

* email y Telegram: El dispositivo [nombre del dispositivo] ha generado una alarma de tipo Puerta abierta/cerrada. La tensión actual de la batería es [tensión] V, y el umbral de alarma es [umbral de alarma de nivel bajo de batería] V.
* IFTTT: {"value1":"'[nombre del dispositivo]'","value2":"'[estado del sensor de efecto hall]'","value3":"'[tensión de la batería]'"}

### Nivel bajo de batería
Las notificaciones para la alarma de nivel bajo de batería tienen la siguiente estructura:

* email y Telegram: El dispositivo [nombre del dispositivo] ha generado una alarma de tipo Nivel bajo de batería. La tensión actual de la batería es [tensión] V, y el umbral de alarma es [umbral de alarma de nivel bajo de batería] V.
* IFTTT: {"value1":"'[nombre del dispositivo]'","value2":"'[estado del sensor de efecto hall]'","value3":"'[tensión de la batería]'"}

### Inactividad
Las notificaciones para la alarma de inactividad tienen la siguiente estructura:

* email y Telegram: El dispositivo [nombre del dispositivo] ha generado una alarma de inactividad.
* IFTTT: {"value1":"'[nombre del dispositivo]'","value2":"'[estado del sensor de efecto hall]'","value3":"'[tensión de la batería]'"}

# Formato de pre-aprovisionamiento
Para preaprovisionar dispositivos de tipo V02_001 se requiere un archivo CSV que use como separador el signo ; (punto y coma) y contenga los siguientes campos:
* name: Nombre del dispositivo precedido del carácter P; por ejemplo: P0000000100000001
* type: Tipo de dispositivo; por ejemplo V02_001
* nombreOriginal: Nombre original del dispositivo para recuperarlo si el usuario lo resetease; coincidirá con el name (P0000000100000001)
* tipoOriginal: Tipo original del dispositivo para recuperarlo si el usuario lo resetease; coincidirá con el type (V02_001)
* claimingData: Objeto en formato JSON con la clave de reclamación del dispositivo; por ejemplo: {"secretKey": "ABCDEFGHIJKLMNOP", "expirationTime": "1640995200000"}
* claimingDataOriginal: Copia del parámetro anterior porque el anterior será borrado si el usuario reclama el dispositivo, y necesitamos un modo de recuperar esa información si el usuario resetease el dispositivo.
* apropiable: Atributo en el que se indica si el usuario podría tomar posesión del dispositivo mediante downlinks; en este caso el valor será true.
* apropiado: Atributo en el que se indica si el usuario ha tomado posesión del dispositivo; contendrá el valor false
* __cs_url: Si el dispositivo está pre-aprovisionado en ChirpStack, aquí indicaremos el URL.
* __cs_token: Si el dispositivo está pre-aprovisionado en ChirpStack, aquí indicaremos el JWT autorizado en la API.
* admiteABP: Contendrá el valor false porque el tipo de nodo V02_001 no admite ABP
* __heartbeat: Este parámetro corresponde al periodo de envío de heartbeats y contendrá el valor 1 (que es el predeterminado para el tipo V02_001).

# TTNMAD_V02_001_delegate

```xml
<myIoT>
   <delegacion nombre="Bat" label="Permitir ver la tensión de la batería" />
   <delegacion nombre="Hall" label="Permitir ver el estado del sensor Hall" />
   <delegacion nombre="LED" label="Permitir ver el estado del LED" />
   <delegacion nombre="CambiarLED" label="Permitir cambiar el estado del LED" />
   <delegacion nombre="Heartbeat" label="Permitir cambiar el periodo de envío de heartbeats" />
</myIoT>
```

# TTNMAD_V02_001_config

```xml
<myIoT>
   <panel titulo="Configuración general" resumen="Configurar atributos de la entidad" nombreFormulario="General" labelBotonSubmit="Configurar">
      <item tipo="coordenadas" />
      <item tipo="chirpstack" />
      <item tipo="alarma" nombreAlarma="nivelDeBateria" telemetria="Bat" labelAlarma="nivel bajo de batería" tipoAlarma="umbralMinimo" labelAuxAlarma="(V)" histeresis="min">
         <umbralMinimo size="10" min="0" max="4" step="0.1" />
      </item>
      <item tipo="alarma" nombreAlarma="cambioDeEstado" telemetria="Hall" labelAlarma="estado del sensor Hall" tipoAlarma="opciones" labelAuxAlarma="Disparar al" opciones="Abrir/Cerrar" />
      <item tipo="alarma" nombreAlarma="inactividad" />
   </panel>
   <panel tipo="devEUI" />
   <panel titulo="Heartbeat" resumen="Configurar periodo de envío de heartbeat" nombreFormulario="Heartbeat" ultimoDownlink="Heartbeat">
      <item tipo="atributoCompartido" nombreAtributo="Heartbeat" labelAtributo="Número de minutos entre heartbeats." tipoAtributo="numero">
         <atributosHTML size="10" step="1" min="0" max="59" />
      </item>
   </panel>
   <panel titulo="Credenciales LoRaWAN" resumen="Configurar las credenciales LoRaWAN del dispositivo" nombreFormulario="LoRaWAN" ultimoDownlink="tomarPosesion">
      <md-input-container style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Método de activación</label>
         <md-select ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion" required="required">
            <md-option ng-if="vm.attributes.hasOwnProperty('admiteABP') &amp;&amp; vm.attributes.admiteABP==true" value="A" ng-selected="">ABP</md-option>
            <md-option value="O">OTAA</md-option>
         </md-select>
      </md-input-container>
      <md-input-container style="margin: 0px; margin-top: 10px;" ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='A'" class="md-block">
         <label>Device Address (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{8}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam1" required="required" />
      </md-input-container>
      <md-input-container ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='A'" style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Network Session Key (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{32}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam2" required="required" />
      </md-input-container>
      <md-input-container ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='A'" style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Application Session Key (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{32}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam3" required="required" />
      </md-input-container>
      <md-input-container ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='O'" style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Device EUI (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{16}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam1" required="required" />
      </md-input-container>
      <md-input-container ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='O'" style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Application EUI (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{16}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam2" required="required" />
      </md-input-container>
      <md-input-container ng-if="vm.configuracion.___atributosCompartidos.tomarPosesionMetodoActivacion=='O'" style="margin: 0px; margin-top: 10px;" class="md-block">
         <label>Application Key (msb)</label>
         <input type="text" pattern="[0-9a-fA-F]{32}" autocomplete="off" ng-model="vm.configuracion.___atributosCompartidos.tomarPosesionParam3" required="required" />
      </md-input-container>
   </panel>
</myIoT>
```

