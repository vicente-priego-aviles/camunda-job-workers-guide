# Configuración del Proyecto

## Variables de Entorno

Este proyecto usa variables de entorno para configuración sensible.

### Setup Inicial

1. Copiar el archivo de ejemplo:

```bash
   cp .env.example .env
```
1. Editar `.env` con los valores correctos para tu entorno

2. **NUNCA** commitear el archivo `.env`

### Variables Requeridas

- `CAMUNDA_GRPC_ADDRESS`: Dirección del servidor gRPC de Camunda
- `CAMUNDA_REST_ADDRESS`: Dirección de la API REST de Camunda
- `SERVER_PORT`: Puerto donde correrá la aplicación
- `SPRING_PROFILES_ACTIVE`: Profile de Spring (dev, prod)

### Verificar Configuración

```bash
# Verificar que las variables están cargadas
echo $CAMUNDA_GRPC_ADDRESS
```
