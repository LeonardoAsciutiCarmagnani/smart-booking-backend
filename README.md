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

## Stack e Tecnologias

### Front-end

- React + Vite
- TypeScript
- Tailwind CSS
- Shadcn/UI
- Context API
- Zod

### Back-end

- Node.js + Express (Firebase Functions)
- TypeScript
- Zod 
- Firestore

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
  - lembretes
