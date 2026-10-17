---
id: PRESENTATION
title: Orderflow Engine — Fuente de la presentación
status: accepted
lang: es
pair: presentation.md
last-reviewed: 2026-10-17
---

# Orderflow Engine

> **Pregunta que responde este documento:** ¿qué tiene el sistema y cómo funciona, paso a paso, diapositiva por diapositiva?

Motor de precios y confirmación idempotente de pedidos para un marketplace B2B de bebidas en varios países · Jesús Ariel González Bonilla · 2026-10-17

## El problema

- Los tenderos compran a distribuidores de bebidas en una plataforma B2B multi-tenant, en varios países y marcas
- Un pedido pasa por catálogo → carrito → checkout (cotización) → confirmación → historial
- Descuentos por escala, combos, obsequios, impuestos y cupo de crédito pueden actuar sobre el mismo pedido
- En tres releases, un pedido se confirmó y se devolvió su número, pero después no aparecía en el historial
- En algunos casos ni siquiera había quedado registrado
- El reto: hacer eso imposible, y que el total cotizado sea igual al total confirmado

> **Notas:** El enunciado describe un marketplace B2B donde los tenderos piden bebidas a los distribuidores, en varios países y marcas sobre una misma plataforma. Su punto más delicado es un incidente que se repitió en tres releases: el tendero recibía un número de pedido y luego el pedido no estaba en su historial. Todo lo que sigue está diseñado para que eso no vuelva a pasar.

## Qué se entregó

- A1: un motor de precios detrás de `POST /api/v1/quotes`, con el desglose por línea y los totales
- A2: un `POST /api/v1/orders` idempotente, el detalle del pedido y el historial de pedidos
- A3: un ADR de una página, en inglés y español, en Markdown, Word y PDF
- El front opcional, extendido a todo el flujo: carrito, cotización, confirmación, pedido y Mis pedidos
- El stack completo con Docker Compose, y un comando que corre todas las pruebas del espacio de trabajo
- Especificación, contrato de la API, modelo de datos, nueve diagramas y estos documentos de entrega

> **Notas:** Las tres partes del enunciado viven en el repositorio de la API: el motor, la confirmación y el ADR. Alrededor están el cliente web, que el dueño del producto amplió de una pantalla de resumen al flujo completo, la infraestructura que corre todo junto y la documentación. Nada de A1, A2 ni A3 depende del cliente web.

## Seis repositorios

- `orderflow-engine-api`: el entregable, con el código, las pruebas, el README y el ADR
- `orderflow-engine-web`: el cliente Angular, servido por nginx
- `orderflow-engine-infra`: el compose del stack completo, `test-all` y la suite de extremo a extremo
- `orderflow-engine-docs`: especificación, contrato, modelo de datos, arquitectura, diagramas y handoff
- `orderflow-engine-tools`: generadores y verificaciones de los documentos y los repositorios
- `orderflow-engine-app`: reservado para una futura app móvil, sin construir

> **Notas:** Cada componente vive en su propio repositorio, así que el servicio, el cliente, la infraestructura y los documentos evolucionan y se publican por separado. El repositorio de la API se sostiene solo como el entregable que pide el enunciado. El repositorio de infraestructura es el único que referencia a sus hermanos, porque los orquesta.

## Los seis repositorios de un vistazo

![Los seis repositorios del espacio de trabajo y lo que cada uno toma de otro](figures/02-workspace-repositories.png)

> **Notas:** Las flechas muestran lo que cada repositorio toma de otro: la especificación de `orderflow-engine-docs` gobierna a todos, los generadores de `orderflow-engine-tools` producen lo que publica la documentación, el cliente web sigue el contrato de la API, y `orderflow-engine-infra` ejecuta las pruebas de ambos.

## Topología en ejecución

- Un solo origen: nginx sirve el cliente Angular y reenvía `/api/` a la API, sin CORS
- Un servicio Spring Boot 4.1.1 sobre Java 21: adaptador web, casos de uso, motor puro, persistencia
- PostgreSQL 17 es la única fuente de verdad: SQL explícito y bloqueos de fila
- El push y el ERP solo los llama el relay del outbox, después del commit del pedido
- En esta entrega, el push y el ERP son fakes detrás de puertos que deduplican por `event_id`

