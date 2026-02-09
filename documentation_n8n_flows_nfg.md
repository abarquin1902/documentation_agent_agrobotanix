# Flujos N8N - Nanofactor Global

## 🎯 Overview
- **The Real Green - Agente**: Agente dedicado a la atención de clientes de las compras / información sobre 
los productos de The Real Green.
- **ByteGPT + Chat Trigger**: Agente especializado en cultivos de México, dedicado a la atención de clientes sobre 
Agrobotanix, principalmente el producto Agrocker.
- **Collection TRG Products**: Flujo diseñado para consultar los productos de Shopify de The Real Green y generar una colección donde se actualicen precios e información que alimentan al agente : "The Real Green - Agente".
- **Collection TRG Recommendations**: Flujo diseñado para generar una colección de guía de cuidados para las plantas
de los usuarios, para actualizar la información se debe de modificar el json fijo que se encuentra en el segundo nodo 
del flujo.
- **CreateUpdateQdrantAgroProducts**: Flujo diseñado para consultar los productos de Shopify de Agrobotanix y generar 
una colección donde se actualicen precios e información que alimentan al agente de Agro: "ByteGPT + Chat Trigger", el nombre de 
la colección es: agrobotanix_shopify_products.
- **CreateUpdateQdrantCollectionCrops**: Flujo diseñado para poner un json convertido del xlsx que se genera en el spreadsheets,
con este json se genera la colección que ayuda al agente de Agro: "ByteGPT + Chat Trigger" a ser un experto en cultivos.
Al ejecutar este flujo con el json actualizado, se actualiza la colección en Qdrant llamada: "agrobotanixnochon".
- **data_deletion_policy**: Flujo diseñado para tener un endpoint donde cada que vez que fb busque el aviso de 
borrado de datos para los usuarios de TRG, le retorna un html básico y un status 200 para no tener que agregar algo 
que no se utiliza en la pagina web de TRG.
- **Flujo Carrito Abandonado**: Flujo diseñado para cada hora buscar a los usuarios que tienen 3 horas que dejaron 
su carrito abandonado, después de esto les envía una plantilla a los que proporcionaron su número celular como recordatorio 
para que tomen su carrito abandonado.

## 📋 Flujos de Agentes

### The Real Green - Agente
- **Estructura general del flujo y características**:
    - Cuenta con historial de chat con Redis, donde se puede eliminar para hacer pruebas con el comando: "borrar memoria".
    - Cuenta con procesamiento de audio a texto.
    - Cuenta con prevención de mensajes duplicados con Redis.
    - Cuenta con consultas a las bases de conociemiento de: productos y la de cuidados de The Real Green.
    - Hay un agente especializado en responder en las redes sociales.
    - Hay un agente especializado en responder vía Whatsapp Business.
    - Cuenta con distintos tipos de canalizaciones a Kommo.

### ByteGPT + Chat Trigger (Agente de Agrobotanix)
- **Estructura general del flujo y características**:
    - Cuenta con historial de chat con Redis, donde se puede eliminar para hacer pruebas con el comando: "borrar memoria".
    - Cuenta con procesamiento de audio a texto.
    - Cuenta con prevención de mensajes duplicados con Redis.
    - Cuenta con almacenamiento de metadata para responder mejor los problemas de cultivos del usuario por medio de Redis.
    - Cuenta con consultas a las base de conocimientos maestra de cultivos y a la de productos de Agrobotanix.
    - Hay un agente especializado en responder dudas en general.
    - Hay un agente especializado en responder dudas sobre cultivos.
    - Hay un agente especializado en responder dudas sobre el seguimiento de pedido (IN PROGRESS).
    - Hay un agente especializado en crear la orden del usuario vía WA, solo redireccionando al metodo de pago (IN PROGRESS).
    - Cuenta con canalización a atención especializada.

## 🔄 Flujos de Generar/Actualizar colecciones en Qdrant

### Collection TRG Products
- **Qué hace**: Actualiza la colección trg_shopify_products en Qdrant con productos de Shopify
- **Frecuencia**: Cada que se ejecute este flujo en n8n.
- **⚠️ Importante**: No requiere modificaciones

### Collection TRG Recommendations
- **Qué hace**: Actualiza la colección trg_shopify_cuidados.
- **Frecuencia**: Cada que se ejecute este flujo en n8n.
- **📝 REQUIERE EDICIÓN MANUAL**: Para actualizar la colección, se debe de actualizar la información del json 
que se encuentra en el segundo nodo llamado: read_json_cuidados, despues de actualizar este nodo, solo se guarda el flujo y se 
ejecuta para actualizar / crear.

### CreateUpdateQdrantAgroProducts 
- **Qué hace**: Actualiza la colección agrobotanix_shopify_products.
- **Frecuencia**: Cada que se ejecute este flujo en n8n.
- **⚠️ Importante**: No requiere modificaciones.

### CreateUpdateQdrantCollectionCrops
- **Qué hace**: Actualiza la colección agrobotanixnochon.
- **Frecuencia**: Cada que se ejecute este flujo en n8n.
- **📝 REQUIERE EDICIÓN MANUAL**: Para actualizar la colección, se debe de actualizar la información del json 
que se encuentra en el segundo nodo llamado: read_json_expert_crops, despues de actualizar este nodo, solo se guarda el flujo y se 
ejecuta para actualizar / crear.

## 🔄 Flujos Adicionales

### data_deletion_policy
- **Qué hace**: Devuelve un status 200 y un html básico para cuando FB busque el aviso de borrado de datos para los usuarios de TRG.
- **Frecuencia**: Cada que se consulte el url del Webhook.
- **⚠️ Importante**: No requiere modificaciones

### Flujo Carrito Abandonado (Cron Job)
- **Qué hace**: Busca a todos los usuarios de la página web de TRG que llevan 3 horas de haber abandopnado su carrito de compras y que también 
propoprcionaron su número celular, después les envía una plantilla como recordatorio para que no dejen abandonado su carrito de compras.
- **Frecuencia**: Cada hora revisa a los usuarios que lleven tres horas de haber abandonado su carrito de compras.
- **⚠️ Importante**: No requiere modificaciones
