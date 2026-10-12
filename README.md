# BancoXYZ: modernización de servicios bancarios

**Entrega:** Desarrollo Backend III, Semana 9 -  Evaluacion final transversal
**Responsable de esta entrega:** Egor Llancapichun  
**Repositorio de publicación:** [Egor-ll/Exp3_ETF_BackendIII](https://github.com/Egor-ll/Exp3_ETF_BackendIII)

## Propósito

Este proyecto presenta una propuesta de modernización para BancoXYZ. El sistema separa capacidades de negocio, adapta las respuestas a los canales digitales y automatiza procesos de datos que antes se ejecutaban en lotes. La solución está preparada para ejecutarse con Docker Compose y utiliza servicios de Spring Boot.

## Componentes

| Área | Función |
|---|---|
| Procesamiento batch | Jobs Spring Batch para resumir transacciones, calcular intereses y generar estados de cuenta anuales. |
| BFF | Aplicaciones independientes para Web, móvil y cajeros automáticos, con respuestas ajustadas a cada canal. |
| Servicios de negocio | Servicios de Cuentas, Clientes y Pagos, además del backend bancario. |
| Configuración y descubrimiento | Spring Cloud Config y Eureka para distribuir propiedades y registrar servicios. |
| Identidad | Keycloak y OAuth2/JWT para autenticar clientes y proteger endpoints. |
| Resiliencia | Resilience4j para limitar reintentos y aislar fallos de servicios dependientes. |
| Eventos | Apache Kafka para publicar y consumir eventos de operaciones. |
| Persistencia | PostgreSQL para los datos operacionales y el repositorio de metadatos de Spring Batch. |
| Despliegue | Docker Compose para levantar la topología de laboratorio; Caddy se habilita en el perfil de EC2. |

La definición de Compose contiene 16 servicios locales. Los Jobs batch se ejecutan como procesos separados y comparten la base de datos de la aplicación; no permanecen activos como un servicio de Compose.

## Tecnologías principales

Java 21, Spring Boot, Spring Batch, Spring Cloud, Spring Security, OAuth2, Keycloak, Resilience4j, Apache Kafka, PostgreSQL 17 y Docker Compose.

## Inicio rápido

1. Instala Java 21, Git y Docker con Docker Compose v2.
2. Clona el repositorio y entra en la carpeta del proyecto:

   ```bash
   git clone https://github.com/Egor-ll/Exp3_ETF_BackendIII.git
   cd Exp3_ETF_BackendIII
   ```

3. Crea la configuración local a partir de `.env.example` y completa las contraseñas y URL de OAuth2 en tu equipo. El archivo `.env` no debe publicarse.
4. Valida la configuración y levanta los contenedores:

   ```bash
   docker compose config --quiet
   docker compose up -d --build
   ```

5. Sigue [instrucciones.md](instrucciones.md) para iniciar los Jobs y comprobar cada servicio. Usa [despliegue.md](despliegue.md) para el entorno de AWS.

No ejecutes `docker compose down -v` si necesitas conservar los datos del volumen de PostgreSQL.

## Documentación de la entrega

- [Instrucciones de ejecución y comprobación](instrucciones.md)
- [Guía de despliegue](despliegue.md)
- [Propuesta técnica](Propuesta_Tecnica.md)

## Estado de la evidencia

La documentación del proyecto registra pruebas locales de los procesos batch, los BFF, Kafka y los servicios de negocio. El despliegue de Semana 9 en una instancia EC2 debe comprobarse directamente en AWS antes de presentarlo como disponible. El PDF técnico solicitado por la pauta y el video MP4 de 5 a 7 minutos no estaban incluidos en los archivos recibidos para esta actualización.

## Procedencia y contribuciones

Esta entrega está preparada para publicarse bajo la cuenta de Egor Llancapichun. El código recibido incluye trabajo previo asociado al repositorio [LadyRed145/bancoxyzbatch](https://github.com/LadyRed145/bancoxyzbatch); sus contribuciones e historial de Git se mantienen identificables. Las contribuciones posteriores deben quedar registradas con su autor correspondiente.