![Topología en ejecución: un servicio, una base de datos, efectos solo por el outbox](figures/01-runtime-topology.png)

> **Notas:** En ejecución hay un servicio y una base de datos. El navegador solo llama a su propio origen, y nginx reenvía las rutas de la API. Nada en el camino de la petición llama al push ni al ERP: esos efectos salen por el relay del outbox una vez confirmado el pedido.

## Paso 1: elegir la tienda y armar el carrito

- El selector de tienda lista ocho tiendas sembradas, dos por país, agrupadas por tenant
- El carrito lee los productos del tenant desde la API en cada visita (`GET /api/v1/products`)
- Cada producto muestra su imagen, nombre, SKU, precio de lista sin impuestos y tasa de impuesto
- "Precios en COP" nombra la moneda, porque COP y USD comparten el símbolo `$`
- Se elige contado o crédito y se toca "Cotizar"
- El carrito solo muestra unidades: ni subtotal ni total hasta la cotización

![El carrito: el catálogo del tenant con precios de lista sin impuestos](screenshots/es/02-catalog.png)

> **Notas:** El selector de tienda reemplaza al inicio de sesión, que está fuera de alcance: envía un tenant y un cliente sembrados en las cabeceras. El carrito lista exactamente lo que la API puede cotizar y nunca calcula dinero, así que el total llega con la cotización. El navegador guarda un carrito por tienda.

## Paso 2: cotizar

- `POST /api/v1/quotes` calcula el precio del carrito una sola vez y sella la cotización
- El resumen muestra cada línea, los totales del pedido y hasta cuándo se garantiza el precio
- El precio se garantiza por 15 minutos, un tiempo de vida definido por país
- A crédito, muestra el cupo disponible y si el pedido cabe; todavía no se reserva nada
- Si no cabe, "Confirmar pedido" queda deshabilitado y "Pagar de contado" vuelve a cotizar de contado

![El resumen del pedido: líneas, totales y hasta cuándo se garantiza el precio](screenshots/es/04-quote-summary.png)

> **Notas:** Esta es la pantalla que el enunciado ofrecía como front opcional. Lee la cotización desde la API en cada visita y muestra cada monto tal como lo envió la API. La verificación de cupo aquí es informativa: el cupo solo se consume al confirmar.

## Paso 3: leer las palancas en la cotización

- Cada descuento muestra su origen: "Descuento por volumen" para una escala, o "Combo"
- Un combo es una sola entrada de descuento, y sus productos nunca muestran un descuento suelto
- Una línea dentro de un combo muestra su "Parte del combo", que solo sirve para derivar su base gravable
- Un obsequio es una línea aparte con su cantidad, sin costo, con su valor de referencia
- La base gravable, los impuestos y el total cierran el desglose

![El desglose de la cotización con las palancas aplicadas](screenshots/es/05-quote-levers.png)

> **Notas:** Este es el desglose que pide el enunciado: por línea y en totales, el subtotal bruto, los descuentos con su origen, los obsequios con su cantidad, la base gravable, los impuestos y el total. El combo aparece una sola vez, como exige el enunciado, y su parte en cada línea solo explica el impuesto de esa línea. Donde el país grava los obsequios, la pantalla aclara que el impuesto del obsequio lo paga el cliente.

## Paso 4: confirmar el pedido

- "Confirmar pedido" crea una clave de idempotencia para la cotización y la guarda en el navegador
- El botón queda deshabilitado mientras la petición está en curso, así que un doble tap envía una sola petición
- Un reintento tras no recibir respuesta, un 5xx o una recarga envía la misma clave
- Un 201 abre el pedido con su número y el aviso "Pedido confirmado"
- Un 201 repetido dice "Este pedido ya estaba registrado": ni un segundo pedido ni más cupo usado

![El pedido justo después de confirmarlo, con su número](screenshots/es/06-order-confirmed.png)

