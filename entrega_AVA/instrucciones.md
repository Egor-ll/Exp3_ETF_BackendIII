# Guía para ejecutar y comprobar BancoXYZ

**Asignatura:** Desarrollo Backend III (PBY2203), Semana 9  - Evaluacion Final Transversal
**Responsable de esta versión:** Egor Llancapichun  
**Repositorio de destino:** [Egor-ll/Exp3_ETF_BackendIII](https://github.com/Egor-ll/Exp3_ETF_BackendIII)

## 1. Alcance

Esta guía cubre la instalación local, el arranque de los servicios y las comprobaciones disponibles para BancoXYZ Semana 9. Está escrita para ejecutarse desde la raíz del repositorio, en Linux con Bash y Fish. Usa `bash` para los scripts `.sh` y `fish` para los scripts `.fish`.

Algunas validaciones ejecutan Jobs o crean transacciones de laboratorio. Antes de iniciarlas, revisa las advertencias de cada sección y confirma que Docker apunte al entorno local de pruebas.

### Reglas para proteger los datos

- No cambies los CSV de `Semana 9/batch-service/src/main/resources/data/`.
- No ejecutes `docker compose down -v`, `DROP`, `TRUNCATE` ni borrados manuales de la base.
- No ejecutes `Semana 9/database/02_seed_processed_snapshot.sql` como parte de la instalación actual.
- Las pruebas de Pagos escriben cuentas y movimientos QA en PostgreSQL; no son consultas de solo lectura.
- No publiques `.env`, tokens, contraseñas ni llaves privadas PKCS12.

## 2. Preparar el equipo

Instala Java 21, Git, Docker Engine con Docker Compose v2, Bash, Fish y las utilidades `curl`, `jq`, `python3` y `sha256sum` que usan algunos verificadores.

Clona el repositorio y entra a su carpeta:

```bash
git clone https://github.com/Egor-ll/Exp3_ETF_BackendIII.git
cd Exp3_ETF_BackendIII
```

Genera la configuración local y completa las variables requeridas en `.env`:

```bash
cp .env.example .env
chmod 600 .env
```

Usa secretos propios para la base de datos y los clientes OAuth2. No pegues esos valores en capturas, comandos compartidos ni archivos que subirás al repositorio.

Valida que Compose pueda interpretar la configuración:

```bash
docker compose config --quiet
```

Si falla, revisa las variables señaladas en el mensaje antes de iniciar contenedores.

## 3. Compilar y levantar el entorno local

Compila y ejecuta las pruebas Maven definidas por los módulos:

```bash
./mvnw clean verify
```

Este comando reconstruye los directorios `target/`; no borra los volúmenes de datos Docker.

Levanta los servicios:

```bash
docker compose up -d --build
docker compose ps
```

Espera a que los servicios con healthcheck estén `healthy` antes de correr las validaciones funcionales. Para revisar un servicio concreto:

```bash
docker compose logs --tail=100 postgres
docker compose logs --tail=100 keycloak
docker compose logs --tail=100 discovery-server
docker compose logs --tail=100 config-server
```

La definición local incluye 16 servicios. Spring Batch se ejecuta como proceso separado y usa la base PostgreSQL del stack.

## 4. Comprobar componentes

| Componente | Comprobación sugerida |
|---|---|
| PostgreSQL | `docker compose ps postgres`; revisar el estado `healthy`. |
| Kafka | `fish "Semana 9/scripts/verificar_kafka_cluster.fish"` |
| Kafka UI | Abrir `http://127.0.0.1:8090` desde el equipo donde corre Docker. |
| Keycloak / OAuth2 | `fish "Semana 9/scripts/verificar_oauth2.fish"` |
| Config Server | `docker compose logs --tail=100 config-server` |
| Eureka | `docker compose logs --tail=100 discovery-server` |
| Bank Backend | `docker compose ps bank-backend` |
| Consumidor de retiros | `docker compose logs --tail=100 retiros-event-consumer` |
| BFF Web, móvil y ATM | `fish "Semana 9/scripts/verificar_bff.fish"` |
| Cuentas y Clientes | `docker compose ps cuentas-service clientes-service` |
| Pagos | Ejecutar la prueba de laboratorio de la sección 6. |

Los endpoints protegidos requieren tokens válidos. Evita incluir JWT en evidencia pública.

## 5. Ejecutar los Jobs de Spring Batch

Los Jobs procesan estos archivos:

| Archivo | Job |
|---|---|
| `movimientos_financieros_diarios.csv` | `transaccionJob` |
| `intereses_trimestrales.csv` | `cuentaInteresJob` |
| `estados_financieros_anuales.csv` | `estadoCuentaAnualJob` |

Confirma primero que `postgres` está saludable y que la base existente es la de laboratorio `bancoxyz`. Después ejecuta:

```bash
bash "Semana 9/scripts/reparar_conexion_y_validar_batch_s9.sh"
```

El script verifica las tablas requeridas, calcula huellas de los CSV y lanza los Jobs contra el PostgreSQL de Compose. Escribe metadatos de Batch y resultados derivados. No modifica los CSV ni ejecuta DDL, `DROP` o `TRUNCATE`.

La evidencia entregada para esta versión registra tres ejecuciones completadas (`34`, `35` y `36`) y conteos de 50, 474, 20 y 261 en las tablas de resultados. Comprueba la salida de la ejecución actual en vez de tomar esos valores históricos como prueba de que tu instalación funcionó.

## 6. Probar Pagos

El endpoint de solicitudes de Pagos requiere JWT válido y una cabecera `Idempotency-Key` con un UUID. Las pruebas cubren el resultado de la operación, la repetición de la clave y la persistencia tras reiniciar el servicio.

Para ejecutar las operaciones de laboratorio:

```bash
bash "Semana 9/scripts/verificar_pagos_tls_kafka_s9.sh" --laboratorio
```

**Esta prueba escribe cuentas QA y movimientos en PostgreSQL.** Ejecuta solo contra una instancia de laboratorio y conserva la base si necesitas revisar los resultados. La evidencia existente informa operaciones `APLICADO`, una respuesta `409` al reutilizar la clave con otro cuerpo y persistencia después de reiniciar `pagos-service`; los resultados deben contrastarse con la salida local.

El servicio Pagos usa tablas de gestión propias; no modifica las tablas derivadas por los Jobs Batch ni representa una conexión con una red bancaria real.

## 7. Probar BFF, OAuth2 y resiliencia

Comprueba el acceso autenticado y el comportamiento por canal:

```fish
fish "Semana 9/scripts/verificar_oauth2.fish"
fish "Semana 9/scripts/verificar_bff.fish"
fish "Semana 9/scripts/verificar_actuator_resilience.fish"
```

El script de fallo y recuperación detiene temporalmente el contenedor `bank-backend` para observar el Circuit Breaker. Úsalo únicamente en el stack local y no durante una demostración con usuarios atendidos:

```fish
fish "Semana 9/scripts/verificar_resilience4j_fallo_recuperacion.fish"
```

El script incluye una rutina para volver a iniciar el contenedor al terminar. Aun así, verifica el estado con `docker compose ps` después de la prueba.

## 8. Comprobar Kafka

Ejecuta verificaciones de la topología y de los productores/consumidores:

```fish
fish "Semana 9/scripts/verificar_kafka_cluster.fish"
fish "Semana 9/scripts/verificar_kafka_microservicios.fish"
```

Para probar pausa y reanudación del listener:

```fish
fish "Semana 9/scripts/verificar_control_kafka_listener.fish"
```

La última prueba altera el consumo durante unos instantes. El entorno descrito por Compose usa tres brokers Kafka en el mismo host y `PLAINTEXT`; no es un clúster distribuido ni cifrado para producción.

## 9. Auditar y detener el stack

Para una auditoría estática de la estructura sin lanzar nuevamente los Jobs:

```bash
bash "Semana 9/scripts/cierre_total_bancoxyz_s9.sh" --estatico
```

El modo `--todo` incluye ejecución de Batch sobre la base ya existente. Úsalo solo cuando aceptes que se escriban nuevas ejecuciones y resultados derivados.

Si necesitas liberar espacio, primero simula la limpieza de salidas Maven:

```bash
bash "Semana 9/scripts/limpiar_targets_bancoxyz.sh" --dry-run
```

La ejecución sin `--dry-run` elimina solo directorios `target/` que pertenecen a módulos Maven; obliga a recompilar después. Para apagar contenedores sin eliminar datos:

```bash
docker compose down
```

No agregues `-v` si debes conservar los volúmenes de PostgreSQL y Kafka.

## 10. Ejecución en AWS EC2

El script de despliegue usa el perfil `ec2` y requiere variables, DNS y reglas de red configurados:

```bash
bash "Semana 9/scripts/desplegar_ec2_final.sh"
```

Antes de ejecutarlo, sigue la secuencia y requisitos de [despliegue.md](despliegue.md). Una ejecución satisfactoria debe comprobarse desde la instancia y desde un cliente externo. Los certificados locales de los BFF no sustituyen el TLS público de un dominio real.

## 11. Evidencias para la entrega

Guarda las salidas de comandos y capturas sin mostrar contraseñas, tokens ni llaves. La pauta también solicita un informe PDF basado en la plantilla del AVA y un video MP4 de 5–7 minutos con webcam. Esos elementos deben producirse aparte y colocarse con los documentos finales.

## Procedencia

Responsable de esta versión: Egor Llancapichun. El código recibido contiene trabajo previo e historial asociado a [LadyRed145/bancoxyzbatch](https://github.com/LadyRed145/bancoxyzbatch), con una atribución a Natalia Alvarado. La procedencia y los autores de cambios anteriores se mantienen identificables.

