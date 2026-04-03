# LogiCuba Pasajes v5.0

## Sistema Integral de Transporte Interprovincial para Cuba

---

**Versión:** 5.0
**Fecha:** Abril 2026
**Estado:** Documento Técnico Maestro
**Autor:** LogiCuba Development Team

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Arquitectura del Sistema](#2-arquitectura-del-sistema)
3. [Geografía de Cuba: Provincias y Municipios](#3-geografía-de-cuba-provincias-y-municipios)
4. [Sistema de Canales de Comunicación](#4-sistema-de-canales-de-comunicación)
5. [Integración de Pagos: Transfermóvil y EnZona](#5-integración-de-pagos-transfermóvil-y-enzona)
6. [API del Sistema](#6-api-del-sistema)
7. [Base de Datos](#7-base-de-datos)
8. [Seguridad y Protección](#8-seguridad-y-protección)
9. [Sistema Anti-Fraude](#9-sistema-anti-fraude)
10. [Flujos de Usuario](#10-flujos-de-usuario)
11. [Cola Offline](#11-cola-offline)
12. [Despliegue e Infraestructura](#12-despliegue-e-infraestructura)
13. [Glosario](#13-glosario)

---

## 1. Resumen Ejecutivo

### 1.1 Descripción del Proyecto

**LogiCuba Pasajes** es una plataforma digital diseñada para conectar pasajeros con transportistas en toda Cuba, funcionando incluso durante outages de internet. El sistema utiliza múltiples canales de comunicación (WhatsApp, Telegram, SMS) para garantizar máxima accesibilidad.

### 1.2 Propuesta de Valor

| Actor | Beneficio Principal |
|-------|-------------------|
| **Pasajeros** | Encuentran transporte confiable con precios transparentes, calificaciones verificadas y confirmación automática |
| **Transportistas** | Llenan asientos vacíos, reciben pagos garantizados y construyen reputación digital |
| **Ecosistema** | Digitaliza un mercado informal que mueve millones de CUP mensualmente |

### 1.3 Modelo de Negocio

| Concepto | Monto | Beneficiario |
|----------|-------|-------------|
| Tarifa de servicio | 50-150 CUP/reserva | LogiCuba |
| Comisión transportista | 100-500 CUP/reserva | LogiCuba |
| Suscripción Estándar | 2,000 CUP/mes | Transportistas opcionales |
| Suscripción Premium | 5,000 CUP/mes | Transportistas opcionales |

### 1.4 Stack Tecnológico

| Componente | Tecnología | Función |
|-----------|-----------|----------|
| Backend | Bun + Hono + TypeScript | API REST |
| Base de datos | PostgreSQL 16 + Redis 7 | Persistencia y caché |
| WhatsApp | Evolution API v1.8.2 | Canal primario |
| Telegram | Telegraf | Canal secundario |
| SMS | Xiaomi Gateway | Canal de respaldo + notificaciones |
| Pagos | Transfermóvil + EnZona | Procesamiento de pagos |
| Infraestructura | Oracle Cloud Free Tier | Servidor |
| Seguridad | Cloudflare Tunnel | Acceso seguro |

---

## 2. Arquitectura del Sistema

### 2.1 Vista General

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USUARIOS FINALES                                │
│                                                                             │
│    📱 Pasajeros               📱 Transportistas              👨‍💼 Admin       │
│    WhatsApp/Telegram/SMS      WhatsApp/Telegram/SMS       Web Panel        │
└───────────────────┬─────────────────────┬───────────────────┬───────────────┘
                    │                     │                   │
                    ▼                     ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ORACLE CLOUD (Backend Central)                        │
│                      ARM Ampere A1 • 4 CPU • 24 GB RAM                      │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐   │
│  │   Evolution     │  │    Telegraf    │  │     API LogiCuba           │   │
│  │   API v1.8.2   │  │   (Telegram)   │  │  Bun + Hono + TypeScript   │   │
│  │   Puerto 8080  │  │   Puerto 3004   │  │     Puerto 3002            │   │
│  └────────┬───────┘  └───────┬────────┘  └─────────────┬──────────────┘   │
│           │                   │                          │                  │
│           ▼                   ▼                          ▼                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PostgreSQL 16 + Redis 7                             │   │
│  │                    Persistencia • Caché • Sesiones                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐   │
│  │    Typebot     │  │  Uptime Kuma   │  │   Cola de Sincronización   │   │
│  │    Puerto 3000 │  │  Puerto 3003    │  │                            │   │
│  └────────────────┘  └────────────────┘  └────────────────────────────┘   │
│                                                                             │
│  IMPORTANTE: WhatsApp SOLO envía por WhatsApp (Evolution API)               │
│              WhatsApp NUNCA envía SMS reales                                │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              │ Solicitudes de envío SMS
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          XIAOMI REDMI 12R PRO                               │
│                       Gateway SMS + Procesador de Pagos                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Termux + Termux:API                           │   │
│  │                                                                      │   │
│  │  Script 1: sms-gateway.ts                                           │   │
│  │  ├── Recibe SMS de usuarios                                        │   │
│  │  ├── Procesa comandos comprimidos                                   │   │
│  │  ├── Consulta Oracle cuando hay internet                            │   │
│  │  ├── Cola offline cuando no hay conexión                           │   │
│  │  └── Responde al usuario por SMS                                   │   │
│  │                                                                      │   │
│  │  Script 2: sms-pagos.ts                                            │   │
│  │  ├── Intercepta SMS del número 900                                 │   │
│  │  ├── Parsea confirmaciones de Transfermóvil                        │   │
│  │  └── Envía datos al servidor Oracle                                │   │
│  │                                                                      │   │
│  │  Script 3: sms-notificaciones.ts                                   │   │
│  │  ├── Recibe solicitudes del backend                                │   │
│  │  ├── Procesa recordatorios y confirmaciones                       │   │
│  │  └── Envía SMS al usuario (NO por WhatsApp)                       │   │
│  │                                                                      │   │
│  │  Script 4: enzona-webhook.ts                                       │   │
│  │  ├── Recibe callbacks de EnZona                                    │   │
│  │  ├── Verifica estado de pagos                                      │   │
│  │  └── Confirma reservas                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  📡  SIM 1: Para pagos (compartido con sms-pagos)                          │
│  📡  SIM 2: Para usuarios (número público de LogiCuba)                     │
│  🔋  Conectado a WiFi + cargador permanente + UPS                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Principio Fundamental: Separación Estricta de Canales

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║                       SEPARACIÓN ESTRICTA DE CANALES                          ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                               ║
║  ┌───────────────────────────────────────────────────────────────────────┐   ║
║  │  WHATSAPP (Evolution API)                                              │   ║
║  │  ────────────────────────────────────                                  │   ║
║  │  • SOLO envía y recibe mensajes por WhatsApp                          │   ║
║  │  • Requiere smartphone con WhatsApp + datos móviles                    │   ║
║  │  • NUNCA envía SMS reales                                             │   ║
║  │  • NUNCA envía notificaciones proactivas                              │   ║
║  │  • Límite: 500 mensajes/día                                           │   ║
║  │  • Canal primario: 70% del tráfico esperado                            │   ║
║  └───────────────────────────────────────────────────────────────────────┘   ║
║                                                                               ║
║  ┌───────────────────────────────────────────────────────────────────────┐   ║
║  │  TELEGRAM (Telegraf)                                                   │   ║
║  │  ───────────────────────────────────                                   │   ║
║  │  • SOLO envía y recibe mensajes por Telegram                          │   ║
║  │  • Requiere smartphone con Telegram + datos móviles                    │   ║
║  │  • Sin límite de mensajes                                             │   ║
║  │  • Ideal para usuarios técnicos y developers                           │   ║
║  │  • Canal secundario: 20% del tráfico esperado                         │   ║
║  └───────────────────────────────────────────────────────────────────────┘   ║
║                                                                               ║
║  ┌───────────────────────────────────────────────────────────────────────┐   ║
║  │  SMS (Xiaomi Gateway)                                                  │   ║
║  │  ──────────────────────────                                            │   ║
║  │  • SOLO envía SMS reales por SIM móvil                                │   ║
║  │  • Funciona SIN internet (crítico para Cuba)                          │   ║
║  │  • Costo: 2 CUP por mensaje para el usuario                           │   ║
║  │  • USA para:                                                          │   ║
║  │    - Recibir mensajes de usuarios sin smartphone                       │   ║
║  │    - Responder consultas por SMS                                       │   ║
║  │    - Notificaciones proactivas (recordatorios, confirmaciones)        │   ║
║  │    - Recordatorios 24h y 2h antes del viaje                           │   ║
║  │  • Canal de respaldo: 10% del tráfico esperado                        │   ║
║  └───────────────────────────────────────────────────────────────────────┘   ║
║                                                                               ║
║  ⚠️  REGLA DE ORO: Cada canal comunica SOLO por su medio                    ║
║                                                                               ║
║     WhatsApp ──► WhatsApp    │    Telegram ──► Telegram    │    SMS ──► SMS  ║
║                                                                               ║
║  Razones de seguridad:                                                        ║
║  • Previene baneo de WhatsApp (patrones de spam evitados)                    ║
║  • Reduce riesgo de bloqueo por Meta                                          ║
║  • Notificaciones SMS llegan aunque no haya datos móviles                     ║
║                                                                               ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 2.3 Componentes de la Arquitectura

| Componente | Tecnología | Puerto | Función |
|-----------|-----------|--------|----------|
| API REST | Bun + Hono + TypeScript | 3002 | Motor de lógica de negocio y orquestación |
| PostgreSQL | PostgreSQL 16 Alpine | 5432 | Persistencia de datos |
| Redis | Redis 7 Alpine | 6379 | Caché, sesiones y colas |
| Evolution API | v1.8.2 + Baileys | 8080 | Integración WhatsApp (SOLO envío WA) |
| Telegraf | Bot Telegram | 3004 | Integración Telegram (SOLO envío TG) |
| SMS Gateway | Xiaomi + Termux | N/A | Gateway SMS (envío real) |
| Typebot | Builder | 3000 | Flujos conversacionales visuales |
| Uptime Kuma | Monitor | 3003 | Monitoreo de servicios |
| Cloudflared | Tunnel | Docker | Acceso seguro sin IP pública |

---

## 3. Geografía de Cuba: Provincias y Municipios

### 3.1 Estructura Administrativa

Cuba está dividida administrativamente en **15 provincias** y **1 municipio especial** (Isla de la Juventud), para un total de **168 municipios**.

### 3.2 Provincias con sus Municipios

#### Pinar del Río (11 municipios)

**Código ISO:** `PR` | **Capital:** Pinar del Río

| Código | Municipio | Tipo |
|--------|-----------|------|
| PR-01 | Mantua | Municipal |
| PR-02 | Minas de Matahambre | Municipal |
| PR-03 | Pinar del Río | Capital Provincial |
| PR-04 | San Juan y Martínez | Municipal |
| PR-05 | San Luis | Municipal |
| PR-06 | Sandino | Municipal |
| PR-07 | La Palma | Municipal |
| PR-08 | Los Palacios | Municipal |
| PR-09 | Consolación del Sur | Municipal |
| PR-10 | Guane | Municipal |
| PR-11 | Viñales | Municipal |

---

#### Artemisa (11 municipios)

**Código ISO:** `AR` | **Capital:** Artemisa

| Código | Municipio | Tipo |
|--------|-----------|------|
| AR-01 | Sandino | Municipal |
| AR-02 | Mariel | Municipal |
| AR-03 | Bauta | Municipal |
| AR-04 | Caimito | Municipal |
| AR-05 | Guanajay | Municipal |
| AR-06 | Artemisa | Capital Provincial |
| AR-07 | San Cristóbal | Municipal |
| AR-08 | Candelaria | Municipal |
| AR-09 | San Antonio de los Baños | Municipal |
| AR-10 | Güira de Melena | Municipal |
| AR-11 | Alquízar | Municipal |

---

#### La Habana (15 municipios)

**Código ISO:** `LH` | **Capital:** La Habana (Capital de la República)

| Código | Municipio | Tipo |
|--------|-----------|------|
| LH-01 | La Habana Vieja | Municipal |
| LH-02 | Centro Habana | Municipal |
| LH-03 | Regla | Municipal |
| LH-04 | Guanabacoa | Municipal |
| LH-05 | San Miguel del Padrón | Municipal |
| LH-06 | Diez de Octubre | Municipal |
| LH-07 | Cerro | Municipal |
| LH-08 | Marianao | Municipal |
| LH-09 | Plaza de la Revolución | Municipal |
| LH-10 | Playa | Municipal |
| LH-11 | La Lisa | Municipal |
| LH-12 | Boyeros | Municipal |
| LH-13 | Arroyo Naranjo | Municipal |
| LH-14 | Cotorro | Municipal |
| LH-15 | La Habana del Este | Municipal |

---

#### Mayabeque (11 municipios)

**Código ISO:** `MJ` | **Capital:** San José de las Lajas

| Código | Municipio | Tipo |
|--------|-----------|------|
| MJ-01 | San José de las Lajas | Capital Provincial |
| MJ-02 | Jaruco | Municipal |
| MJ-03 | Santa Cruz del Norte | Municipal |
| MJ-04 | Madruga | Municipal |
| MJ-05 | Nueva Paz | Municipal |
| MJ-06 | San Nicolás | Municipal |
| MJ-07 | Güines | Municipal |
| MJ-08 | Melena del Sur | Municipal |
| MJ-09 | Batabanó | Municipal |
| MJ-10 | Quivicán | Municipal |
| MJ-11 | Bejucal | Municipal |

---

#### Matanzas (13 municipios)

**Código ISO:** `MT` | **Capital:** Matanzas

| Código | Municipio | Tipo |
|--------|-----------|------|
| MT-01 | Matanzas | Capital Provincial |
| MT-02 | Cárdenas | Municipal |
| MT-03 | Martí | Municipal |
| MT-04 | Colón | Municipal |
| MT-05 | Perico | Municipal |
| MT-06 | Juncalito | Municipal |
| MT-07 | Calimete | Municipal |
| MT-08 | Los Arabos | Municipal |
| MT-09 | Pedro Betancourt | Municipal |
| MT-10 | Jovellanos | Municipal |
| MT-11 | Limonar | Municipal |
| MT-12 | Unión de Reyes | Municipal |
| MT-13 | Jagüey Grande | Municipal |
| MT-14 | Ciénaga de Zapata | Municipal |

---

#### Cienfuegos (8 municipios)

**Código ISO:** `CF` | **Capital:** Cienfuegos (Patrimonio de la Humanidad)

| Código | Municipio | Tipo |
|--------|-----------|------|
| CF-01 | Cienfuegos | Capital Provincial |
| CF-02 | Palmira | Municipal |
| CF-03 | Abreus | Municipal |
| CF-04 | Cruces | Municipal |
| CF-05 | Rodas | Municipal |
| CF-06 | Aguada de Pasajeros | Municipal |
| CF-07 | Cumanayagua | Municipal |
| CF-08 | Lajas | Municipal |

---

#### Villa Clara (13 municipios)

**Código ISO:** `VC` | **Capital:** Santa Clara

| Código | Municipio | Tipo |
|--------|-----------|------|
| VC-01 | Santa Clara | Capital Provincial |
| VC-02 | Remedios | Municipal |
| VC-03 | San Juan de los Remedios | Municipal |
| VC-04 | Caibarién | Municipal |
| VC-05 | Camajuaní | Municipal |
| VC-06 | Placetas | Municipal |
| VC-07 | Cifuentes | Municipal |
| VC-08 | Sagua la Grande | Municipal |
| VC-09 | Quemado de Güines | Municipal |
| VC-10 | Corralillo | Municipal |
| VC-11 | Ranchuelo | Municipal |
| VC-12 | Encrucijada | Municipal |
| VC-13 | Manicaragua | Municipal |
| VC-14 | Santo Domingo | Municipal |

---

#### Sancti Spíritus (8 municipios)

**Código ISO:** `SS` | **Capital:** Sancti Spíritus

| Código | Municipio | Tipo |
|--------|-----------|------|
| SS-01 | Sancti Spíritus | Capital Provincial |
| SS-02 | Trinidad | Municipal (Patrimonio) |
| SS-03 | Sancti Spíritus | Municipal |
| SS-04 | Jatibonico | Municipal |
| SS-05 | Taguasco | Municipal |
| SS-06 | Cabaiguán | Municipal |
| SS-07 | Yaguajay | Municipal |
| SS-08 | La Sierpe | Municipal |
| SS-09 | Fomento | Municipal |

---

#### Ciego de Ávila (10 municipios)

**Código ISO:** `CA` | **Capital:** Ciego de Ávila

| Código | Municipio | Tipo |
|--------|-----------|------|
| CA-01 | Ciego de Ávila | Capital Provincial |
| CA-02 | Morón | Municipal |
| CA-03 | Bolivia | Municipal |
| CA-04 | Chambas | Municipal |
| CA-05 | Florencia | Municipal |
| CA-06 | Majagua | Municipal |
| CA-07 | Ciro Redondo | Municipal |
| CA-08 | Venezuela | Municipal |
| CA-09 | Baraguá | Municipal |
| CA-10 | Primero de Enero | Municipal |

---

#### Camagüey (13 municipios)

**Código ISO:** `CM` | **Capital:** Camagüey (Patrimonio de la Humanidad)

| Código | Municipio | Tipo |
|--------|-----------|------|
| CM-01 | Camagüey | Capital Provincial |
| CM-02 | Guáimaro | Municipal |
| CM-03 | Esmeralda | Municipal |
| CM-04 | Florida | Municipal |
| CM-05 | Camagüey | Municipal |
| CM-06 | Nuevitas | Municipal |
| CM-07 | Santa Cruz del Sur | Municipal |
| CM-08 | Minas | Municipal |
| CM-09 | Carlos M. de Céspedes | Municipal |
| CM-10 | Jimaguayú | Municipal |
| CM-11 | Najasa | Municipal |
| CM-12 | Sibanicú | Municipal |
| CM-13 | Sierra de Cubitas | Municipal |
| CM-14 | Vertientes | Municipal |

---

#### Las Tunas (8 municipios)

**Código ISO:** `LT` | **Capital:** Las Tunas

| Código | Municipio | Tipo |
|--------|-----------|------|
| LT-01 | Las Tunas | Capital Provincial |
| LT-02 | Manatí | Municipal |
| LT-03 | Puerto Padre | Municipal |
| LT-04 | Jesús Menéndez | Municipal |
| LT-05 | Majibacoa | Municipal |
| LT-06 | Jobabo | Municipal |
| LT-07 | Colombia | Municipal |
| LT-08 | Amancio | Municipal |

---

#### Holguín (14 municipios)

**Código ISO:** `HO` | **Capital:** Holguín

| Código | Municipio | Tipo |
|--------|-----------|------|
| HO-01 | Holguín | Capital Provincial |
| HO-02 | Gibara | Municipal |
| HO-03 | Banes | Municipal |
| HO-04 | Antilla | Municipal |
| HO-05 | Báguanos | Municipal |
| HO-06 | Calixto García | Municipal |
| HO-07 | Cacocum | Municipal |
| HO-08 | Urbano Noris | Municipal |
| HO-09 | Cueto | Municipal |
| HO-10 | Mayarí | Municipal |
| HO-11 | Frank País | Municipal |
| HO-12 | Sagua de Tánamo | Municipal |
| HO-13 | Moa | Municipal |
| HO-14 | Rafael Freyre | Municipal |

---

#### Granma (13 municipios)

**Código ISO:** `GR` | **Capital:** Bayamo

| Código | Municipio | Tipo |
|--------|-----------|------|
| GR-01 | Bayamo | Capital Provincial |
| GR-02 | Manzanillo | Municipal |
| GR-03 | Niquero | Municipal |
| GR-04 | Media Luna | Municipal |
| GR-05 | Pilón | Municipal |
| GR-06 | Campechuela | Municipal |
| GR-07 | Yara | Municipal |
| GR-08 | Buey Arriba | Municipal |
| GR-09 | Guisa | Municipal |
| GR-10 | Jiguaní | Municipal |
| GR-11 | Cauto Cristo | Municipal |
| GR-12 | Río Cauto | Municipal |
| GR-13 | Bartolomé Masó | Municipal |

---

#### Santiago de Cuba (9 municipios)

**Código ISO:** `SC` | **Capital:** Santiago de Cuba

| Código | Municipio | Tipo |
|--------|-----------|------|
| SC-01 | Santiago de Cuba | Capital Provincial |
| SC-02 | Palma Soriano | Municipal |
| SC-03 | San Luis | Municipal |
| SC-04 | Contramaestre | Municipal |
| SC-05 | Mella | Municipal |
| SC-06 | San Cristóbal | Municipal |
| SC-07 | Songo-La Maya | Municipal |
| SC-08 | Segundo Frente | Municipal |
| SC-09 | Tercer Frente | Municipal |
| SC-10 | Guamá | Municipal |

---

#### Guantánamo (10 municipios)

**Código ISO:** `GT` | **Capital:** Guantánamo

| Código | Municipio | Tipo |
|--------|-----------|------|
| GT-01 | Guantánamo | Capital Provincial |
| GT-02 | Baracoa | Municipal |
| GT-03 | Maisí | Municipal |
| GT-04 | Imías | Municipal |
| GT-05 | San Antonio del Sur | Municipal |
| GT-06 | Caimanera | Municipal |
| GT-07 | Manuel Tames | Municipal |
| GT-08 | Yateras | Municipal |
| GT-09 | Niceto Pérez | Municipal |
| GT-10 | El Salvador | Municipal |

---

#### Isla de la Juventud (Municipio Especial)

**Código ISO:** `IJ` | **Capital:** Nueva Gerona

| Código | Municipio | Tipo |
|--------|-----------|------|
| IJ-01 | Isla de la Juventud | Municipio Especial |
| IJ-02 | Nueva Gerona | Cabecera Municipal |
| IJ-03 | Santa Fe | Asentamiento |
| IJ-04 | Demajagua | Asentamiento |

---

### 3.3 Resumen Estadístico

| Nivel | Cantidad |
|-------|----------|
| Provincias | 15 |
| Municipio Especial | 1 |
| Total Municipios | 168 |
| Total Asentamientos | ~600+ |

### 3.4 Códigos de Provincia para SMS

Para optimizar el uso en comandos SMS, se utilizan códigos cortos:

| Código | Provincia | Capital |
|--------|-----------|---------|
| LH | La Habana | La Habana |
| PR | Pinar del Río | Pinar del Río |
| AR | Artemisa | Artemisa |
| MJ | Mayabeque | San José de las Lajas |
| MT | Matanzas | Matanzas |
| CF | Cienfuegos | Cienfuegos |
| VC | Villa Clara | Santa Clara |
| SS | Sancti Spíritus | Sancti Spíritus |
| CA | Ciego de Ávila | Ciego de Ávila |
| CM | Camagüey | Camagüey |
| LT | Las Tunas | Las Tunas |
| HO | Holguín | Holguín |
| GR | Granma | Bayamo |
| SC | Santiago de Cuba | Santiago de Cuba |
| GT | Guantánamo | Guantánamo |
| IJ | Isla de la Juventud | Nueva Gerona |

---

## 4. Sistema de Canales de Comunicación

### 4.1 Comparativa de Canales

| Característica | WhatsApp | Telegram | SMS |
|---------------|----------|----------|-----|
| **Costo usuario** | Datos (caro) | Datos (medio) | 2 CUP |
| **Requiere smartphone** | Sí | Sí | No |
| **Requiere internet** | Sí | Sí | No |
| **Funciona offline** | No | No | Sí |
| **Interfaz rica** | Sí (botones, imágenes) | Sí (stickers, menús) | No (solo texto) |
| **Riesgo de baneo** | Alto | Medio | Ninguno |
| **Límite mensajes** | 500/día | Sin límite | Sin límite |
| **Ubicación GPS** | Sí | Sí | No |
| **Canal confirmado** | No | No | Sí |
| **Velocidad respuesta** | Rápida | Rápida | Variable |
| **Penetración Cuba 2026** | 85-90% | 25-35% | 91% |
| **Uso en LogiCuba** | Primario | Secundario | Respaldo/Notif. |

### 4.2 Cuándo Usar Cada Canal

| Escenario | Canal | Razón |
|-----------|-------|-------|
| Usuario con internet estable | WhatsApp | Mejor UX |
| Usuario con internet limitada | Telegram | Menor consumo datos |
| Apagón/crisis conectividad | SMS | Solo canal funcional |
| Transportista en ruta | SMS | No requiere concentración |
| Recordatorios 24h/2h | SMS (Xiaomi) | Llega sin internet |
| Confirmaciones proactivas | SMS (Xiaomi) | No зависит de WA |
| Pasajero urbano | WhatsApp | Experiencia familiar |

### 4.3 Flujo de Enrutamiento

```
┌─────────────────────────────────────────────────────────────────┐
│                    MOTOR DE ENRUTAMIENTO                          │
│                                                                  │
│  Mensaje recibido ──► Identificar canal ──► Normalizar ──► Procesar   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  CANAL DETECTADO                                            │ │
│  │                                                              │ │
│  │  WhatsApp ──► WhatsApp Response                             │ │
│  │  ├── Botones interactivos                                   │ │
│  │  ├── Imágenes de vehículos                                   │ │
│  │  ├── Ubicación GPS                                          │ │
│  │  └── Flujo guiado con menús                                 │ │
│  │                                                              │ │
│  │  Telegram ──► Telegram Response                             │ │
│  │  ├── Comandos slash (/)                                     │ │
│  │  ├── Inline keyboards                                       │ │
│  │  └── Menús persistentes                                     │ │
│  │                                                              │ │
│  │  SMS ──► SMS Response                                       │ │
│  │  ├── Comandos cortos (B, P, R, C)                          │ │
│  │  ├── Respuestas comprimidas                                  │ │
│  │  └── Cola offline automática                                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  NORMALIZACIÓN                                              │ │
│  │                                                              │ │
│  │  Todos los canales ──► Formato interno estándar            │ │
│  │                                                              │ │
│  │  {                                                          │ │
│  │    canal: "whatsapp" | "telegram" | "sms",                │ │
│  │    telefono: "+535...",                                    │ │
│  │    usuario_id: "uuid",                                     │ │
│  │    comando: "BUSCAR",                                        │ │
│  │    parametros: [...],                                        │ │
│  │    timestamp: "ISO8601",                                    │ │
│  │    requiere_respuesta: true                                │ │
│  │  }                                                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Comandos SMS

| Comando | Función | Ejemplo |
|---------|---------|---------|
| B origen destino fecha | Buscar viaje | `B LH SC 15/04` |
| P origen destino hora precio asientos | Publicar viaje | `P LH SC 8:00 1800 4` |
| R codigo asientos | Reservar | `R ABC123 2` |
| C codigo | Confirmar pago | `C LCB-8X2K` |
| M | Mis viajes activos | `M` |
| MR | Mis reservas | `MR` |
| V codigo | Ver detalles | `V ABC123` |
| CAL 1-5 | Calificar | `CAL 5` |
| AY | Ayuda | `AY` |
| ESTADO codigo | Estado reserva | `ESTADO LCB-8X2K` |
| CANCELAR codigo | Cancelar reserva | `CANCELAR LCB-8X2K` |

---

## 5. Integración de Pagos: Transfermóvil y EnZona

### 5.1 Modelo de Pagos Híbrido

LogiCuba soporta dos métodos de pago para maximizar cobertura de usuarios:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MODELO DE PAGOS HÍBRIDO                              │
│                                                                             │
│                    ┌─────────────────────┐                                 │
│                    │   USUARIO LOGICUBA  │                                 │
│                    └──────────┬──────────┘                                 │
│                               │                                              │
│              ┌────────────────┼────────────────┐                              │
│              ▼                                 ▼                              │
│   ┌─────────────────────┐         ┌─────────────────────┐                  │
│   │   MÉTODO 1          │         │   MÉTODO 2          │                  │
│   │   TRANSFERMÓVIL    │         │   ENZONA            │                  │
│   │   (Principal)       │         │   (Alternativa)      │                  │
│   │                     │         │                     │                  │
│   │  • 5.6M usuarios   │         │  • +200K usuarios   │                  │
│   │  • Offline SMS      │         │  • API oficial       │                  │
│   │  • Sin registro     │         │  • Requiere internet │                  │
│   │  • Interceptación   │         │  • OAuth2 seguridad  │                  │
│   └─────────────────────┘         └─────────────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Transfermóvil (Método Principal)

#### 5.2.1 Flujo de Pago

```
┌─────────────────────────────────────────────────────────────────┐
│               FLUJO DE PAGO TRANSFERMÓVIL                       │
│                                                                 │
│  1. Usuario crea reserva                                        │
│     └─► Sistema genera código LCB-XXXX                          │
│     └─► Muestra número de pago LogiCuba                         │
│                                                                 │
│  2. Usuario abre app Transfermóvil                              │
│     └─► Transfiere al número indicado                           │
│                                                                 │
│  3. Banco envía SMS de confirmación                             │
│     └─► Número 900 (BANDEC)                                    │
│     └─► Contenido: monto, referencia, número origen             │
│                                                                 │
│  4. Xiaomi intercepta SMS                                      │
│     └─► Script sms-pagos.ts detecta SMS del 900                 │
│     └─► Parsea datos del mensaje                                │
│     └─► Envía a Oracle Cloud                                    │
│                                                                 │
│  5. Oracle verifica                                             │
│     └─► Monto coincide con reserva                              │
│     └─► Ventana de 30 minutos válida                           │
│     └─► Usuario no duplicado                                    │
│                                                                 │
│  6. Sistema confirma                                            │
│     └─► Reserva cambia a "pagada"                               │
│     └─► Notificación al transportista (SMS)                     │
│     └─► Confirmación al pasajero (SMS)                          │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.2.2 Script de Interceptación (TypeScript)

```typescript
/**
 * sms-pagos.ts
 * Script para Xiaomi Gateway que intercepta SMS del banco
 * y verifica pagos realizados por Transfermóvil
 */

interface PagoSMS {
  monto: number;
  referencia: string;
  numeroOrigen: string;
  fecha: Date;
  hash: string;
}

interface VerificacionPago {
  confirmado: boolean;
  codigoReserva?: string;
  mensaje: string;
  scoreFraude?: number;
}

// Número del banco que envía confirmaciones
const BANCO_NUMERO = '900';
const API_URL = process.env.API_URL || 'https://api.logicuba.cu';

/**
 * Procesa un SMS recibido del banco
 */
async function procesarSMSPago(sms: { number: string; body: string; date: string }): Promise<void> {
  // Ignorar SMS que no sean del banco
  if (sms.number !== BANCO_NUMERO) {
    return;
  }

  // Parsear el SMS del banco
  const pago = parsearSMSBanco(sms.body);
  if (!pago) {
    console.error('[PAGOS] Error al parsear SMS del banco');
    return;
  }

  // Verificar pago en el servidor
  const verificacion = await verificarPago(pago);

  if (verificacion.confirmado) {
    console.log(`[PAGOS] ✅ Pago confirmado para reserva: ${verificacion.codigoReserva}`);
    // Notificar a transportista y pasajero
    await notificarConfirmacion(verificacion.codigoReserva!);
  } else {
    console.log(`[PAGOS] ⚠️ Pago no podido confirmar: ${verificacion.mensaje}`);
  }
}

/**
 * Parsea el formato de SMS del banco
 * Formato esperado:
 * "Se ha recibido una transferencia de 3,650 CUP desde la cuenta
 *  terminada en 1234. Ref: ABC123. Saldo: 500 CUP"
 */
function parsearSMSBanco(texto: string): PagoSMS | null {
  try {
    // Extraer monto
    const montoMatch = texto.match(/(\d{1,3}(?:,\d{3})*(?:\.\d{2})?)\s*CUP/);
    if (!montoMatch) {
      console.error('[PAGOS] No se encontró monto en SMS');
      return null;
    }
    const monto = parseInt(montoMatch[1].replace(/,/g, ''));

    // Extraer referencia
    const refMatch = texto.match(/Ref:\s*(\w+)/);
    if (!refMatch) {
      console.error('[PAGOS] No se encontró referencia en SMS');
      return null;
    }
    const referencia = refMatch[1];

    // Extraer número de origen (últimos 4 dígitos)
    const origenMatch = texto.match(/terminada en (\d{4})/);
    const numeroOrigen = origenMatch ? origenMatch[1] : 'desconocido';

    // Generar hash para evitar duplicados
    const hash = generarHash(texto + texto.date);

    return {
      monto,
      referencia,
      numeroOrigen,
      fecha: new Date(texto.date),
      hash,
    };
  } catch (error) {
    console.error('[PAGOS] Error en parseo:', error);
    return null;
  }
}

/**
 * Genera hash SHA-256 para identificar SMS duplicados
 */
function generarHash(texto: string): string {
  // Implementación usando Web Crypto API o biblioteca crypto
  const encoder = new TextEncoder();
  const data = encoder.encode(texto);
  // En Node.js/Bun usar: crypto.createHash('sha256')
  // Por simplicidad, aquí usamos una implementación básica
  let hash = 0;
  for (let i = 0; i < texto.length; i++) {
    const char = texto.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash; // Convertir a 32bit integer
  }
  return Math.abs(hash).toString(16);
}

/**
 * Verifica el pago con el servidor
 */
async function verificarPago(pago: PagoSMS): Promise<VerificacionPago> {
  try {
    const response = await fetch(`${API_URL}/pagos/verificar`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': process.env.INTERNAL_API_KEY || '',
      },
      body: JSON.stringify({
        monto: pago.monto,
        referencia: pago.referencia,
        numero_origen: pago.numeroOrigen,
        hash_sms: pago.hash,
        fecha_sms: pago.fecha.toISOString(),
      }),
      signal: AbortSignal.timeout(15000),
    });

    if (!response.ok) {
      return {
        confirmado: false,
        mensaje: 'Error de conexión con servidor',
      };
    }

    return await response.json();
  } catch (error) {
    console.error('[PAGOS] Error al verificar pago:', error);
    return {
      confirmado: false,
      mensaje: 'Error de red',
    };
  }
}

/**
 * Notifica confirmación a las partes
 */
async function notificarConfirmacion(codigoReserva: string): Promise<void> {
  try {
    await fetch(`${API_URL}/reservas/${codigoReserva}/notificar-confirmacion`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': process.env.INTERNAL_API_KEY || '',
      },
    });
  } catch (error) {
    console.error('[PAGOS] Error al notificar:', error);
  }
}

// Exportar para uso en otros módulos
export { procesarSMSPago, parsearSMSBanco, verificarPago };
```

### 5.3 EnZona (Método Alternativo)

#### 5.3.1 Información General

EnZona es la plataforma de pagos electrónicos de Cuba con API documentada. Permite integración oficial para comercios.

**Portal:** https://www.enzona.net/
**Documentación API:** https://api.enzona.net/store/
**Librería Python:** `pip install enzona-api`
**Librería Dart/Flutter:** `dart pub add enzona`

#### 5.3.2 Proceso de Registro

```
PASO 1: Registrar el negocio
├── Acceder a https://bulevar.enzona.net/
├── Llenar formulario de comerciante
└── Esperar aprobación (72 horas)

PASO 2: Obtener credenciales
├── Acceder a https://api.enzona.net/store/
├── Copiar Consumer Key
├── Copiar Consumer Secret
└── Guardar de forma segura

PASO 3: Configurar en LogiCuba
├── Agregar credenciales como variables de entorno
├── Configurar URLs de retorno
└── Implementar webhook para callbacks
```

#### 5.3.3 Flujo de Pago EnZona

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUJO DE PAGO ENZONA                          │
│                                                                 │
│  1. Usuario selecciona "Pagar con EnZona"                      │
│     └─► Backend crea transacción                                 │
│     └─► Obtiene link de confirmación                            │
│     └─► Envía link al usuario                                   │
│                                                                 │
│  2. Usuario paga                                                │
│     └─► Redirige a plataforma EnZona                           │
│     └─► Confirma pago en EnZona                                │
│     └─► EnZona redirige a return_url                           │
│                                                                 │
│  3. Backend recibe callback                                     │
│     └─► Verifica transaction_uuid                              │
│     └─► Confirma estado del pago                                │
│     └─► Actualiza reserva a "pagada"                           │
│                                                                 │
│  4. Confirmación                                                │
│     └─► Notificación al transportista                          │
│     └─► Confirmación al pasajero                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.3.4 Implementación Backend (TypeScript)

```typescript
/**
 * enzona.service.ts
 * Servicio para procesar pagos a través de EnZona
 */

interface EnZonaConfig {
  consumerKey: string;
  consumerSecret: string;
  merchantUuid: string;
  returnUrl: string;
  cancelUrl: string;
  apiUrl: string; // https://api.enzona.net para producción
}

interface CreatePaymentRequest {
  descripcion: string;
  monto: number;
  moneda: 'CUP' | 'CUC' | 'USD';
  reservaid: string;
  items?: Array<{
    nombre: string;
    cantidad: number;
    precio: number;
  }>;
}

interface PaymentResult {
  exito: boolean;
  transactionUuid?: string;
  linkConfirmacion?: string;
  mensaje?: string;
}

interface PaymentVerification {
  exito: boolean;
  estado: 'ACEPTADO' | 'CONFIRMADO' | 'FALLIDO' | 'CANCELADO';
  mensaje: string;
}

class EnZonaService {
  private config: EnZonaConfig;
  private accessToken: string | null = null;
  private tokenExpiry: Date | null = null;

  constructor(config: EnZonaConfig) {
    this.config = config;
  }

  /**
   * Obtiene token de acceso OAuth2
   */
  private async obtenerAccessToken(): Promise<string> {
    // Verificar si el token actual es válido
    if (this.accessToken && this.tokenExpiry && this.tokenExpiry > new Date()) {
      return this.accessToken;
    }

    const credentials = Buffer.from(
      `${this.config.consumerKey}:${this.config.consumerSecret}`
    ).toString('base64');

    const response = await fetch(`${this.config.apiUrl}/token`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        Authorization: `Basic ${credentials}`,
      },
      body: 'grant_type=client_credentials',
    });

    if (!response.ok) {
      throw new Error('Error al obtener token de EnZona');
    }

    const data = await response.json();
    this.accessToken = data.access_token;

    // Token expira en 1 hora por defecto
    this.tokenExpiry = new Date(Date.now() + 55 * 60 * 1000);

    return this.accessToken!;
  }

  /**
   * Crea una transacción de pago en EnZona
   */
  async crearPago(request: CreatePaymentRequest): Promise<PaymentResult> {
    try {
      const token = await this.obtenerAccessToken();

      const paymentData = {
        description_payment: request.descripcion,
        merchant_uuid: this.config.merchantUuid,
        merchant_op_id: `LOGI-${request.reservaid}-${Date.now()}`,
        return_url: this.config.returnUrl,
        cancel_url: this.config.cancelUrl,
        currency: request.moneda,
        amount: {
          total: request.monto,
          shipping: 0,
          tax: 0,
          discount: 0,
          tip: 0,
        },
        lst_products: request.items || [
          {
            name: request.descripcion,
            quantity: 1,
            price: request.monto,
            tax: 0,
          },
        ],
      };

      const response = await fetch(`${this.config.apiUrl}/payments`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify(paymentData),
      });

      if (!response.ok) {
        const error = await response.json();
        return {
          exito: false,
          mensaje: error.message || 'Error al crear pago',
        };
      }

      const data = await response.json();

      return {
        exito: true,
        transactionUuid: data.transaction_uuid,
        linkConfirmacion: data.link_confirm,
      };
    } catch (error) {
      console.error('[EnZona] Error al crear pago:', error);
      return {
        exito: false,
        mensaje: 'Error de conexión con EnZona',
      };
    }
  }

  /**
   * Completa un pago (después de confirmación del usuario)
   */
  async completarPago(transactionUuid: string): Promise<PaymentVerification> {
    try {
      const token = await this.obtenerAccessToken();

      const response = await fetch(
        `${this.config.apiUrl}/payments/${transactionUuid}/complete`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
          },
        }
      );

      if (!response.ok) {
        return {
          exito: false,
          estado: 'FALLIDO',
          mensaje: 'Error al completar pago',
        };
      }

      const data = await response.json();

      return {
        exito: true,
        estado: data.status_denom || 'CONFIRMADO',
        mensaje: 'Pago completado exitosamente',
      };
    } catch (error) {
      console.error('[EnZona] Error al completar pago:', error);
      return {
        exito: false,
        estado: 'FALLIDO',
        mensaje: 'Error de conexión con EnZona',
      };
    }
  }

  /**
   * Obtiene estado de un pago
   */
  async obtenerEstadoPago(transactionUuid: string): Promise<PaymentVerification> {
    try {
      const token = await this.obtenerAccessToken();

      const response = await fetch(
        `${this.config.apiUrl}/payments/${transactionUuid}`,
        {
          method: 'GET',
          headers: {
            Authorization: `Bearer ${token}`,
          },
        }
      );

      if (!response.ok) {
        return {
          exito: false,
          estado: 'FALLIDO',
          mensaje: 'Pago no encontrado',
        };
      }

      const data = await response.json();

      return {
        exito: true,
        estado: data.status_denom,
        mensaje: data.status_denom,
      };
    } catch (error) {
      console.error('[EnZona] Error al obtener estado:', error);
      return {
        exito: false,
        estado: 'FALLIDO',
        mensaje: 'Error de conexión',
      };
    }
  }

  /**
   * Procesa reembolso
   */
  async reembolsar(
    transactionUuid: string,
    monto?: number,
    descripcion?: string
  ): Promise<{ exito: boolean; mensaje: string }> {
    try {
      const token = await this.obtenerAccessToken();

      const refundData: Record<string, unknown> = {};
      if (monto) {
        refundData.total = monto;
      }
      if (descripcion) {
        refundData.description = descripcion;
      }

      const response = await fetch(
        `${this.config.apiUrl}/payments/${transactionUuid}/refund`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
          },
          body: JSON.stringify(refundData),
        }
      );

      if (!response.ok) {
        return {
          exito: false,
          mensaje: 'Error al procesar reembolso',
        };
      }

      return {
        exito: true,
        mensaje: 'Reembolso procesado',
      };
    } catch (error) {
      console.error('[EnZona] Error al reembolsar:', error);
      return {
        exito: false,
        mensaje: 'Error de conexión',
      };
    }
  }
}

// Instancia singleton
const enzonaService = new EnZonaService({
  consumerKey: process.env.ENZONA_CONSUMER_KEY || '',
  consumerSecret: process.env.ENZONA_CONSUMER_SECRET || '',
  merchantUuid: process.env.ENZONA_MERCHANT_UUID || '',
  returnUrl: `${process.env.API_URL}/pagos/enzona/callback`,
  cancelUrl: `${process.env.API_URL}/pagos/enzona/cancelado`,
  apiUrl: process.env.ENZONA_API_URL || 'https://api.enzona.net',
});

export { EnZonaService, enzonaService };
```

#### 5.3.5 Endpoint de Callback (Express/Hono)

```typescript
/**
 * enzona.routes.ts
 * Rutas para callbacks de EnZona
 */

import { Hono } from 'hono';
import { enzonaService } from '../services/enzona.service';
import { reservaService } from '../services/reserva.service';

const router = new Hono();

/**
 * Callback después de que el usuario paga en EnZona
 * GET /pagos/enzona/callback?transaction_uuid=xxx
 */
router.get('/callback', async (c) => {
  const { transaction_uuid, status } = c.req.query();

  if (!transaction_uuid) {
    return c.json({ error: 'Transaction UUID requerido' }, 400);
  }

  try {
    // Verificar estado del pago
    const estado = await enzonaService.obtenerEstadoPago(transaction_uuid);

    if (estado.estado === 'CONFIRMADO') {
      // Extraer ID de reserva del merchant_op_id
      const reservaId = await obtenerReservaPorTransactionUuid(transaction_uuid);

      if (reservaId) {
        // Confirmar reserva
        await reservaService.confirmarPago(reservaId);

        return c.redirect(`${process.env.FRONTEND_URL}/reserva/${reservaId}/confirmada`);
      }
    }

    // Pago fallido o cancelado
    return c.redirect(`${process.env.FRONTEND_URL}/reserva/error?tx=${transaction_uuid}`);
  } catch (error) {
    console.error('[Callback] Error:', error);
    return c.json({ error: 'Error procesando callback' }, 500);
  }
});

/**
 * Webhook para confirmaciones de EnZona
 * POST /pagos/enzona/webhook
 */
router.post('/webhook', async (c) => {
  const body = await c.req.json();

  // Verificar autenticación del webhook
  const authHeader = c.req.header('Authorization');
  if (authHeader !== `Bearer ${process.env.ENZONA_WEBHOOK_SECRET}`) {
    return c.json({ error: 'No autorizado' }, 401);
  }

  try {
    const { transaction_uuid, status } = body;

    if (status === 'CONFIRMADO') {
      const reservaId = await obtenerReservaPorTransactionUuid(transaction_uuid);
      if (reservaId) {
        await reservaService.confirmarPago(reservaId);
      }
    }

    return c.json({ received: true });
  } catch (error) {
    console.error('[Webhook] Error:', error);
    return c.json({ error: 'Error procesando webhook' }, 500);
  }
});

/**
 * Cancelación de pago
 * GET /pagos/enzona/cancelado
 */
router.get('/cancelado', async (c) => {
  return c.redirect(`${process.env.FRONTEND_URL}/reserva/cancelada`);
});

export default router;
```

---

## 6. API del Sistema

### 6.1 Endpoints Principales

#### Autenticación

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/auth/registrar` | Registrar nuevo usuario |
| POST | `/auth/login` | Iniciar sesión |
| POST | `/auth/verificar` | Verificar código OTP |
| POST | `/auth/refresh` | Refrescar token |

#### Usuarios

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/usuarios/:id` | Obtener perfil |
| PUT | `/usuarios/:id` | Actualizar perfil |
| GET | `/usuarios/:id/viajes` | Viajes del usuario |
| GET | `/usuarios/:id/reservas` | Reservas del usuario |
| PUT | `/usuarios/:id/calificacion` | Actualizar rating |

#### Viajes

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/viajes` | Buscar viajes |
| POST | `/viajes` | Publicar viaje |
| GET | `/viajes/:id` | Detalles del viaje |
| PUT | `/viajes/:id` | Actualizar viaje |
| DELETE | `/viajes/:id` | Cancelar viaje |
| GET | `/viajes/:id/disponibilidad` | Asientos disponibles |

#### Reservas

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/reservas` | Crear reserva |
| GET | `/reservas/:id` | Detalles de reserva |
| PUT | `/reservas/:id/confirmar` | Confirmar pago |
| POST | `/reservas/:id/cancelar` | Cancelar reserva |
| GET | `/reservas/:codigo/:codigoConfirmacion` | Buscar por código |

#### Pagos

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/pagos/crear` | Crear pago |
| POST | `/pagos/verificar` | Verificar pago Transfermóvil |
| POST | `/pagos/enzona/callback` | Callback EnZona |
| GET | `/pagos/:id/estado` | Estado del pago |

### 6.2 Formato de Respuestas

```typescript
// Respuesta exitosa
interface ApiResponse<T> {
  exito: true;
  datos: T;
  mensaje?: string;
}

// Respuesta de error
interface ApiError {
  exito: false;
  error: {
    codigo: string;
    mensaje: string;
    detalles?: Record<string, string[]>;
  };
}
```

### 6.3 Ejemplo de Solicitud

```bash
# Buscar viajes
curl -X GET "https://api.logicuba.cu/viajes?origen=LH&destino=SC&fecha=2026-04-15" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json"

# Respuesta
{
  "exito": true,
  "datos": {
    "viajes": [
      {
        "id": "uuid-1234",
        "codigo": "ABC123",
        "transportista": {
          "id": "uuid-5678",
          "nombre": "Juan Carlos Pérez",
          "telefono": "+5351234567",
          "rating": 4.7,
          "totalViajes": 156
        },
        "origen": {
          "provincia": "LH",
          "municipio": "La Habana",
          "puntoEncuentro": "Parada Principal, Centro Habana"
        },
        "destino": {
          "provincia": "SC",
          "municipio": "Santa Clara"
        },
        "fecha": "2026-04-15",
        "hora": "08:00",
        "precioPorAsiento": 1800,
        "asientosTotales": 4,
        "asientosDisponibles": 2,
        "vehiculo": {
          "marca": "Toyota",
          "modelo": "Hiace",
          "color": "Blanco",
          "matricula": "ABC-123"
        },
        "estado": "activo"
      }
    ],
    "total": 1
  }
}
```

---

## 7. Base de Datos

### 7.1 Esquema de Provincias y Municipios

```sql
-- Tabla: provincias
CREATE TABLE provincias (
    id SERIAL PRIMARY KEY,
    codigo VARCHAR(2) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    capital VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Tabla: municipios
CREATE TABLE municipios (
    id SERIAL PRIMARY KEY,
    provincia_id INTEGER REFERENCES provincias(id) NOT NULL,
    codigo VARCHAR(4) NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    es_capital BOOLEAN DEFAULT FALSE,
    es_municipio_especial BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(provincia_id, codigo),
    UNIQUE(nombre)
);

-- Índices
CREATE INDEX idx_municipios_provincia ON municipios(provincia_id);
CREATE INDEX idx_municipios_codigo ON municipios(codigo);
```

### 7.2 Tablas Principales

```sql
-- Tabla: usuarios
CREATE TABLE usuarios (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    telefono VARCHAR(15) UNIQUE NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    tipo VARCHAR(20) NOT NULL CHECK (tipo IN ('transportista', 'pasajero', 'admin')),
    provincia_id INTEGER REFERENCES provincias(id),
    municipio_id INTEGER REFERENCES municipios(id),
    estado VARCHAR(20) DEFAULT 'activo' CHECK (estado IN ('activo', 'suspendido', 'baneado')),
    rating_promedio DECIMAL(3,2) DEFAULT 0.00 CHECK (rating_promedio >= 0 AND rating_promedio <= 5),
    total_calificaciones INTEGER DEFAULT 0,
    total_viajes INTEGER DEFAULT 0,
    badges JSONB DEFAULT '[]',
    suscripcion VARCHAR(20) DEFAULT 'gratis' CHECK (suscripcion IN ('gratis', 'estandar', 'premium')),
    suscripcion_expira TIMESTAMP WITH TIME ZONE,
    preferencia_pago VARCHAR(20) DEFAULT 'transfermovil' CHECK (preferencia_pago IN ('transfermovil', 'enzona')),
    fecha_registro TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    ultimo_acceso TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    canal_preferido VARCHAR(20) DEFAULT 'whatsapp',
    canales_activos JSONB DEFAULT '["whatsapp"]',
    metadata JSONB DEFAULT '{}',
    CONSTRAINT telefono_formato CHECK (telefono ~ '^\+53[5-9]\d{8}$')
);

CREATE INDEX idx_usuarios_telefono ON usuarios(telefono);
CREATE INDEX idx_usuarios_tipo ON usuarios(tipo);
CREATE INDEX idx_usuarios_canal ON usuarios(canal_preferido);
CREATE INDEX idx_usuarios_provincia ON usuarios(provincia_id);
```

```sql
-- Tabla: vehiculos
CREATE TABLE vehiculos (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    propietario_id UUID REFERENCES usuarios(id) NOT NULL,
    marca VARCHAR(50) NOT NULL,
    modelo VARCHAR(50) NOT NULL,
    año INTEGER,
    color VARCHAR(30),
    matricula VARCHAR(10) UNIQUE NOT NULL,
    capacidad INTEGER NOT NULL CHECK (capacidad > 0 AND capacidad <= 50),
    tiene_aire_acondicionado BOOLEAN DEFAULT FALSE,
    tiene_audio BOOLEAN DEFAULT FALSE,
    permite_mascotas BOOLEAN DEFAULT FALSE,
    permite_fumar BOOLEAN DEFAULT FALSE,
    foto_url TEXT,
    verificado BOOLEAN DEFAULT FALSE,
    fecha_registro TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT matricula_formato CHECK (matricula ~ '^[A-Z]{3}-\d{3}$')
);

CREATE INDEX idx_vehiculos_propietario ON vehiculos(propietario_id);
```

```sql
-- Tabla: viajes
CREATE TABLE viajes (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    codigo_corto VARCHAR(10) UNIQUE NOT NULL,
    transportista_id UUID REFERENCES usuarios(id) NOT NULL,
    vehiculo_id UUID REFERENCES vehiculos(id) NOT NULL,
    origen_provincia_id INTEGER REFERENCES provincias(id) NOT NULL,
    origen_municipio_id INTEGER REFERENCES municipios(id) NOT NULL,
    origen_punto_encuentro VARCHAR(200),
    destino_provincia_id INTEGER REFERENCES provincias(id) NOT NULL,
    destino_municipio_id INTEGER REFERENCES municipios(id) NOT NULL,
    destino_punto_encuentro VARCHAR(200),
    fecha_salida DATE NOT NULL,
    hora_salida TIME NOT NULL,
    hora_llegada_estimada TIME,
    precio_por_asiento INTEGER NOT NULL CHECK (precio_por_asiento > 0),
    asientos_totales INTEGER NOT NULL CHECK (asientos_totales > 0),
    asientos_disponibles INTEGER NOT NULL CHECK (asientos_disponibles >= 0),
    notas TEXT,
    estado VARCHAR(20) DEFAULT 'activo' CHECK (estado IN ('activo', 'lleno', 'en_curso', 'completado', 'cancelado')),
    canal_publicacion VARCHAR(20) DEFAULT 'whatsapp',
    fecha_creacion TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    fecha_actualizacion TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT asientos_disponibles CHECK (asientos_disponibles <= asientos_totales)
);

CREATE INDEX idx_viajes_ruta ON viajes(origen_provincia_id, destino_provincia_id, fecha_salida);
CREATE INDEX idx_viajes_fecha ON viajes(fecha_salida);
CREATE INDEX idx_viajes_transportista ON viajes(transportista_id);
CREATE INDEX idx_viajes_estado ON viajes(estado);
CREATE INDEX idx_viajes_codigo ON viajes(codigo_corto);
```

```sql
-- Tabla: reservas
CREATE TABLE reservas (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    codigo_confirmacion VARCHAR(10) UNIQUE NOT NULL,
    viaje_id UUID REFERENCES viajes(id) NOT NULL,
    pasajero_id UUID REFERENCES usuarios(id) NOT NULL,
    asientos_reservados INTEGER DEFAULT 1 CHECK (asientos_reservados > 0),
    precio_total INTEGER NOT NULL CHECK (precio_total > 0),
    tarifa_servicio INTEGER NOT NULL CHECK (tarifa_servicio >= 0),
    metodo_pago VARCHAR(20) DEFAULT 'transfermovil' CHECK (metodo_pago IN ('transfermovil', 'enzona')),
    estado VARCHAR(20) DEFAULT 'pendiente_pago' CHECK (estado IN (
        'pendiente_pago', 'pagada', 'confirmada', 'en_curso', 'completada', 'cancelada', 'no_show'
    )),
    canal_reserva VARCHAR(20) DEFAULT 'whatsapp',
    expira_en TIMESTAMP WITH TIME ZONE NOT NULL,
    fecha_creacion TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    fecha_pago TIMESTAMP WITH TIME ZONE,
    fecha_confirmacion TIMESTAMP WITH TIME ZONE,
    fecha_completado TIMESTAMP WITH TIME ZONE,
    fecha_cancelacion TIMESTAMP WITH TIME ZONE,
    motivo_cancelacion TEXT,
    -- Datos de pago Transfermóvil
    pago_transfermovil_hash VARCHAR(64),
    pago_transfermovil_monto INTEGER,
    pago_transfermovil_referencia VARCHAR(50),
    -- Datos de pago EnZona
    pago_enzona_transaction_uuid VARCHAR(100),
    -- Datos de pago
    pago_fecha_verificacion TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_reservas_viaje ON reservas(viaje_id);
CREATE INDEX idx_reservas_pasajero ON reservas(pasajero_id);
CREATE INDEX idx_reservas_estado ON reservas(estado);
CREATE INDEX idx_reservas_codigo ON reservas(codigo_confirmacion);
CREATE INDEX idx_reservas_expira ON reservas(expira_en) WHERE estado = 'pendiente_pago';
```

```sql
-- Tabla: calificaciones
CREATE TABLE calificaciones (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    reserva_id UUID REFERENCES reservas(id) NOT NULL,
    de_usuario_id UUID REFERENCES usuarios(id) NOT NULL,
    a_usuario_id UUID REFERENCES usuarios(id) NOT NULL,
    calificacion INTEGER NOT NULL CHECK (calificacion >= 1 AND calificacion <= 5),
    comentario TEXT,
    fecha_creacion TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(reserva_id, de_usuario_id)
);

CREATE INDEX idx_calificaciones_a_usuario ON calificaciones(a_usuario_id);
```

```sql
-- Tabla: mensajes_cola (cola offline)
CREATE TABLE mensajes_cola (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    telefono VARCHAR(15) NOT NULL,
    mensaje TEXT NOT NULL,
    canal VARCHAR(20) DEFAULT 'sms',
    estado VARCHAR(20) DEFAULT 'pendiente' CHECK (estado IN ('pendiente', 'procesado', 'fallido')),
    respuesta TEXT,
    recibido_en TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    procesado_en TIMESTAMP WITH TIME ZONE,
    cola_offline_id VARCHAR(50),
    intentos INTEGER DEFAULT 0,
    ultimo_intento TIMESTAMP WITH TIME ZONE,
    error_mensaje TEXT
);

CREATE INDEX idx_cola_estado ON mensajes_cola(estado, recibido_en);
CREATE INDEX idx_cola_telefono ON mensajes_cola(telefono);
```

### 7.3 Seed Data: Provincias y Municipios

```typescript
// data/provincias-municipios.seed.ts

export const PROVINCIAS_MUNICIPIOS = [
  {
    codigo: 'PR',
    nombre: 'Pinar del Río',
    capital: 'Pinar del Río',
    municipios: [
      'Mantua', 'Minas de Matahambre', 'Pinar del Río', 'San Juan y Martínez',
      'San Luis', 'Sandino', 'La Palma', 'Los Palacios', 'Consolación del Sur',
      'Guane', 'Viñales'
    ]
  },
  {
    codigo: 'AR',
    nombre: 'Artemisa',
    capital: 'Artemisa',
    municipios: [
      'Sandino', 'Mariel', 'Bauta', 'Caimito', 'Guanajay', 'Artemisa',
      'San Cristóbal', 'Candelaria', 'San Antonio de los Baños',
      'Güira de Melena', 'Alquízar'
    ]
  },
  {
    codigo: 'LH',
    nombre: 'La Habana',
    capital: 'La Habana',
    municipios: [
      'La Habana Vieja', 'Centro Habana', 'Regla', 'Guanabacoa',
      'San Miguel del Padrón', 'Diez de Octubre', 'Cerro', 'Marianao',
      'Plaza de la Revolución', 'Playa', 'La Lisa', 'Boyeros',
      'Arroyo Naranjo', 'Cotorro', 'La Habana del Este'
    ]
  },
  {
    codigo: 'MJ',
    nombre: 'Mayabeque',
    capital: 'San José de las Lajas',
    municipios: [
      'San José de las Lajas', 'Jaruco', 'Santa Cruz del Norte', 'Madruga',
      'Nueva Paz', 'San Nicolás', 'Güines', 'Melena del Sur',
      'Batabanó', 'Quivicán', 'Bejucal'
    ]
  },
  {
    codigo: 'MT',
    nombre: 'Matanzas',
    capital: 'Matanzas',
    municipios: [
      'Matanzas', 'Cárdenas', 'Martí', 'Colón', 'Perico', 'Jovellanos',
      'Calimete', 'Los Arabos', 'Pedro Betancourt', 'Limonar',
      'Unión de Reyes', 'Jagüey Grande', 'Ciénaga de Zapata'
    ]
  },
  {
    codigo: 'CF',
    nombre: 'Cienfuegos',
    capital: 'Cienfuegos',
    municipios: [
      'Cienfuegos', 'Palmira', 'Abreus', 'Cruces', 'Rodas',
      'Aguada de Pasajeros', 'Cumanayagua', 'Lajas'
    ]
  },
  {
    codigo: 'VC',
    nombre: 'Villa Clara',
    capital: 'Santa Clara',
    municipios: [
      'Santa Clara', 'San Juan de los Remedios', 'Caibarién', 'Camajuaní',
      'Placetas', 'Cifuentes', 'Sagua la Grande', 'Quemado de Güines',
      'Corralillo', 'Ranchuelo', 'Encrucijada', 'Manicaragua', 'Santo Domingo'
    ]
  },
  {
    codigo: 'SS',
    nombre: 'Sancti Spíritus',
    capital: 'Sancti Spíritus',
    municipios: [
      'Sancti Spíritus', 'Trinidad', 'Jatibonico', 'Taguasco', 'Cabaiguán',
      'Yaguajay', 'La Sierpe', 'Fomento'
    ]
  },
  {
    codigo: 'CA',
    nombre: 'Ciego de Ávila',
    capital: 'Ciego de Ávila',
    municipios: [
      'Ciego de Ávila', 'Morón', 'Bolivia', 'Chambas', 'Florencia',
      'Majagua', 'Ciro Redondo', 'Venezuela', 'Baraguá', 'Primero de Enero'
    ]
  },
  {
    codigo: 'CM',
    nombre: 'Camagüey',
    capital: 'Camagüey',
    municipios: [
      'Camagüey', 'Guáimaro', 'Esmeralda', 'Florida', 'Nuevitas',
      'Santa Cruz del Sur', 'Minas', 'Carlos M. de Céspedes', 'Jimaguayú',
      'Najasa', 'Sibanicú', 'Sierra de Cubitas', 'Vertientes'
    ]
  },
  {
    codigo: 'LT',
    nombre: 'Las Tunas',
    capital: 'Las Tunas',
    municipios: [
      'Las Tunas', 'Manatí', 'Puerto Padre', 'Jesús Menéndez',
      'Majibacoa', 'Jobabo', 'Colombia', 'Amancio'
    ]
  },
  {
    codigo: 'HO',
    nombre: 'Holguín',
    capital: 'Holguín',
    municipios: [
      'Holguín', 'Gibara', 'Banes', 'Antilla', 'Báguanos', 'Calixto García',
      'Cacocum', 'Urbano Noris', 'Cueto', 'Mayarí', 'Frank País',
      'Sagua de Tánamo', 'Moa', 'Rafael Freyre'
    ]
  },
  {
    codigo: 'GR',
    nombre: 'Granma',
    capital: 'Bayamo',
    municipios: [
      'Bayamo', 'Manzanillo', 'Niquero', 'Media Luna', 'Pilón',
      'Campechuela', 'Yara', 'Buey Arriba', 'Guisa', 'Jiguaní',
      'Cauto Cristo', 'Río Cauto', 'Bartolomé Masó'
    ]
  },
  {
    codigo: 'SC',
    nombre: 'Santiago de Cuba',
    capital: 'Santiago de Cuba',
    municipios: [
      'Santiago de Cuba', 'Palma Soriano', 'San Luis', 'Contramaestre',
      'Mella', 'Songo-La Maya', 'Segundo Frente', 'Tercer Frente', 'Guamá'
    ]
  },
  {
    codigo: 'GT',
    nombre: 'Guantánamo',
    capital: 'Guantánamo',
    municipios: [
      'Guantánamo', 'Baracoa', 'Maisí', 'Imías', 'San Antonio del Sur',
      'Caimanera', 'Manuel Tames', 'Yateras', 'Niceto Pérez', 'El Salvador'
    ]
  },
  {
    codigo: 'IJ',
    nombre: 'Isla de la Juventud',
    capital: 'Nueva Gerona',
    municipios: [
      'Isla de la Juventud'
    ],
    esMunicipioEspecial: true
  }
];

export function seedProvinciasMunicipios() {
  // Implementación para insertar datos en la base de datos
}
```

---

## 8. Seguridad y Protección

### 8.1 Principios de Seguridad

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║                         PRINCIPIOS DE SEGURIDAD                                ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                               ║
║  1. DEFENSA EN PROFUNDIDAD                                                   ║
║     ├── Múltiples capas de protección                                        ║
║     ├── Si una falla, otras siguen protegiendo                                ║
║     └── Principio: nunca confiar ciegamente                                   ║
║                                                                               ║
║  2. MÍNIMO PRIVILEGIO                                                        ║
║     ├── Cada componente tiene solo permisos necesarios                        ║
║     ├── Tokens con alcance limitado                                           ║
║     └── Credenciales segregadas                                               ║
║                                                                               ║
║  3. DEFENSE IN DEPTH                                                         ║
║     ├── Validación en entrada                                                ║
║     ├── Sanitización de datos                                                ║
║     ├── Prepared statements (SQL injection prevention)                        ║
║     └── Rate limiting y throttling                                            ║
║                                                                               ║
║  4. SECURE BY DEFAULT                                                        ║
║     ├── HTTPS obligatorio                                                     ║
║     ├── Headers de seguridad                                                  ║
║     ├── Tokens con expiración                                                ║
║     └── Logs sin datos sensibles                                             ║
║                                                                               ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 8.2 Autenticación y Autorización

```typescript
/**
 * Sistema de autenticación JWT con refresh tokens
 */

// Tipos
interface TokenPayload {
  usuarioId: string;
  tipo: 'access' | 'refresh';
  iat: number;
  exp: number;
}

interface UsuarioAutenticado {
  id: string;
  telefono: string;
  tipo: 'transportista' | 'pasajero' | 'admin';
}

// Generación de tokens
async function generarTokens(usuario: UsuarioAutenticado) {
  const accessToken = await new SignJWT({
    usuarioId: usuario.id,
    tipo: usuario.tipo,
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('15m') // 15 minutos
    .sign(env.JWT_SECRET);

  const refreshToken = await new SignJWT({
    usuarioId: usuario.id,
    tipo: 'refresh',
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('7d') // 7 días
    .sign(env.JWT_SECRET);

  return { accessToken, refreshToken };
}

// Middleware de autenticación
const authMiddleware = async (c: Context, next: Next) => {
  const authHeader = c.req.header('Authorization');

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return c.json({ error: 'Token requerido' }, 401);
  }

  const token = authHeader.substring(7);

  try {
    const payload = await jwtVerify(token, env.JWT_SECRET);

    c.set('usuario', {
      id: payload.usuarioId,
      tipo: payload.tipo,
    });

    await next();
  } catch (error) {
    if (error instanceof TokenExpiredError) {
      return c.json({ error: 'Token expirado', codigo: 'TOKEN_EXPIRADO' }, 401);
    }
    return c.json({ error: 'Token inválido' }, 401);
  }
};

// Middleware de autorización por rol
function requiereRol(...rolesPermitidos: string[]) {
  return async (c: Context, next: Next) => {
    const usuario = c.get('usuario') as UsuarioAutenticado;

    if (!usuario || !rolesPermitidos.includes(usuario.tipo)) {
      return c.json({ error: 'No autorizado' }, 403);
    }

    await next();
  };
}
```

### 8.3 Rate Limiting

```typescript
/**
 * Rate limiting para prevenir ataques de fuerza bruta
 */

import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

// Rate limiter para intentos de login
const loginRatelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '1 m'), // 5 intentos por minuto
  analytics: true,
  prefix: 'ratelimit:login',
});

// Rate limiter general para API
const apiRatelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'), // 100 requests por minuto
  analytics: true,
  prefix: 'ratelimit:api',
});

// Rate limiter para SMS (muy importante para prevenir abuso)
const smsRatelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 h'), // 10 SMS por hora por número
  analytics: true,
  prefix: 'ratelimit:sms',
});

// Middleware de rate limiting
async function rateLimitMiddleware(c: Context, next: Next) {
  const ip = c.req.header('x-forwarded-for') || c.req.header('cf-connecting-ip') || 'unknown';
  const path = c.req.path;

  let ratelimit: Ratelimit;

  if (path.includes('/auth/login')) {
    ratelimit = loginRatelimit;
  } else if (path.includes('/sms') || path.includes('/bot/sms')) {
    ratelimit = smsRatelimit;
  } else {
    ratelimit = apiRatelimit;
  }

  const { success, remaining, reset } = await ratelimit.limit(ip);

  // Agregar headers de rate limit
  c.res.headers.set('X-RateLimit-Remaining', remaining.toString());
  c.res.headers.set('X-RateLimit-Reset', reset.toString());

  if (!success) {
    return c.json({
      error: 'Demasiadas solicitudes',
      retryAfter: reset,
    }, 429);
  }

  await next();
}
```

### 8.4 Validación de Entrada

```typescript
/**
 * Validación robusta de datos de entrada usando Zod
 */

import { z } from 'zod';

// Esquema para registro de usuario
const esquemaRegistro = z.object({
  telefono: z
    .string()
    .regex(/^\+53[5-9]\d{8}$/, 'Formato de teléfono inválido (ej: +5351234567)'),
  nombre: z
    .string()
    .min(2, 'Nombre debe tener al menos 2 caracteres')
    .max(100, 'Nombre debe tener máximo 100 caracteres')
    .regex(/^[a-zA-ZáéíóúÁÉÍÓÚñÑ\s]+$/, 'Nombre solo puede contener letras'),
  tipo: z.enum(['transportista', 'pasajero']),
  provinciaCodigo: z.string().length(2),
});

// Esquema para búsqueda de viajes
const esquemaBusquedaViaje = z.object({
  origen: z.string().length(2),
  destino: z.string().length(2),
  fecha: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).pipe(
    z.coerce.date()
  ),
  asientos: z.number().int().min(1).max(10).optional().default(1),
});

// Esquema para crear reserva
const esquemaCrearReserva = z.object({
  viajeId: z.string().uuid(),
  asientos: z.number().int().min(1).max(4),
});

// Validar y sanitizar
function validar<T>(esquema: z.ZodSchema<T>, datos: unknown): T {
  const resultado = esquema.safeParse(datos);

  if (!resultado.success) {
    const errores: Record<string, string[]> = {};

    for (const issue of resultado.error.issues) {
      const campo = issue.path.join('.');
      if (!errores[campo]) {
        errores[campo] = [];
      }
      errores[campo].push(issue.message);
    }

    throw new ValidationError('Datos inválidos', errores);
  }

  return resultado.data;
}

// Middleware de validación
function validarSchema(esquema: z.ZodSchema) {
  return async (c: Context, next: Next) => {
    try {
      const body = await c.req.json();
      const datosValidados = validar(esquema, body);
      c.set('datos', datosValidados);
      await next();
    } catch (error) {
      if (error instanceof ValidationError) {
        return c.json({
          error: 'Validación fallida',
          detalles: error.detalles,
        }, 400);
      }
      throw error;
    }
  };
}
```

### 8.5 Headers de Seguridad

```typescript
/**
 * Configuración de headers de seguridad
 */

import { helmet } from 'hono/helmet';

// Middleware de seguridad
const securityHeaders = helmet({
  contentSecurityPolicy: {
    directive: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.logicuba.cu'],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
  xFrameOptions: 'DENY',
  xContentTypeOptions: 'nosniff',
  referrerPolicy: 'strict-origin-when-cross-origin',
  permissionsPolicy: {
    camera: [],
    microphone: [],
    geolocation: [],
  },
});

// Prevenir ataques de clickjacking
const antiClickjacking = async (c: Context, next: Next) => {
  c.res.headers.set('X-Frame-Options', 'DENY');
  c.res.headers.set('X-XSS-Protection', '1; mode=block');
  await next();
};
```

### 8.6 Cifrado de Datos Sensibles

```typescript
/**
 * Cifrado de datos sensibles en la base de datos
 */

import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from 'crypto';

// Configuración de cifrado
const ALGORITMO = 'aes-256-gcm';
const IV_LENGTH = 16;
const AUTH_TAG_LENGTH = 16;
const SALT_LENGTH = 32;

/**
 * Cifra un texto con clave derivada
 */
function cifrar(texto: string, claveMaestra: string): string {
  const salt = randomBytes(SALT_LENGTH);
  const key = scryptSync(claveMaestra, salt, 32);
  const iv = randomBytes(IV_LENGTH);

  const cipher = createCipheriv(ALGORITMO, key, iv);

  let cifrado = cipher.update(texto, 'utf8', 'hex');
  cifrado += cipher.final('hex');

  const authTag = cipher.getAuthTag();

  // Formato: salt:iv:authTag:cifrado
  return `${salt.toString('hex')}:${iv.toString('hex')}:${authTag.toString('hex')}:${cifrado}`;
}

/**
 * Descifra un texto cifrado
 */
function descifrar(textoCifrado: string, claveMaestra: string): string {
  const [saltHex, ivHex, authTagHex, cifrado] = textoCifrado.split(':');

  const salt = Buffer.from(saltHex, 'hex');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  const key = scryptSync(claveMaestra, salt, 32);

  const decipher = createDecipheriv(ALGORITMO, key, iv);
  decipher.setAuthTag(authTag);

  let texto = decipher.update(cifrado, 'hex', 'utf8');
  texto += decipher.final('utf8');

  return texto;
}

/**
 * Cifrar número de tarjeta (ejemplo)
 */
function cifrarTelefono(telefono: string): string {
  return cifrar(telefono, process.env.ENCRYPTION_KEY!);
}

function descifrarTelefono(telefonoCifrado: string): string {
  return descifrar(telefonoCifrado, process.env.ENCRYPTION_KEY!);
}
```

### 8.7 Variables de Entorno de Seguridad

```bash
# .env.example - NUNCA commitear este archivo

# Base de datos
DATABASE_URL=postgresql://usuario:password@host:5432/logicuba
POSTGRES_PASSWORD= 生成强密码

# Redis
REDIS_PASSWORD= 生成强密码

# JWT
JWT_SECRET= 生成256位随机字符串
JWT_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_IN=7d

# Encryption
ENCRYPTION_KEY= 生成256位随机字符串

# WhatsApp
EVOLUTION_API_KEY= 生成强密码

# Telegram
TELEGRAM_BOT_TOKEN= 从BotFather获取

# Cloudflare
CLOUDFLARE_TUNNEL_TOKEN= 从Cloudflare获取

# EnZona
ENZONA_CONSUMER_KEY= 从EnZona获取
ENZONA_CONSUMER_SECRET= 从EnZona获取
ENZONA_MERCHANT_UUID= 从EnZona获取
ENZONA_WEBHOOK_SECRET= 生成强密码

# API
INTERNAL_API_KEY= 生成强密码 (para comunicación Xiaomi-Backend)

# Rate Limiting
UPSTASH_REDIS_REST_URL= 从Upstash获取
UPSTASH_REDIS_REST_TOKEN= 从Upstash获取
```

---

## 9. Sistema Anti-Fraude

### 9.1 Estrategias de Detección

```typescript
/**
 * Sistema de detección de fraude
 */

interface FraudeCheckResult {
  esSospechoso: boolean;
  score: number;
  razones: string[];
  acciones: FraudeAccion[];
}

type FraudeAccion =
  | 'BLOQUEAR'
  | 'REVISAR_MANUAL'
  | 'REQUERIR_VERIFICACION'
  | 'LIMITAR_TRANSACCIONES';

interface PaymentData {
  monto: number;
  referencia: string;
  numeroOrigen: string;
  hashSms: string;
  fechaSms: Date;
}

interface ReservaData {
  id: string;
  precioTotal: number;
  tarifaServicio: number;
  expiraEn: Date;
  pasajeroId: string;
}

/**
 * Calcula score de fraude para un pago
 */
async function calcularScoreFraude(
  pago: PaymentData,
  reserva: ReservaData,
  usuario: { id: string; totalViajes: number; rating: number }
): Promise<FraudeCheckResult> {
  let score = 0;
  const razones: string[] = [];
  const acciones: FraudeAccion[] = [];

  // 1. Verificar monto coincide exactamente
  const montoEsperado = reserva.precioTotal + reserva.tarifaServicio;
  if (pago.monto !== montoEsperado) {
    score += 30;
    razones.push(`Monto no coincide: esperado ${montoEsperado}, recibido ${pago.monto}`);
  }

  // 2. Verificar ventana de tiempo (30 minutos)
  const ahora = new Date();
  const diffMs = ahora.getTime() - reserva.expiraEn.getTime();
  const diffMin = diffMs / (1000 * 60);

  if (diffMin > 5) {
    score += 25;
    razones.push(`Pago fuera de ventana: ${diffMin.toFixed(0)} minutos tarde`);
  }

  // 3. Verificar si el número está registrado
  const usuarioRegistrado = await verificarUsuarioRegistrado(pago.numeroOrigen);
  if (!usuarioRegistrado) {
    score += 20;
    razones.push('Número de origen no registrado en el sistema');
  }

  // 4. Historial del usuario
  if (usuario.totalViajes < 3) {
    score += 10;
    razones.push('Usuario nuevo con menos de 3 viajes');
  }

  if (usuario.rating < 3.0) {
    score += 5;
    razones.push('Usuario con rating bajo');
  }

  // 5. Verificar hash duplicado (SMS ya procesado)
  const hashDuplicado = await verificarHashDuplicado(pago.hashSms);
  if (hashDuplicado) {
    score += 50;
    razones.push('Hash de SMS duplicado - posible replay attack');
    acciones.push('BLOQUEAR');
  }

  // 6. Verificar rate limiting del número
  const intentosRecientes = await obtenerIntentosRecientes(pago.numeroOrigen);
  if (intentosRecientes > 3) {
    score += 15;
    razones.push(`Múltiples intentos de pago: ${intentosRecientes}`);
    acciones.push('REVISAR_MANUAL');
  }

  // 7. Detectar patrones anómalos
  const patronesAnomalos = await detectarPatronesAnomalos(
    pago.numeroOrigen,
    reserva.pasajeroId
  );
  if (patronesAnomalos.detectado) {
    score += patronesAnomalos.score;
    razones.push(...patronesAnomalos.razones);
  }

  // Determinar acciones según score
  if (score >= 50) {
    acciones.push('BLOQUEAR');
  } else if (score >= 30) {
    acciones.push('REVISAR_MANUAL');
  } else if (score >= 15) {
    acciones.push('REQUERIR_VERIFICACION');
  }

  return {
    esSospechoso: score >= 30,
    score,
    razones,
    acciones: [...new Set(acciones)], // Eliminar duplicados
  };
}

/**
 * Verifica si el hash ya fue procesado
 */
async function verificarHashDuplicado(hash: string): Promise<boolean> {
  const existente = await db.pagos.findFirst({
    where: { hash_sms: hash }
  });
  return !!existente;
}

/**
 * Obtiene intentos de pago recientes de un número
 */
async function obtenerIntentosRecientes(numero: string): Promise<number> {
  const hace30Min = new Date(Date.now() - 30 * 60 * 1000);

  const count = await db.pagos.count({
    where: {
      numero_origen: numero,
      fecha_intento: { gte: hace30Min },
    }
  });

  return count;
}

/**
 * Detecta patrones anómalos de comportamiento
 */
async function detectarPatronesAnomalos(
  numeroOrigen: string,
  pasajeroId: string
): Promise<{ detectado: boolean; score: number; razones: string[] }> {
  const razones: string[] = [];
  let score = 0;

  // Patrón: Mismo número hace pagos de diferentes usuarios
  const pagosMismoNumero = await db.reservas.count({
    where: {
      pago_transfermovil_referencia: { not: null },
      created_at: { gte: new Date(Date.now() - 24 * 60 * 60 * 1000) },
    }
  });

  if (pagosMismoNumero > 5) {
    score += 20;
    razones.push('Múltiples reservas desde mismo número en 24h');
  }

  // Patrón: Pagos consecutivos muy rápidos
  const ultimoPago = await db.pagos.findFirst({
    where: { numero_origen: numeroOrigen },
    orderBy: { fecha_intento: 'desc' }
  });

  if (ultimoPago) {
    const diffMs = Date.now() - new Date(ultimoPago.fecha_intento).getTime();
    if (diffMs < 60000) { // Menos de 1 minuto
      score += 15;
      razones.push('Pagos demasiado rápidos consecutivos');
    }
  }

  // Patrón: Mismo monto en diferentes reservas
  const reservasRecientes = await db.reservas.findMany({
    where: {
      pasajero_id: pasajeroId,
      fecha_creacion: { gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) },
    },
    select: { precio_total: true }
  });

  if (reservasRecientes.length >= 3) {
    const montos = reservasRecientes.map(r => r.precio_total);
    const todosIguales = montos.every(m => m === montos[0]);
    if (todosIguales) {
      score += 10;
      razones.push('Mismo monto en múltiples reservas');
    }
  }

  return {
    detectado: score > 0,
    score,
    razones,
  };
}
```

### 9.2 Tabla de blacklist

```sql
-- Tabla: blacklist_numeros
CREATE TABLE blacklist_numeros (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    telefono VARCHAR(15) NOT NULL UNIQUE,
    razon VARCHAR(100) NOT NULL,
    detalle TEXT,
    creado_por UUID REFERENCES usuarios(id),
    fecha_creacion TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    fecha_expiracion TIMESTAMP WITH TIME ZONE,
    activo BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_blacklist_telefono ON blacklist_numeros(telefono) WHERE activo = TRUE;
```

---

## 10. Flujos de Usuario

### 10.1 Flujo Pasajero: Buscar y Reservar

```
┌─────────────────────────────────────────────────────────────────┐
│              FLUJO PASAJERO: BUSCAR Y RESERVAR                  │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 1: Usuario contacta a LogiCuba                             │
│  ──────────────────────────────────────────────────────────    │
│                                                                 │
│  WhatsApp: "Hola"                                               │
│  Telegram: /start                                                │
│  SMS: (cualquier mensaje)                                        │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 2: Menú Principal                                         │
│  ──────────────────────────────                                  │
│                                                                 │
│  WhatsApp (botones):                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  🚌 Buscar Viaje    📋 Mis Reservas    ❓ Ayuda        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Telegram (menú):                                               │
│  /buscar | /reservas | /ayuda                                   │
│                                                                 │
│  SMS: Menú principal en texto                                     │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 3: Búsqueda de Viaje                                       │
│  ──────────────────────────                                      │
│                                                                 │
│  WhatsApp:                                                       │
│  Origen: [Lista de provincias]                                   │
│  Destino: [Lista de provincias]                                  │
│  Fecha: [Selector de fecha]                                      │
│                                                                 │
│  Telegram:                                                       │
│  /buscar LH SC 15/04                                             │
│                                                                 │
│  SMS:                                                            │
│  B LH SC 15/04                                                   │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 4: Resultados                                              │
│  ──────────────                                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  🚗 Juan Carlos Pérez    ⭐4.7 (156 viajes)            │    │
│  │  ────────────────────────────────────────              │    │
│  │  La Habana → Santa Clara                             │    │
│  │  📅 15 de Abril  |  🕗 8:00 AM                        │    │
│  │  💰 1,800 CUP por asiento                            │    │
│  │  💺 4 asientos disponibles                            │    │
│  │  ❄️ A/C | 🎵 Audio | 🐕 Mascotas: No                  │    │
│  │                                                       │    │
│  │  [Reservar Asientos]                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 5: Selección de Asientos                                  │
│  ───────────────────────────                                    │
│                                                                 │
│  ¿Cuántos asientos necesitas?                                    │
│  [1] [2] [3] [4]                                                │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 6: Confirmación de Reserva                                 │
│  ─────────────────────────────────                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ✅ RESERVA CREADA                                     │    │
│  │  ─────────────────────────────────────                  │    │
│  │  Código: LCB-8X2K                                     │    │
│  │  Viaje: La Habana → Santa Clara                       │    │
│  │  Fecha: 15 de Abril, 8:00 AM                           │    │
│  │  Pasajero: María López                                 │    │
│  │  Asientos: 2                                           │    │
│  │  ─────────────────────────────────────                  │    │
│  │  Subtotal: 3,600 CUP                                  │    │
│  │  Tarifa servicio: 150 CUP                             │    │
│  │  TOTAL: 3,750 CUP                                     │    │
│  │  ─────────────────────────────────────                  │    │
│  │  ⏰ Expira en 30 minutos                              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 7: Pago                                                   │
│  ─────────                                                      │
│                                                                 │
│  Para pagar, transfiere a:                                        │
│  📱 +5351234567 (Transfermóvil)                                  │
│  o                                                              │
│  💳 Pagar con EnZona                                             │
│                                                                 │
│  SMS (Xiaomi):                                                   │
│  Expira: 14:32                                                   │
│  Transfiere 3750 CUP al +5351234567                             │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 8: Confirmación de Pago                                    │
│  ────────────────────────────                                    │
│                                                                 │
│  ✅ PAGO CONFIRMADO                                              │
│                                                                 │
│  Tu viaje está reservado.                                         │
│  Código: LCB-8X2K                                                │
│  La Habana → Santa Clara                                        │
│  15/Abr, 8:00 AM                                                 │
│                                                                 │
│  Muestra este código al transportista al abordar.                 │
│                                                                 │
│  📱 Recordatorio: Te enviaremos SMS 24h y 2h antes.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Flujo Transportista: Publicar Viaje

```
┌─────────────────────────────────────────────────────────────────┐
│              FLUJO TRANSPORTISTA: PUBLICAR VIAJE                 │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 1: Registro (primera vez)                                  │
│  ──────────────────────────────                                  │
│                                                                 │
│  WhatsApp: "Hola, quiero registrarme como transportista"         │
│  Telegram: /registro                                             │
│  SMS: REG TIPO:TRANSPORTISTA NOMBRE:Juan Carlos                 │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 2: Registrar Vehículo                                     │
│  ────────────────────────────                                     │
│                                                                 │
│  WhatsApp:                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  🚗 Registrar Vehículo                                  │    │
│  │                                                       │    │
│  │  Marca: [Toyota]                                       │    │
│  │  Modelo: [Hiace]                                       │    │
│  │  Año: [2020]                                           │    │
│  │  Color: [Blanco]                                       │    │
│  │  Matrícula: [ABC-123]                                  │    │
│  │  Capacidad: [10] asientos                               │    │
│  │                                                       │    │
│  │  [ ] Aire acondicionado                                │    │
│  │  [ ] Audio                                             │    │
│  │  [ ] Permite mascotas                                  │    │
│  │                                                       │    │
│  │  [Enviar]                                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 3: Publicar Viaje                                          │
│  ──────────────────────                                          │
│                                                                 │
│  WhatsApp:                                                       │
│  📢 Publicar Viaje                                               │
│                                                                 │
│  Origen: [La Habana]                                             │
│  Punto de encuentro: [Parada Principal, Centro Habana]          │
│                                                                 │
│  Destino: [Santa Clara]                                          │
│  Punto de llegada: [Terminal de ómnibus]                        │
│                                                                 │
│  Fecha: [15 de Abril]                                            │
│  Hora de salida: [8:00 AM]                                        │
│                                                                 │
│  Precio por asiento: [1,800 CUP]                                │
│                                                                 │
│  [Publicar Viaje]                                                │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 4: Confirmación                                            │
│  ───────────────────                                              │
│                                                                 │
│  ✅ VIAJE PUBLICADO                                              │
│                                                                 │
│  Código: VH-ABC123                                              │
│  La Habana → Santa Clara                                         │
│  15/Abr, 8:00 AM                                                 │
│  1,800 CUP/asiento | 10 asientos                                │
│                                                                 │
│  Recibirás notificación cuando alguien reserve.                  │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 5: Notificación de Reserva                                  │
│  ─────────────────────────────────                               │
│                                                                 │
│  🔔 NUEVA RESERVA                                               │
│                                                                 │
│  Pasajero: María López                                          │
│  Asientos: 2                                                     │
│  Total: 3,750 CUP (3,600 + 150 comisión)                        │
│                                                                 │
│  Código de confirmación: LCB-8X2K                               │
│  El pasajero mostrará este código al abordar.                    │
│                                                                 │
│  [Confirmar Abordaje]                                            │
│                                                                 │
│  ════════════════════════════════════════════════════════════   │
│                                                                 │
│  PASO 6: Confirmar Abordaje                                       │
│  ─────────────────────────                                       │
│                                                                 │
│  Al confirmar, el viaje se marca como completado                  │
│  y se habilita la calificación.                                   │
│                                                                 │
│  [Confirmar que María abordó]                                    │
│                                                                 │
│  ✅ ABORDAJE CONFIRMADO                                          │
│                                                                 │
│  ¡Gracias! El viaje se ha completado.                            │
│  Califica a María: [⭐1] [⭐2] [⭐3] [⭐4] [⭐5]                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Cola Offline

### 11.1 Funcionamiento

```
┌─────────────────────────────────────────────────────────────────┐
│                    SISTEMA DE COLA OFFLINE                        │
│                                                                 │
│  ESCENARIO: Usuario envía SMS pero no hay internet              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PASO 1: Xiaomi recibe SMS                              │    │
│  │                                                         │    │
│  │  De: +5351234567                                        │    │
│  │  Mensaje: B LH SC 15/04                                 │    │
│  │  Timestamp: 14:32:00                                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PASO 2: ¿Hay internet?                                 │    │
│  │                                                         │    │
│  │  SÍ ──► Enviar inmediatamente a Oracle                  │    │
│  │                                                         │    │
│  │  NO ──► Guardar en cola local (sms-cola.json)          │    │
│  │                                                         │    │
│  │  Respuesta: "✅ Recibido. Procesando..."               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PASO 3: Sincronización cuando hay internet             │    │
│  │                                                         │    │
│  │  Cola se sincroniza con Oracle Cloud                     │    │
│  │  Oracle procesa y guarda respuesta                       │    │
│  │  Xiaomi recibe respuesta y envía SMS al usuario          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 Script de Cola Offline (TypeScript)

```typescript
/**
 * sms-gateway.ts
 * Script para Xiaomi Gateway con cola offline
 */

interface ColaItem {
  id: string;
  phone: string;
  message: string;
  timestamp: string;
  attempts: number;
  lastAttempt: string;
  status: 'pending' | 'sent' | 'failed';
}

interface SMS {
  number: string;
  body: string;
  date: string;
}

class SMSGateway {
  private readonly API_URL: string;
  private readonly QUEUE_FILE: string;
  private readonly MAX_REINTENTOS: number = 50;
  private cola: ColaItem[] = [];
  private online: boolean = false;
  private telefonoLogiCuba: string;

  constructor() {
    this.API_URL = process.env.API_URL || 'https://api.logicuba.cu';
    this.QUEUE_FILE = '/data/data/com.termux/files/home/sms-cola.json';
    this.telefonoLogiCuba = process.env.TELEFONO_LOGICUBA || '+5351234567';

    this.cargarCola();
    this.iniciar();
  }

  /**
   * Inicia los procesos del gateway
   */
  private async iniciar(): Promise<void> {
    // Verificar internet cada 30 segundos
    setInterval(async () => {
      this.online = await this.verificarInternet();

      if (this.online && this.cola.length > 0) {
        console.log(`[Gateway] 🌐 Sincronizando ${this.cola.length} mensajes...`);
        await this.sincronizarCola();
      }
    }, 30000);

    // Procesar SMS nuevos cada 5 segundos
    setInterval(async () => {
      await this.procesarSMSNuevos();
    }, 5000);

    // Verificar mensajes expirados cada minuto
    setInterval(async () => {
      await this.procesarMensajesExpirados();
    }, 60000);
  }

  /**
   * Verifica conexión a internet
   */
  private async verificarInternet(): Promise<boolean> {
    try {
      const controller = new AbortController();
      const timeout = setTimeout(() => controller.abort(), 5000);

      const res = await fetch(`${this.API_URL}/health`, {
        signal: controller.signal,
      });

      clearTimeout(timeout);
      return res.ok;
    } catch {
      return false;
    }
  }

  /**
   * Lee SMS recientes del teléfono
   */
  private async leerSMSRecientes(): Promise<SMS[]> {
    // Implementación específica para Termux:API
    // Esta es una versión placeholder
    try {
      // En producción, usar: await termuxSMS.read({ limit: 50 }));
      return [];
    } catch (error) {
      console.error('[Gateway] Error leyendo SMS:', error);
      return [];
    }
  }

  /**
   * Envía SMS usando Termux:API
   */
  private async enviarSMS(to: string, message: string): Promise<boolean> {
    try {
      // En producción, usar: await termuxSMS.send(to, message);
      console.log(`[Gateway] 📤 SMS → ${to}: ${message.substring(0, 50)}...`);
      return true;
    } catch (error) {
      console.error('[Gateway] Error enviando SMS:', error);
      return false;
    }
  }

  /**
   * Procesa SMS nuevos recibidos
   */
  private async procesarSMSNuevos(): Promise<void> {
    const mensajes = await this.leerSMSRecientes();

    for (const msg of mensajes) {
      // Ignorar SMS del banco (los procesa sms-pagos.ts)
      if (msg.number === '900' || msg.number === '+53900') {
        continue;
      }

      // Ignorar SMS enviados por nosotros
      if (msg.number === this.telefonoLogiCuba) {
        continue;
      }

      if (this.online) {
        await this.enviarAOracle(msg);
      } else {
        await this.agregarACola(msg);
        await this.enviarSMS(
          msg.number,
          '✅ Recibido. Será procesado cuando haya conexión.'
        );
      }
    }
  }

  /**
   * Agrega mensaje a la cola offline
   */
  private async agregarACola(msg: SMS): Promise<void> {
    const id = `${Date.now()}-${Math.random().toString(36).substring(2, 6)}`;

    const item: ColaItem = {
      id,
      phone: msg.number,
      message: msg.body,
      timestamp: msg.date,
      attempts: 0,
      lastAttempt: new Date().toISOString(),
      status: 'pending',
    };

    // Verificar si ya existe mensaje similar
    const existe = this.cola.find(
      c => c.phone === msg.number &&
           c.message === msg.body &&
           Date.now() - new Date(c.timestamp).getTime() < 60000 // 1 minuto
    );

    if (!existe) {
      this.cola.push(item);
      await this.guardarCola();
      console.log(`[Gateway] 📥 Mensaje agregado a cola: ${id}`);
    }
  }

  /**
   * Sincroniza cola con Oracle
   */
  private async sincronizarCola(): Promise<void> {
    const pendientes = this.cola.filter(item => item.status === 'pending');

    for (const item of pendientes) {
      try {
        const res = await fetch(`${this.API_URL}/bot/sms`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-API-Key': process.env.INTERNAL_API_KEY || '',
          },
          body: JSON.stringify({
            phone: item.phone,
            message: item.message,
            queued_at: item.timestamp,
            queue_id: item.id,
          }),
          signal: AbortSignal.timeout(15000),
        });

        if (res.ok) {
          const data = await res.json();

          if (data.response) {
            await this.enviarSMS(item.phone, data.response);
          }

          item.status = 'sent';
          console.log(`[Gateway] ✅ Mensaje sincronizado: ${item.id}`);
        }
      } catch (error) {
        item.attempts++;
        item.lastAttempt = new Date().toISOString();

        if (item.attempts >= this.MAX_REINTENTOS) {
          await this.enviarSMS(
            item.phone,
            '❌ No pudimos procesar tu mensaje. Intenta más tarde.'
          );
          item.status = 'failed';
          console.error(`[Gateway] ❌ Mensaje fallido: ${item.id}`);
        }
      }
    }

    // Limpiar mensajes procesados y fallidos
    this.cola = this.cola.filter(item => item.status === 'pending');
    await this.guardarCola();
  }

  /**
   * Procesa mensajes expirados
   */
  private async procesarMensajesExpirados(): Promise<void> {
    // Encontrar reservas pendientes que ya expiraron
    // y notificar a los usuarios
    try {
      await fetch(`${this.API_URL}/reservas/procesar-expiradas`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': process.env.INTERNAL_API_KEY || '',
        },
      });
    } catch (error) {
      console.error('[Gateway] Error procesando expiradas:', error);
    }
  }

  /**
   * Envía mensaje al servidor Oracle
   */
  private async enviarAOracle(msg: SMS): Promise<void> {
    try {
      const res = await fetch(`${this.API_URL}/bot/sms`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': process.env.INTERNAL_API_KEY || '',
        },
        body: JSON.stringify({
          phone: msg.number,
          message: msg.body,
          received_at: msg.date,
        }),
      });

      if (res.ok) {
        const data = await res.json();
        if (data.response) {
          await this.enviarSMS(msg.number, data.response);
        }
      }
    } catch (error) {
      console.error('[Gateway] Error enviando a Oracle:', error);
      await this.agregarACola(msg);
    }
  }

  /**
   * Guarda cola en archivo
   */
  private async guardarCola(): Promise<void> {
    try {
      // En Node.js: fs.writeFileSync(this.QUEUE_FILE, JSON.stringify(this.cola));
      // En Bun: Bun.write(this.QUEUE_FILE, JSON.stringify(this.cola));
      console.log(`[Gateway] 💾 Cola guardada: ${this.cola.length} items`);
    } catch (error) {
      console.error('[Gateway] Error guardando cola:', error);
    }
  }

  /**
   * Carga cola desde archivo
   */
  private async cargarCola(): Promise<void> {
    try {
      // En Node.js: const data = fs.readFileSync(this.QUEUE_FILE, 'utf8');
      // En Bun: const data = Bun.file(this.QUEUE_FILE);
      // this.cola = JSON.parse(data);

      console.log(`[Gateway] 📂 Cola cargada: ${this.cola.length} items`);
    } catch {
      this.cola = [];
      console.log('[Gateway] 📂 Cola inicializada vacía');
    }
  }
}

// Iniciar gateway
new SMSGateway();
```

---

## 12. Despliegue e Infraestructura

### 12.1 Docker Compose

```yaml
version: '3.8'

services:
  # CAPA DE DATOS
  postgres:
    image: postgres:16.2-alpine
    container_name: logicuba-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-logicuba}
      POSTGRES_USER: ${POSTGRES_USER:-logicuba_admin}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD required}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.2-alpine
    container_name: logicuba-redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:?REDIS_PASSWORD required}
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "127.0.0.1:6379:6379"

  # COMUNICACIONES
  evolution-api:
    image: atendai/evolution-api:v1.8.2
    container_name: logicuba-evolution
    restart: unless-stopped
    environment:
      AUTHENTICATION_API_KEY: ${EVOLUTION_API_KEY:?EVOLUTION_API_KEY required}
      DATABASE_ENABLED: "true"
      DATABASE_PROVIDER: "postgresql"
      DATABASE_CONNECTION_URI: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
      CACHE_REDIS_ENABLED: "true"
      CACHE_REDIS_URI: redis://:${REDIS_PASSWORD}@redis:6379
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  # BACKEND API
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: logicuba-api
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: "3002"
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET required}
      ENCRYPTION_KEY: ${ENCRYPTION_KEY:?ENCRYPTION_KEY required}
      EVOLUTION_API_URL: http://evolution-api:8080
      EVOLUTION_API_KEY: ${EVOLUTION_API_KEY}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
      API_URL: ${API_URL}
      FRONTEND_URL: ${FRONTEND_URL}
      # EnZona
      ENZONA_CONSUMER_KEY: ${ENZONA_CONSUMER_KEY}
      ENZONA_CONSUMER_SECRET: ${ENZONA_CONSUMER_SECRET}
      ENZONA_MERCHANT_UUID: ${ENZONA_MERCHANT_UUID}
      ENZONA_WEBHOOK_SECRET: ${ENZONA_WEBHOOK_SECRET}
      # Upstash
      UPSTASH_REDIS_REST_URL: ${UPSTASH_REDIS_REST_URL}
      UPSTASH_REDIS_REST_TOKEN: ${UPSTASH_REDIS_REST_TOKEN}
    ports:
      - "127.0.0.1:3002:3002"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  # TELEGRAM BOT
  telegram-bot:
    build:
      context: ./telegram-bot
      dockerfile: Dockerfile
    container_name: logicuba-telegram
    restart: unless-stopped
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      API_URL: http://api:3002
      INTERNAL_API_KEY: ${INTERNAL_API_KEY}
    depends_on:
      - api

  # TYPEBOT
  typebot-builder:
    image: baptistearno/typebot-builder:2.27.0
    container_name: logicuba-typebot-builder
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      NEXTAUTH_URL: ${TYPEBOT_BUILDER_URL}
      ENCRYPTION_SECRET: ${TYPEBOT_ENCRYPTION_SECRET}
    ports:
      - "127.0.0.1:3000:3000"

  # MONITOREO
  uptime-kuma:
    image: louislam/uptime-kuma:1.23.13-alpine
    container_name: logicuba-uptime
    restart: unless-stopped
    volumes:
      - uptime_data:/app/data
    ports:
      - "127.0.0.1:3003:3001"

  # TÚNEL
  cloudflared:
    image: cloudflare/cloudflared:2024.2.1
    container_name: logicuba-tunnel
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    depends_on:
      - api

volumes:
  postgres_data:
  redis_data:
  uptime_data:

networks:
  default:
    name: logicuba-network
```

### 12.2 Costos de Operación

| Servicio | Costo Mensual | Notas |
|----------|--------------|-------|
| Oracle Cloud Free Tier | $0 | 4 CPU, 24 GB RAM, 200 GB storage |
| Evolution API | $0 | Self-hosted |
| Telegraf | $0 | Open source |
| Redis | $0 | Incluido en Oracle |
| PostgreSQL | $0 | Incluido en Oracle |
| Typebot | $0 | Self-hosted |
| Cloudflare Tunnel | $0 | Plan gratuito |
| Uptime Kuma | $0 | Self-hosted |
| EnZona | $0 | Sin costo de integración |
| SMS datos (Xiaomi) | $2-5 | Datos móviles para gateway |
| Dominio | $0 | Cloudflare free |
| **Total** | **$2-5/mes** | |

---

## 13. Glosario

| Término | Definición |
|---------|------------|
| **API REST** | Interfaz de programación que usa el protocolo HTTP para comunicación |
| **Baileys** | Librería para conectarse a WhatsApp Web |
| **Canal** | Medio de comunicación (WhatsApp, Telegram, SMS) |
| **Cloudflare Tunnel** | Servicio que expone servicios locales sin IP pública |
| **CUP** | Peso Cubano, moneda nacional de Cuba |
| **EnZona** | Plataforma de pagos electrónicos de Cuba |
| **Evolution API** | API para integrar WhatsApp Business |
| **Gateway** | Punto de entrada/salida para comunicaciones |
| **GDS** | Global Distribution System (sistema de distribución global) |
| **JWT** | JSON Web Token, estándar para autenticación |
| **Notificación proactiva** | Mensaje enviado sin que el usuario lo solicite |
| **OAuth2** | Protocolo de autorización estándar |
| **Oracle Cloud Free Tier** | Servicios gratuitos de Oracle Cloud |
| **Rate limiting** | Técnica para limitar la cantidad de requests |
| **Redis** | Base de datos en memoria para caché |
| **SIM** | Subscriber Identity Module (tarjeta SIM del móvil) |
| **Sitrans** | Empresa de Servicios de Información del Transporte |
| **SMS Gateway** | Dispositivo/software que envía SMS reales |
| **Termux** | Emulador de terminal para Android |
| **Transfermóvil** | Aplicación de pagos móviles de ETECSA/BANDEC |
| **UUID** | Identificador único universal |
| **WETRANSP** | Sistema GDS cubano para gestión de transporte |
| **XETID** | Empresa de Tecnologías de la Información para la Defensa |

---

## Anexos

### Anexo A: Códigos de Provincia para SMS

| Código | Provincia |
|--------|-----------|
| LH | La Habana |
| PR | Pinar del Río |
| AR | Artemisa |
| MJ | Mayabeque |
| MT | Matanzas |
| CF | Cienfuegos |
| VC | Villa Clara |
| SS | Sancti Spíritus |
| CA | Ciego de Ávila |
| CM | Camagüey |
| LT | Las Tunas |
| HO | Holguín |
| GR | Granma |
| SC | Santiago de Cuba |
| GT | Guantánamo |
| IJ | Isla de la Juventud |

### Anexo B: Estados de Reserva

| Estado | Descripción |
|--------|-------------|
| `pendiente_pago` | Reserva creada, esperando pago |
| `pagada` | Pago recibido, en verificación |
| `confirmada` | Pago verificado, reserva activa |
| `en_curso` | Viaje en progreso |
| `completada` | Viaje finalizado |
| `cancelada` | Reserva cancelada |
| `no_show` | Pasajero no se presentó |

### Anexo C: Estados de Viaje

| Estado | Descripción |
|--------|-------------|
| `activo` | Viaje publicado y disponible |
| `lleno` | Todos los asientos reservados |
| `en_curso` | Viaje en progreso |
| `completado` | Viaje finalizado |
| `cancelado` | Viaje cancelado por transportista |

---

**Documento generado:** Abril 2026
**Versión:** 5.0
**Estado:** Final
**Autor:** LogiCuba Development Team
**Licencia:** Propietaria