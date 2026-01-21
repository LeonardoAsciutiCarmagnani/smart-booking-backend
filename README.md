# SmartBooking

## Plataforma de agendamentos com foco em **fluxos reais de produto** e **confiabilidade**, permitindo que prestadores configurem disponibilidade e serviços, e clientes realizem agendamentos em uma página pública.

O projeto foi desenvolvido com arquitetura orientada a **resiliência**, utilizando **Redis** para lock/cache e **BullMQ** para automações assíncronas (expiração e lembretes).

> _“Arquitetura resiliente é aquela que assume falhas como parte do ambiente e implementa mecanismos como idempotência, locks, retries e processamento assíncrono para garantir consistência e continuidade operacional.”_

---

## Visão Geral

### Perfis do sistema

- **Prestador (Admin/Provider)**: configura serviços, disponibilidade, bloqueios e gerencia agendamentos.
- **Cliente (Público)**: visualiza horários disponíveis, cria agendamento e confirma.

### Fluxo principal (booking lifecycle)

1. Cliente seleciona serviço + data + horário disponível
2. Sistema cria um agendamento **PENDING** (reserva temporária)
3. Se o cliente confirmar dentro do prazo → status vira **CONFIRMED**
4. Se não confirmar → expira automaticamente → status vira **EXPIRED**

---

## Principais Features

### Público (Cliente)

- Página pública do prestador (`/p/:providerSlug`)
- Seleção de serviço e horários disponíveis
- Criação de agendamento com status **PENDING**
- Confirmação do agendamento (**PENDING → CONFIRMED**)
- Tela de status do agendamento (`/booking/:bookingId`)

### Painel do Prestador (Admin)

- CRUD de serviços (duração, buffers, status ativo)
- Configuração de disponibilidade semanal
- Bloqueios (dias/intervalos específicos)
- Visualização e gestão de agendamentos por status
- Timeline de eventos do agendamento (auditoria)

---

## Diferenciais Técnicos (Nível Produção)

### 1) Idempotência (Idempotency Key)

Operações críticas como criação de agendamento utilizam **Idempotency-Key** para evitar duplicidade em casos como:

- double click no botão
- reenvio por instabilidade de rede
- retries automáticos do cliente

**Resultado:** a mesma requisição não cria 2 agendamentos.

---

### 2) Concorrência e Consistência (Redis Lock)

Para impedir que duas pessoas reservem o mesmo horário simultaneamente, o backend aplica um **lock por slot** usando Redis:

- `lock:{providerId}:{startAtISO}` com TTL curto (ex.: 10s)

**Resultado:** elimina conflito de concorrência no momento da criação do booking.

---

### 3) Automação com BullMQ (Jobs Delayed)

O sistema executa tarefas assíncronas para garantir consistência e reduzir carga no request principal.

Exemplos:

- Expirar automaticamente bookings `PENDING` após X minutos
- Agendar lembretes (24h / 1h antes do horário) _(opcional/Plus)_

---

### 4) Cache de Slots (Redis)

O endpoint de slots disponíveis pode ser custoso (regras + bloqueios + bookings do dia).Para otimizar, o sistema utiliza cache por data/serviço:

- `slots:{providerId}:{serviceId}:{YYYY-MM-DD}` com TTL curto

Cache é invalidado quando:

- cria/expira/confirma/cancela booking
- altera disponibilidade
- cria/remove bloqueio

---

### 5) Auditoria (Booking Events)

Toda transição importante registra um evento em histórico:

- created
- confirmed
- expired
- cancelled
- reminder_sent

**Resultado:** rastreabilidade completa e fácil debugging.

---

## Stack e Tecnologias

### Front-end

- React + Vite
- TypeScript
- Tailwind CSS
- Shadcn/UI
- Zustand (ou Context API)
- Zod (validação de formulários)

### Back-end

- Node.js + Express (em Firebase Functions)
- TypeScript
- Zod (validação de payloads)
- Firestore (persistência)

### Infra / Assíncrono

- Firebase Hosting (front)
- Firebase Functions (API)
- Redis (cache + lock)
- BullMQ (jobs e workers)

---

## Arquitetura (Resumo)

- **Firestore** é a fonte de verdade (bookings, services, availability)
- **Redis** é usado para:
  - lock por slot (concorrência)
  - cache de slots (performance)
- **BullMQ** executa automações:
  - expiração de booking pendente
  - lembretes (opcional)

---

## Status do Booking (State Machine)

Estados:

- `PENDING`: reservado temporariamente, aguardando confirmação
- `CONFIRMED`: confirmado e válido
- `EXPIRED`: expirou automaticamente por falta de confirmação
- `CANCELLED`: cancelado manualmente
- `NO_SHOW`: cliente não compareceu

Transições permitidas (exemplo):

- `PENDING → CONFIRMED | EXPIRED | CANCELLED`
- `CONFIRMED → CANCELLED | NO_SHOW`