> **Notas:** La idempotencia del lado del cliente refleja la del servidor: una clave por cotización, creada en el primer tap y reutilizada en cada reintento. Si la red falla después del commit del pedido, el reintento recibe el mismo pedido en lugar de un duplicado. El carrito de la tienda se vacía después del 201.

## Paso 5: encontrarlo en Mis pedidos

- Mis pedidos lee el historial desde la API en cada visita (`GET /api/v1/orders?customerId=`)
- Del más reciente al más antiguo: número de pedido, fecha de confirmación y total con el código de su moneda
- El pedido recién confirmado aparece primero, con el total confirmado
- El cliente no guarda copia de ningún pedido, así que ninguna caché puede ocultar uno ya confirmado
- Cada fila abre su pedido, que se arma a partir de su cotización sellada

![Mis pedidos, del más reciente al más antiguo](screenshots/es/07-my-orders.png)

> **Notas:** Aquí se cierra el ciclo del defecto reportado: el número que el tendero acaba de recibir está en su historial de inmediato. El historial lee las mismas tablas primarias que escribió la confirmación, y el flujo de extremo a extremo lo verifica en español y en inglés.

## El pipeline de precios y el orden de las palancas

1. Elegibilidad de obsequios, sobre cantidades pagadas o el subtotal bruto antes de descuentos
2. Los combos consumen primero sus unidades: una entrada `COMBO`, nunca un segundo descuento sobre esas unidades
3. Descuentos por escala en tramos marginales, sobre las unidades que no consumió un combo
4. Neto de línea = bruto − descuentos, y luego la base gravable
5. Impuesto por línea con la tasa de cada SKU, y luego total = Σ neto + impuestos
6. Verificación de cupo sobre el total final, en la capa de aplicación, fuera del motor puro

> **Notas:** El motor es una función pura: primero normaliza el carrito, y el mismo carrito con la misma foto de datos siempre da el mismo desglose. Los combos van antes que las escalas para que las mismas unidades nunca se descuenten dos veces, y el impuesto se calcula después de los descuentos comerciales, como pide el enunciado. El golden test 1 muestra los tramos marginales: 25 colas a 2,500 dan 1,250 más 1,500, un descuento de 2,750 y un total de 71,103.

## El pipeline de precios de un vistazo

![El pipeline de precios: el orden en que se aplican las palancas](figures/05-pricing-pipeline.png)

> **Notas:** Los pasos en ámbar son los únicos puntos de redondeo: los descuentos y el impuesto de cada línea, con la escala y el modo de redondeo del país. El total es la suma de las líneas redondeadas y nunca se redondea otra vez, y la evaluación del cupo corre fuera del motor.

## Dinero y redondeo

- `Money` envuelve un `BigDecimal`; una prueba de arquitectura prohíbe `float` y `double` en el dominio
- La escala y el modo de redondeo salen de los datos del país: COP escala 0, USD escala 2, `HALF_UP`
- Nunca de `java.util.Currency`, que reporta COP con dos decimales
- Solo se redondea por línea: cada componente de descuento y cada impuesto
- Los totales son sumas de líneas ya redondeadas y nunca se vuelven a redondear
- En la API, el dinero es un string JSON con la escala de la moneda, nunca un número

> **Notas:** Redondear por línea coincide con la forma en que la factura electrónica lleva el impuesto, así que el pedido y su factura siempre cuadran. El golden test 4 lo muestra: un descuento de combo de 3,600 se reparte en 2,647 y 953 por mayor residuo, los impuestos son 1,397 y 132, y el total es 11,529. El cliente web tampoco hace aritmética con dinero: formatea los strings de la API.

## Cotización = confirmación

- La cotización queda sellada: sus líneas y componentes de descuento solo se insertan
- Cada línea guarda el precio y la tasa de impuesto vigentes al cotizar
- El pedido no guarda ningún monto: referencia la cotización, así que cada cifra existe una sola vez
- La confirmación nunca vuelve a ejecutar el motor, así que un cambio posterior de precios o palancas no llega al pedido
- Compromiso: una cotización abierta conserva su precio hasta vencer, como máximo 15 minutos
- El golden test 6 cambia un precio y desactiva una palanca entre ambos momentos: mismo total, y una cotización nueva difiere

