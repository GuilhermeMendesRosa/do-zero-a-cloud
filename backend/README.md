# Backend

API Spring Boot do workshop. O roteiro completo está no [README principal](../README.md).

## Executar

Requer apenas Java 17. Esta branch usa H2 em memória, então não é preciso instalar nem configurar um banco local.

```bash
./mvnw spring-boot:run
```

A API usa `http://localhost:8090/api/v1` e o health check está em `/actuator/health`. Os dados existem apenas enquanto a API está em execução e são apagados ao reiniciá-la.
