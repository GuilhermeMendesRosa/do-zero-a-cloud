# Backend — Implementação final

API Spring Boot completa do workshop, com os checkpoints de quadros, colunas e
tarefas implementados. O projeto inicial para a atividade está em
[backend](../backend/), e o roteiro está no [README principal](../README.md).

## Executar

Requer Java 17 e PostgreSQL com o banco `kanban`.

```bash
./mvnw spring-boot:run
```

Variáveis opcionais:

```text
DB_URL=jdbc:postgresql://localhost:5432/kanban
DB_USERNAME=postgres
DB_PASSWORD=postgres
```

A API usa `http://localhost:8090/api/v1` e o health check está em `/actuator/health`.
