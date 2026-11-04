# api/

Contrato da API do projeto.

## O que guardar aqui

- [`rotas.md`](rotas.md): planejamento das rotas (objetivo, entrada, validações e status codes), escrito antes da implementação.
- `openapi.yaml`: descrição de cada endpoint (rota, método, parâmetros, respostas e erros).

O contrato é escrito antes ou junto com o backend. Ele facilita testar a API e integrar o frontend sem depender de conversa.

## Observação

A API externa do RAWG (busca de jogos) não é descrita aqui. O que fica aqui é a API do próprio STAYVILE. As decisões sobre como usamos o RAWG vão em um ADR.