![Cotización: el precio se calcula una vez y se sella](figures/06-quote-sequence.png)

> **Notas:** El total cotizado es igual al confirmado por construcción, no porque se comparen dos cálculos: hay una sola copia de cada cifra y el pedido apunta a ella. El ADR registra el compromiso: un precio equivocado se respeta como máximo durante un tiempo de vida, y la revocación de cotizaciones es el siguiente paso de su mitigación.

## La confirmación idempotente en una transacción

1. Insertar la clave de idempotencia; una petición concurrente con la misma clave espera por ella
2. Bloquear la cotización y verificar su dueño, que aún no tenga pedido y que no haya vencido
3. A crédito, una actualización condicional que nunca supera el cupo
4. Insertar el pedido (sin montos), su fila del libro de cupo si es a crédito, y los eventos del outbox para push y ERP
5. Guardar el cuerpo del 201 en la clave y hacer commit
6. Solo después del commit se envía el 201

![Confirmación: una transacción, y la respuesta solo después del commit](figures/07-confirm-transaction.png)

> **Notas:** Todas las escrituras de la confirmación ocurren en una transacción de PostgreSQL, y la respuesta se envía solo después del commit, así que un número de pedido devuelto siempre apunta a una fila confirmada y legible. La misma clave con el mismo cuerpo devuelve la respuesta guardada byte a byte, con `Idempotent-Replayed: true`, mientras que una petición fallida no se guarda, así que el cliente puede corregir la causa y reintentar. Los bloqueos se toman siempre en el orden clave, cotización, cupo, así que las confirmaciones concurrentes no pueden entrar en deadlock.

## El outbox: efectos que no se pierden ni se aplican dos veces

- La transacción del pedido escribe un evento del outbox para el push y otro para el ERP
- Cada segundo, el relay reclama los eventos pendientes con `FOR UPDATE SKIP LOCKED` bajo un lease de 30 segundos
- Llama al push o al ERP fuera de toda transacción, y les pasa el `event_id`
- Un fallo se reintenta con backoff exponencial y jitter completo, con un techo que va de 2 segundos a 5 minutos
- Tras 8 intentos, o ante un error que reintentar no corrige, el evento pasa a `DEAD` y se cuenta en `outbox_dead`
- Los consumidores deduplican por `event_id`, así que reenviar no tiene un segundo efecto

> **Notas:** Si el push o el ERP están caídos, el pedido sigue confirmado y el efecto espera en el outbox, el gauge `outbox_pending` muestra el atraso, y la entrega se reanuda cuando el sistema se recupera. Si el servicio se detiene entre el envío y la marca, el lease vence y el evento se envía de nuevo, lo cual es seguro porque los consumidores deduplican. Un operador reencola un evento `DEAD` con una actualización SQL, documentada en el README de la API.

## El ciclo de vida del outbox de un vistazo

![Ciclo de vida del outbox: entrega al menos una vez, los consumidores deduplican](figures/09-outbox-lifecycle.png)

> **Notas:** Un evento se escribe como `PENDING` en la transacción del pedido, se toma con un lease y `SKIP LOCKED` mientras se envía, y pasa a `SENT` cuando la llamada sale bien. Un fallo reintentable o un lease vencido lo devuelve a `PENDING` con backoff; un error no reintentable o el último intento lo deja en `DEAD`, y un operador puede reencolarlo.

## Cuatro países, dos idiomas

- `co-alpha`, Colombia: COP con escala 0, impuesto del 19% y del 5%, obsequios gravados
- `ec-alpha`, Ecuador: USD con escala 2, impuesto del 15%, obsequios exentos
- `gt-alpha`, Guatemala: GTQ con escala 2, impuesto del 12%, obsequios exentos
- `pe-alpha`, Perú: PEN con escala 2, impuesto del 18%, obsequios gravados
- Guatemala y Perú llegaron como un changelog de Liquibase con datos, sin cambiar el motor ni el código de la API
- Español por defecto e inglés completo: la API sigue `Accept-Language` y el cliente web cambia de idioma en ejecución

