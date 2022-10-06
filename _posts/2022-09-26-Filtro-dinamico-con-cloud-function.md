---
title: Http endpoint con filtro de búsqueda dinámico con serverless function
author: Benjamin
date: 2022-09-26 18:32:00 -0500
categories: [Programacion, Python, Serverless ]
tags: [python, cloud, gcp, serverless]
---

# Http endpoint con filtro de búsqueda dinámico con serverless function

En esta ocacion presentare la implementacion de una funcion serverless con GCP la idea aplica a otros proveedores de cloud computing solo que cambian el nombre y la forma de despliegue pero es la misma muñeca pero con diferente sombrero.

## Creando la funcion con la Gcloud CLI

Si tienes configurado tu consola ya sabras que puedes desplegar cualquier cosa en minutos con unas lineas de comando si no hechale una visita a este link y sigue las instrucciones del tio Google [Gcloud CLI](https://cloud.google.com/sdk/gcloud/). si ya tienes todo configurado el siguiente script deslplegara una cloud function que se gatillara con una peticion http

```bash

gcloud functions deploy songs-searchs-function \
--gen2 \
--runtime=python310 \
--env-vars-file .env.yaml \
--region=us-east1 \
--source=. \
--entry-point=request_http \
--trigger-http \
--allow-unauthenticated

```

Si todo sale bien reibiras una respuesta en la terminal con formato json a modod de resumen indicandote el endpoint http donde debes realizar la peticion y podras usar tu cloud-fucntion

## Song searcher application

