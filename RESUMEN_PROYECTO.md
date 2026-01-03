# Resumen de Proyecto: Adaptador GPIO PCI (UEFI-Custom)

## Objetivo
- Tarjeta target PCI 5V con puerto GPIO de 4 u 8 bits usando 3x GAL22V10 (expandible).

## Base de hardware y firmware
- Motherboard: Gigabyte GA-G41M-ES2 (PCI 5V).
- BIOS: Coreboot + EDK2 (UEFI) con control manual de rangos de IO/memoria.
- Logica: GAL22V10 programadas en CUPL (TL866II); elegidas por tolerancia 5V y comportamiento determinista.

## Estrategia de implementacion
- Fase actual: Adaptador fisico PCI a protoboard/cable flex y verificacion de integridad de senal.
- Ruta B (Direct IO): direccion de IO fija (ej. 0x300) reconocida por BIOS; se omite Plug & Play inicialmente.
- Software futuro: driver UEFI en C (EDK2) definiendo EFI_SYSTEM_TABLE, UEFI_main y protocolos de IO PCI.

## Division logica tentativo
- GAL1: Decodificacion de direccion/comandos y chip-select.
- GAL2: Maquina de estados PCI (FRAME#, DEVSEL#, TRDY#).
- GAL3: Registro/latch de datos GPIO.

## Retos tecnicos
- Integridad de senal: skew y reflexiones en cable flex a 33 MHz; considerar resistencias de terminacion (~22 Ohm).
- Validacion temprana: medir RST#, CLK con osciloscopio; usar analizador logico antes de poblar GAL.

## Pasos siguientes sugeridos
- Completar adaptador fisico y confirmar arranque de la placa con el adaptador vacio.
- Medir alimentacion (+5V) y tierra en el zocalo; verificar CLK ~33 MHz.
- Probar ciclo de IO desde EFI Shell (ej. `io -w 1 300 01`) para observar FRAME# y AD[1:0].

## Pines criticos para Ruta B (minimo viable)
- Alimentacion: +5V, GND.
- Reloj y reset: CLK, RST#.
- Direccion/datos: AD[7:0] suficientes para 0x300.
- Comando: C/BE[3:0]# (lectura 0010, escritura 0011).
- Control: FRAME# (inicio), IRDY#/TRDY# (handshake), DEVSEL# (respuesta target).

## Mapeo de pines esenciales (extracto)
- FRAME#, DEVSEL#, TRDY#, C/BE[1:0]#, AD[9:0], +5V, GND, RST#, CLK, IDSEL opcional para futura ruta Plug & Play.
- Consejos: contar pines desde el bracket; intercalar GND en el flex para reducir crosstalk; para 0x300, AD[9:8]=11, AD[7:2]=000000.

## Validacion fisica recomendada
1) Alimentacion: 5V estables entre +5V y GND.
2) Oscilacion: CLK cuadrado a ~33.3 MHz; si hay overshoot, acortar flex o agregar terminacion ~22 Ohm.
3) Estimulacion EFI Shell: capturar FRAME#, AD[0:1], C/BE[0]# al ejecutar `io -w 1 300 01`; se espera caida de FRAME# y patron 0x300 en AD.

## Esbozo de logica GAL1 (decodificador IO 0x300)
- Condicion: FRAME# en bajo AND direccion en rango 0x300-0x303 AND C/BE[0]# en bajo.
- Accion: activar chip-select interno (MY_DEV) en bajo y reflejarlo en DEVSEL# para evitar master abort.
- Sugerencias: usar FIELD addr=[AD9..AD2]; address_match=addr:[300..303]; DEVSEL_OUT = MY_DEV; agregar desacoplos cercanos a la GAL.

## Riesgos y buenas practicas
- Revisar timing PCI 2.2 para decodificacion rapida (Fast/Medium DEVSEL# en ciclo 1-2 tras FRAME#).
- Mantener flex corto (<15 cm), con GND intercalado; confirmar malla de tierra.
- Verificar asignacion de pines en CUPL con el ruteo real antes de programar GAL.