> **Notas:** La moneda, la escala, el modo de redondeo, las tasas de impuesto, la política de impuesto del obsequio y la vigencia de la cotización son datos del país, y ninguno está quemado en el código; Guatemala y Perú fueron la prueba, porque llegaron solo como datos. Agregar un idioma es un archivo por lado. El selector ES/EN traduce la interfaz, mientras que los nombres de productos y tiendas se quedan en el idioma del tenant.

## Calidad y evidencia

- API: 333 pruebas unitarias y 308 de integración en verde, entre ellas los seis golden tests
- JaCoCo: 100% de líneas y ramas en `pricing.domain` y `pricing.domain.lever` (umbral 90%); 99.2% de líneas / 98.0% de ramas en todo el servicio
- PIT: 185 de 185 mutaciones detectadas (100%, umbral 80%)
- Web: 209 specs en 15 archivos, en Chromium headless
- De extremo a extremo: 2 de 2 flujos pasan, en español y en inglés, contra el stack completo
- `test-all`: OK sobre clones nuevos, el 2026-10-17

> **Notas:** Estas cifras salen del handoff, cada una con el comando que la produjo. Las pruebas de integración corren contra un PostgreSQL 17 real con Testcontainers, y ArchUnit mantiene el dominio puro y el límite de la transacción en los casos de uso. Las pruebas de mutación demuestran que las pruebas de precios detectan código incorrecto, no solo que lo ejecutan.

## Cómo ejecutarlo

1. Clonar los seis repositorios lado a lado; Docker debe estar corriendo, con JDK 21 y Node 18 o superior para `test-all`
2. `docker compose up --build` en `orderflow-engine-infra`: la base de datos, la API y el cliente web
3. Abrir `http://localhost:8081`; la salud de la API está en `http://localhost:8080/actuator/health`
4. `./mvnw verify` en `orderflow-engine-api`: el comando único para el código y las pruebas de la API
5. `bash scripts/test-all.sh` en `orderflow-engine-infra`: API, web, extremo a extremo, documentación e integridad

> **Notas:** Tres comandos lo cubren: uno levanta el stack, otro corre las pruebas de la API y otro corre todas las verificaciones del espacio de trabajo. Compose construye las imágenes de la API y del cliente web desde los repositorios hermanos y arranca primero PostgreSQL, y Docker reinicia cada servicio salvo que se haya detenido a mano. En Windows, `pwsh -File scripts/test-all.ps1` corre los mismos pasos.

## Qué no se incluyó y por qué

- Autenticación: fuera de alcance; tenant y cliente llegan en cabeceras, y un gateway los inyectaría
- Administración de catálogo y palancas: los changesets de Liquibase siembran tiendas, productos y palancas
- Cobro, cancelación y liberación de cupo: no se piden
- Clientes reales de push y ERP: los fakes detrás de puertos prueban las garantías de entrega
- Revocación de cotizaciones: el siguiente paso de la mitigación del riesgo del ADR
- Pipeline de CI y SonarQube: la verificación corre en local; un CI correría el mismo `./mvnw verify`

> **Notas:** Cada recorte es deliberado y está registrado, con su motivo, en el README de la API y en el ADR. SonarQube importaría el reporte de JaCoCo que el build ya genera, así que un CI puede agregarlo sin cambiar el proyecto. El repositorio de la app móvil está reservado y no se construyó.

## Cuatro garantías, y preguntas

- El total cotizado es el total confirmado: cotizaciones selladas, de solo inserción
- Nada de punto flotante para dinero: `BigDecimal`, escala tomada de los datos del país, redondeo por línea
- Un 201 significa que el pedido existe y se puede consultar: una transacción, respuesta después del commit
- Los efectos en push y ERP nunca se pierden ni se aplican dos veces: outbox y `event_id`
- Cada garantía tiene sus pruebas: golden, de integración, de mutación y de extremo a extremo
- ¿Preguntas?

> **Notas:** Estas son las cuatro garantías de la especificación, cada una con el mecanismo que la sostiene. El defecto que reportó el enunciado, un número de pedido devuelto y luego ausente, no puede ocurrir por construcción, y las pruebas lo demuestran. Gracias; con gusto recorro cualquier parte en el código.
