# Reservas App

Esta es una aplicación de reserva de salas y laboratorios por franja horaria. Está compuesta por tres capas: una interfaz web en React, una API REST en Node.js con Express, y una base de datos MySQL 8.4.

## Arquitectura

    navegador



    
    reservas-frontend
    React + Vite
    nginx:8080
    (host 3000)
    
    reservas-api
    Node 20 + Express
    :3000
    (host 3001, solo
    depuración)
    
    reservas-db
    MySQL 8.4
    :3306
    (sin puerto
    publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
|---|---|---|---|
| reservas-frontend | React + Vite servido por nginx | 8080 | 3000 |
| reservas-api | Node 20 + Express | 3000 | 3001 (solo depuración) |
| reservas-db | MySQL 8.4 | 3306 | - |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.

