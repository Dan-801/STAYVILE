# Rotas da API

Planejamento das rotas da API da STAYVILE antes da implementação.

O objetivo é deixar claro o que cada rota recebe, o que valida e o que responde, para servir de referência ao backend. Tudo aqui segue a [modelagem de dados](../architecture/modelagem-dados.md). Quando as rotas forem implementadas, o contrato detalhado vai para o `openapi.yaml`.

## Entidades da entrega e onde estão na STAYVILE

A STAYVILE é uma biblioteca de jogos, então algumas entidades do BookHub têm outro nome aqui.

| Entidade da entrega | Na STAYVILE | Recurso | Tabela |
|---|---|---|---|
| Usuários | Usuários | `/users`, `/auth` | `users` |
| Obras | Jogos | `/games` | `games` |
| Categorias | Gêneros | `/genres` | `genres`, `game_genres` |
| Empresas | Fora do MVP | — | — |
| Relação entre usuários e empresas | Fora do MVP | — | — |
| *(além da entrega)* | Empréstimos | `/loans` | `loans` |

- **Gêneros entram no MVP** por causa desta entrega. A modelagem ainda marca `genres` e `game_genres` como v2.
- **Empréstimos** não estão na lista da entrega, mas são o centro do MVP (emprestar, devolver, ver atrasados). Sem eles o backend não fica planejado.
- **Empresas** ficam de fora por enquanto. O motivo e o impacto de incluir depois estão em [Fora do escopo do MVP](#fora-do-escopo-do-mvp).

## Regras gerais

Valem para todas as rotas, salvo quando a rota disser outra coisa.

- **Formato:** corpo da requisição e da resposta em JSON. Datas em ISO 8601, em UTC (ex.: `2026-11-04T14:30:00Z`).
- **IDs:** UUID (ver [ADR 0001](../adr/0001-uuid-ou-serial-nas-chaves.md)). Um ID com formato inválido no caminho retorna `400`.
- **Registro inexistente:** um ID no caminho ou no body que não existe, ou que aponta para um registro desativado (`deleted_at` preenchido), retorna `404`.
- **Campos não aceitos:** campo desconhecido ou que a rota não permite alterar retorna `400`. A API não ignora em silêncio o que não vai usar.
- **Senha:** `password_hash` nunca aparece em resposta.
- **Campos calculados:** `quantity_available` (jogos), `status` (empréstimos) e `is_suspended` (usuários) aparecem nas respostas, mas não são gravados e não são aceitos no body. Ver [Decisões de modelagem](../architecture/modelagem-dados.md#decisões-de-modelagem).
- **Listas:** paginadas com `page` (padrão `1`) e `limit` (padrão `20`, máximo `100`). A resposta vem como `{ "data": [...], "page": 1, "limit": 20, "total": 57 }`.

### Autenticação e permissão

O login usa **sessão guardada no banco**:

- No `POST /auth/login`, o backend cria uma sessão e devolve o identificador dela em um cookie `HttpOnly`, `Secure` e `SameSite=Lax`. O JavaScript da página não consegue ler esse cookie, o que dificulta o roubo da sessão por script.
- A cada requisição, o backend procura a sessão do cookie. Sessão inexistente, expirada ou encerrada conta como sem login válido.
- No `POST /auth/logout`, a sessão é apagada no banco e o cookie é limpo.

Isso pede a tabela `sessions` ligada a `users` (1:N), já prevista nas pendências da modelagem.

Cada rota indica quem pode chamá-la:

| Nível | Quem |
|---|---|
| Pública | Qualquer pessoa, sem login |
| Membro | Qualquer usuário logado (`member` ou `admin`) |
| Dono ou admin | O usuário dono do registro, ou um `admin` |
| Admin | Só usuários com `role = admin` |

Em todas as rotas: **sem login válido → `401`; logado, mas sem permissão → `403`.**

### Status codes usados

| Status | Quando |
|---|---|
| `200 OK` | Consulta ou alteração feita, com corpo na resposta |
| `201 Created` | Registro criado |
| `204 No Content` | Operação feita, sem corpo na resposta (desativar, logout) |
| `400 Bad Request` | Campo obrigatório faltando, formato inválido ou campo não aceito |
| `401 Unauthorized` | Rota exige login e não há login válido, ou credenciais erradas no login |
| `403 Forbidden` | Usuário logado, mas sem permissão para a operação |
| `404 Not Found` | Registro não existe ou está desativado |
| `409 Conflict` | Dados válidos, mas que contrariam uma regra do negócio (e-mail repetido, jogo sem cópia disponível) |
| `502 Bad Gateway` | O RAWG não respondeu ou respondeu com erro |
| `500 Internal Server Error` | Erro inesperado no servidor |

### Formato de erro

Toda resposta de erro segue o mesmo formato:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Dados inválidos.",
    "details": [
      { "field": "email", "message": "Formato de e-mail inválido." }
    ]
  }
}
```

`details` só aparece em erros de validação (`400`).

## Resumo das rotas

| Método | Caminho | Objetivo | Acesso |
|---|---|---|---|
| `POST` | `/users` | Criar conta | Pública |
| `POST` | `/auth/login` | Entrar | Pública |
| `POST` | `/auth/logout` | Sair | Membro |
| `GET` | `/users/me` | Ver a própria conta | Membro |
| `GET` | `/users` | Listar usuários | Admin |
| `GET` | `/users/:id` | Ver um usuário | Dono ou admin |
| `PATCH` | `/users/:id` | Alterar dados da conta | Dono ou admin |
| `DELETE` | `/users/:id` | Desativar conta | Dono ou admin |
| `GET` | `/rawg/games` | Buscar jogos no RAWG | Admin |
| `GET` | `/games` | Listar jogos do acervo | Pública |
| `GET` | `/games/:id` | Ver a ficha de um jogo | Pública |
| `POST` | `/games` | Cadastrar jogo a partir do RAWG | Admin |
| `PATCH` | `/games/:id` | Alterar quantidade ou valor | Admin |
| `PUT` | `/games/:id/genres` | Definir os gêneros de um jogo | Admin |
| `DELETE` | `/games/:id` | Retirar jogo do acervo | Admin |
| `GET` | `/genres` | Listar gêneros | Pública |
| `GET` | `/genres/:id` | Ver um gênero | Pública |
| `POST` | `/genres` | Cadastrar gênero a partir do RAWG | Admin |
| `PATCH` | `/genres/:id` | Renomear gênero | Admin |
| `DELETE` | `/genres/:id` | Excluir gênero | Admin |
| `POST` | `/loans` | Registrar empréstimo | Membro |
| `GET` | `/loans` | Listar empréstimos | Membro |
| `GET` | `/loans/:id` | Ver um empréstimo | Dono ou admin |
| `POST` | `/loans/:id/return` | Registrar devolução | Dono ou admin |

## Usuários

### `POST /users`

- **Objetivo:** criar uma conta. Toda conta nova é `member`.
- **Acesso:** pública.
- **Body:**

  ```json
  { "name": "Ana Souza", "email": "ana@email.com", "password": "senha-forte-123" }
  ```

- **Validações:**
  - `name`: obrigatório, texto, sem só espaços, até 100 caracteres.
  - `email`: obrigatório, formato de e-mail válido. É gravado em minúsculas e sem espaços nas pontas.
  - `password`: obrigatório, no mínimo 8 caracteres. É gravado só como hash.
  - `role` não é aceito: ninguém escolhe o próprio papel no cadastro.

| Status | Quando |
|---|---|
| `201` | Conta criada. Retorna o usuário (`id`, `name`, `email`, `role`, `created_at`) |
| `400` | Campo obrigatório faltando, formato inválido ou `role` enviado |
| `409` | Já existe conta com esse e-mail, inclusive conta desativada |

### `POST /auth/login`

- **Objetivo:** entrar na conta.
- **Acesso:** pública.
- **Body:** `email` e `password`.
- **Validações:**
  - `email` e `password` obrigatórios.
  - A mensagem de erro é a mesma para e-mail inexistente e senha errada, para não revelar quais e-mails têm conta.
  - Conta desativada não entra.

| Status | Quando |
|---|---|
| `200` | Login feito. Cria a sessão, devolve o cookie de sessão e retorna o usuário |
| `400` | `email` ou `password` faltando |
| `401` | E-mail ou senha errados, ou conta desativada |

### `POST /auth/logout`

- **Objetivo:** encerrar o login atual. Apaga a sessão no banco e limpa o cookie.
- **Acesso:** membro.
- **Entrada:** nenhuma.

| Status | Quando |
|---|---|
| `204` | Logout feito |
| `401` | Não há login válido |

### `GET /users/me`

- **Objetivo:** ver os dados da própria conta. É o que a página usa para saber quem está logado. A resposta traz `is_suspended`, para a página avisar quando o usuário está suspenso (ver [Regras dos empréstimos](#regras-dos-empréstimos)).
- **Acesso:** membro.
- **Entrada:** nenhuma.

| Status | Quando |
|---|---|
| `200` | Retorna o usuário logado |
| `401` | Não há login válido |

### `GET /users`

- **Objetivo:** listar os usuários.
- **Acesso:** admin.
- **Query:**
  - `search` (opcional): busca em `name` e `email`.
  - `role` (opcional): `member` ou `admin`.
  - `page` e `limit`.
- **Validações:** `role`, se enviado, só aceita `member` ou `admin`. Contas desativadas não aparecem.

| Status | Quando |
|---|---|
| `200` | Retorna a lista paginada |
| `400` | Filtro com valor inválido |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |

### `GET /users/:id`

- **Objetivo:** ver um usuário.
- **Acesso:** dono ou admin.
- **Parâmetro:** `id` do usuário.

| Status | Quando |
|---|---|
| `200` | Retorna o usuário |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Um membro tentando ver outro usuário |
| `404` | Usuário não existe ou está desativado |

### `PATCH /users/:id`

- **Objetivo:** alterar dados da conta. Só os campos enviados mudam.
- **Acesso:** dono ou admin.
- **Parâmetro:** `id` do usuário.
- **Body (todos opcionais, pelo menos um):** `name`, `email`, `password`, `current_password`, `role`.
- **Validações:**
  - `name`, `email` e `password` seguem as mesmas regras do `POST /users`.
  - Para trocar a própria senha, o dono precisa enviar também `current_password`, e ela precisa estar correta. Um admin trocando a senha de outro usuário não precisa dela.
  - Quando a senha muda, as outras sessões desse usuário são encerradas. A sessão de quem fez a troca continua ativa.
  - `role` só pode ser alterado por admin, e só aceita `member` ou `admin`.
  - Um admin não pode tirar o próprio papel de admin. Isso evita ficar sem nenhum admin.

| Status | Quando |
|---|---|
| `200` | Dados alterados. Retorna o usuário atualizado |
| `400` | Body vazio, formato inválido, campo não aceito ou `password` sem `current_password` |
| `401` | Não há login válido, ou `current_password` errada |
| `403` | Membro alterando outro usuário, membro enviando `role`, ou admin tirando o próprio papel de admin |
| `404` | Usuário não existe ou está desativado |
| `409` | O novo e-mail já pertence a outra conta |

### `DELETE /users/:id`

- **Objetivo:** desativar a conta. Preenche `deleted_at` e não apaga o registro, porque os empréstimos antigos continuam ligados a ele. Todas as sessões do usuário são encerradas.
- **Acesso:** dono ou admin.
- **Parâmetro:** `id` do usuário.
- **Validações:** não desativa conta com empréstimo em aberto (`returned_at` vazio).

| Status | Quando |
|---|---|
| `204` | Conta desativada |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Membro tentando desativar outro usuário |
| `404` | Usuário não existe ou já está desativado |
| `409` | O usuário tem empréstimo em aberto |

## Obras (jogos)

Os dados de catálogo do jogo (`name`, `released_at`, `cover_url` e gêneros) vêm sempre do RAWG, buscados pelo backend. O body só traz o que é da STAYVILE: quantidade de cópias e valor do aluguel. Isso evita ficha com nome digitado errado e mantém a chave do RAWG fora do frontend.

### `GET /rawg/games`

- **Objetivo:** buscar jogos no RAWG pelo nome, para o admin escolher qual cadastrar. O backend repassa a busca ao RAWG com a chave guardada no servidor.
- **Acesso:** admin.
- **Query:** `search` (obrigatório), `page`.
- **Validações:** `search` obrigatório, com pelo menos 2 caracteres.
- **Resposta:** para cada jogo, `rawg_id`, `name`, `released_at`, `cover_url` e `in_library` (se o jogo já está no acervo).

| Status | Quando |
|---|---|
| `200` | Retorna os resultados do RAWG (pode ser lista vazia) |
| `400` | `search` faltando ou curto demais |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `502` | O RAWG não respondeu ou respondeu com erro |

### `GET /games`

- **Objetivo:** listar os jogos do acervo.
- **Acesso:** pública.
- **Query:**
  - `search` (opcional): busca no nome.
  - `genre_id` (opcional): só jogos daquele gênero.
  - `available` (opcional): `true` mostra só jogos com cópia disponível.
  - `page` e `limit`.
- **Validações:** `genre_id` precisa ser UUID; `available` só aceita `true` ou `false`. Jogos retirados do acervo não aparecem.
- **Resposta:** cada jogo traz `quantity_available` calculado.

| Status | Quando |
|---|---|
| `200` | Retorna a lista paginada |
| `400` | Filtro com valor inválido |

### `GET /games/:id`

- **Objetivo:** ver a ficha completa de um jogo.
- **Acesso:** pública.
- **Parâmetro:** `id` do jogo.
- **Resposta:** dados do jogo, `quantity_available` e a lista de gêneros.

| Status | Quando |
|---|---|
| `200` | Retorna a ficha |
| `400` | `id` com formato inválido |
| `404` | Jogo não existe ou foi retirado do acervo |

### `POST /games`

- **Objetivo:** cadastrar um jogo no acervo a partir do RAWG.
- **Acesso:** admin.
- **Body:**

  ```json
  { "rawg_id": 3498, "quantity_total": 3, "rental_price": 12.50 }
  ```

- **Validações:**
  - `rawg_id`: obrigatório, inteiro maior que 0, e precisa existir no RAWG.
  - `quantity_total`: obrigatório, inteiro maior ou igual a 0.
  - `rental_price`: obrigatório, número maior ou igual a 0, com até 2 casas decimais.
  - `name`, `released_at`, `cover_url` e gêneros não são aceitos: vêm do RAWG.
  - O `rawg_id` não pode já estar cadastrado, nem em jogo retirado do acervo.
- **O que o backend faz:** busca a ficha no RAWG, grava o jogo e liga os gêneros. Gênero que ainda não existe em `genres` é criado na hora.

| Status | Quando |
|---|---|
| `201` | Jogo cadastrado. Retorna a ficha com os gêneros |
| `400` | Campo obrigatório faltando, valor inválido ou campo de catálogo enviado |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | `rawg_id` não existe no RAWG |
| `409` | Jogo com esse `rawg_id` já cadastrado |
| `502` | O RAWG não respondeu ou respondeu com erro |

### `PATCH /games/:id`

- **Objetivo:** alterar a quantidade de cópias ou o valor do aluguel.
- **Acesso:** admin.
- **Parâmetro:** `id` do jogo.
- **Body (opcionais, pelo menos um):** `quantity_total`, `rental_price`.
- **Validações:**
  - Mesmas regras do `POST /games` para cada campo.
  - `quantity_total` não pode ficar menor que o número de cópias emprestadas agora.
  - Campos de catálogo e `rawg_id` não são aceitos.
  - Mudar `rental_price` não altera empréstimos já feitos: cada empréstimo guarda o valor da época.

| Status | Quando |
|---|---|
| `200` | Jogo alterado. Retorna a ficha atualizada |
| `400` | Body vazio, valor inválido ou campo não aceito |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | Jogo não existe ou foi retirado do acervo |
| `409` | `quantity_total` menor que as cópias emprestadas agora |

### `PUT /games/:id/genres`

- **Objetivo:** definir a lista de gêneros de um jogo. A lista enviada substitui a anterior. É a rota que mantém a relação N:N em `game_genres`.
- **Acesso:** admin.
- **Parâmetro:** `id` do jogo.
- **Body:**

  ```json
  { "genre_ids": ["9b2f...", "c41a..."] }
  ```

- **Validações:**
  - `genre_ids`: obrigatório, lista de UUIDs, sem repetição. Lista vazia é aceita e deixa o jogo sem gênero.
  - Todos os gêneros da lista precisam existir.

| Status | Quando |
|---|---|
| `200` | Gêneros definidos. Retorna o jogo com a nova lista de gêneros |
| `400` | `genre_ids` faltando, fora do formato ou com repetição |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | Jogo não existe, ou algum gênero da lista não existe |

### `DELETE /games/:id`

- **Objetivo:** retirar o jogo do acervo. Preenche `deleted_at` e mantém o histórico de empréstimos.
- **Acesso:** admin.
- **Parâmetro:** `id` do jogo.
- **Validações:** não retira jogo com cópia emprestada agora.

| Status | Quando |
|---|---|
| `204` | Jogo retirado do acervo |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | Jogo não existe ou já foi retirado |
| `409` | O jogo tem empréstimo em aberto |

## Categorias (gêneros)

Como os jogos, os gêneros vêm do RAWG (`rawg_id` é obrigatório e único). Na maioria dos casos eles são criados sozinhos no `POST /games`. As rotas abaixo servem para consultar e fazer ajustes.

### `GET /genres`

- **Objetivo:** listar os gêneros, por exemplo para montar o filtro da página.
- **Acesso:** pública.
- **Entrada:** nenhuma. Sem paginação, porque são poucos registros.
- **Resposta:** cada gênero com `id`, `name` e quantos jogos do acervo ele tem.

| Status | Quando |
|---|---|
| `200` | Retorna a lista, em ordem alfabética |

### `GET /genres/:id`

- **Objetivo:** ver um gênero. Os jogos do gênero ficam em `GET /games?genre_id=...`.
- **Acesso:** pública.
- **Parâmetro:** `id` do gênero.

| Status | Quando |
|---|---|
| `200` | Retorna o gênero |
| `400` | `id` com formato inválido |
| `404` | Gênero não existe |

### `POST /genres`

- **Objetivo:** cadastrar um gênero do RAWG que ainda não veio junto com nenhum jogo.
- **Acesso:** admin.
- **Body:** `{ "rawg_id": 4 }`
- **Validações:**
  - `rawg_id`: obrigatório, inteiro maior que 0, e precisa existir no RAWG.
  - `name` não é aceito: vem do RAWG.
  - O `rawg_id` não pode já estar cadastrado.

| Status | Quando |
|---|---|
| `201` | Gênero cadastrado. Retorna o gênero |
| `400` | `rawg_id` faltando ou inválido, ou campo não aceito |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | `rawg_id` não existe no RAWG |
| `409` | Gênero com esse `rawg_id` já cadastrado |
| `502` | O RAWG não respondeu ou respondeu com erro |

### `PATCH /genres/:id`

- **Objetivo:** mudar o nome exibido de um gênero, por exemplo para traduzir "Action" para "Ação".
- **Acesso:** admin.
- **Parâmetro:** `id` do gênero.
- **Body:** `{ "name": "Ação" }`
- **Validações:** `name` obrigatório, sem só espaços, até 50 caracteres. `rawg_id` não pode ser alterado.

| Status | Quando |
|---|---|
| `200` | Gênero renomeado. Retorna o gênero |
| `400` | `name` faltando ou inválido, ou campo não aceito |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | Gênero não existe |

### `DELETE /genres/:id`

- **Objetivo:** excluir um gênero. `genres` não tem `deleted_at`, então a exclusão é definitiva.
- **Acesso:** admin.
- **Parâmetro:** `id` do gênero.
- **Validações:** não exclui gênero ligado a algum jogo. Primeiro é preciso tirar o gênero dos jogos com `PUT /games/:id/genres`.

| Status | Quando |
|---|---|
| `204` | Gênero excluído |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Usuário logado não é admin |
| `404` | Gênero não existe |
| `409` | O gênero está ligado a pelo menos um jogo |

## Empréstimos

Não existe rota para apagar ou editar empréstimo. Como diz a modelagem, empréstimo é histórico: um lançamento errado se corrige com a devolução ou com um novo registro.

### Regras dos empréstimos

- **Prazo:** no máximo 72 horas depois de `loaned_at`.
- **Uma cópia por jogo:** o usuário só pode ter um empréstimo em aberto de cada jogo. Depois de devolver, pode pegar o mesmo jogo de novo, mesmo antes de completar as 72 horas.
- **Limite mensal:** no máximo 20 empréstimos por usuário por mês, contando pelo horário de Brasília.
- **Atraso:** passou do `due_at` sem devolução, o empréstimo fica `overdue`. A cópia continua presa até a devolução.
- **Suspensão:** enquanto tiver algum empréstimo atrasado, o usuário fica suspenso e não pode pegar nenhum jogo. Ele continua podendo entrar, consultar o acervo, ver os próprios empréstimos e devolver o jogo. A suspensão acaba quando todos os atrasados forem devolvidos.
- Atraso e suspensão são calculados a partir dos empréstimos, não gravados. É a mesma lógica da modelagem: não existe coluna de status para atualizar todo dia.

### `POST /loans`

- **Objetivo:** registrar o empréstimo de uma cópia de um jogo.
- **Acesso:** membro. O membro empresta para si mesmo. O admin pode registrar para qualquer usuário.
- **Body:**

  ```json
  { "game_id": "5e1c...", "due_at": "2026-11-07T14:30:00Z", "user_id": "a7d0..." }
  ```

- **Validações:**
  - `game_id`: obrigatório, UUID de um jogo do acervo.
  - `due_at`: opcional. **Prazo máximo de 72 horas** depois de `loaned_at`. Se não vier, o prazo é exatamente esse máximo. Se vier, precisa estar no futuro e dentro das 72 horas.
  - `user_id`: opcional. Se não vier, o empréstimo é do usuário logado. Membro só pode enviar o próprio `id`.
  - O jogo precisa ter cópia disponível (`quantity_available` maior que 0).
  - **O usuário não pode estar suspenso**, ou seja, não pode ter nenhum empréstimo atrasado.
  - **O usuário não pode ter outro empréstimo em aberto do mesmo jogo.**
  - **Limite de 20 empréstimos por usuário por mês.** Contam os empréstimos com `loaned_at` dentro do mês atual, devolvidos ou não. O mês segue o horário de Brasília (`America/Sao_Paulo`): a contagem recomeça do zero à 0h do dia 1. Empréstimo registrado por um admin conta para o usuário que pegou o jogo.
  - As regras valem também quando é um admin que registra o empréstimo para o usuário.
  - `loaned_at`, `returned_at` e `rental_price` não são aceitos. O backend grava `loaned_at` com a hora atual e copia `rental_price` do jogo.

| Status | Quando |
|---|---|
| `201` | Empréstimo registrado. Retorna o empréstimo com `status = open` |
| `400` | Campo obrigatório faltando, campo não aceito, ou `due_at` no passado ou mais de 72 horas depois do empréstimo |
| `401` | Não há login válido |
| `403` | Membro tentando registrar empréstimo para outro usuário, ou usuário suspenso por atraso (`code: USER_SUSPENDED`) |
| `404` | Jogo ou usuário não existe, ou está desativado |
| `409` | O jogo não tem cópia disponível (`code: NO_COPY_AVAILABLE`), o usuário já tem esse jogo emprestado (`code: GAME_ALREADY_LOANED`) ou já fez 20 empréstimos no mês (`code: MONTHLY_LOAN_LIMIT`) |

### `GET /loans`

- **Objetivo:** listar empréstimos. É a rota da tela de status e da lista de atrasados.
- **Acesso:** membro. O membro vê só os próprios empréstimos. O admin vê todos.
- **Query:**
  - `status` (opcional): `open`, `overdue` ou `returned`. Atrasados = `?status=overdue`.
  - `user_id` (opcional, só admin).
  - `game_id` (opcional).
  - `page` e `limit`.
- **Validações:** `status` só aceita os três valores. `user_id` e `game_id` precisam ser UUID.
- **Resposta:** cada empréstimo com `status` calculado (`overdue` quando `returned_at` está vazio e `due_at` já passou), e com o nome do jogo e do usuário. Ordem: `loaned_at` do mais recente para o mais antigo.

| Status | Quando |
|---|---|
| `200` | Retorna a lista paginada |
| `400` | Filtro com valor inválido |
| `401` | Não há login válido |
| `403` | Membro filtrando por `user_id` de outro usuário |

### `GET /loans/:id`

- **Objetivo:** ver um empréstimo.
- **Acesso:** dono ou admin.
- **Parâmetro:** `id` do empréstimo.

| Status | Quando |
|---|---|
| `200` | Retorna o empréstimo com `status` calculado |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Membro tentando ver empréstimo de outro usuário |
| `404` | Empréstimo não existe |

### `POST /loans/:id/return`

- **Objetivo:** registrar a devolução. Grava `returned_at` com a hora atual e libera a cópia.
- **Acesso:** dono ou admin.
- **Parâmetro:** `id` do empréstimo.
- **Entrada:** nenhuma no body.
- **Validações:** o empréstimo não pode já estar devolvido. Devolução com atraso é aceita normalmente; o atraso fica visível pela comparação entre `returned_at` e `due_at`. Se era o último empréstimo atrasado do usuário, a suspensão acaba na hora.

| Status | Quando |
|---|---|
| `200` | Devolução registrada. Retorna o empréstimo com `status = returned` |
| `400` | `id` com formato inválido |
| `401` | Não há login válido |
| `403` | Membro tentando devolver empréstimo de outro usuário |
| `404` | Empréstimo não existe |
| `409` | Empréstimo já devolvido |

## Fora do escopo do MVP

### Empresas e relação entre usuários e empresas

A entrega pede rotas para empresas e para a relação entre usuários e empresas. Elas ficam de fora porque a modelagem da STAYVILE não tem essas entidades: hoje é uma única biblioteca, e o papel do usuário (`member` ou `admin`) fica no próprio `users`. Escrever rotas para tabelas que não existem deixaria este documento em conflito com a modelagem.

Se as empresas entrarem depois, o impacto depende do papel que elas tiverem:

- **Empresa só como cadastro** (usuário ligado a uma ou mais empresas, acervo compartilhado): só soma. Entram `companies`, `company_users` e as rotas delas, sem mudar nada do que está aqui.
- **Cada empresa com o próprio acervo** (várias locadoras na mesma plataforma): muda o que já existe.
  - `games` ganha `company_id`, e os jogos já cadastrados precisam ser ligados a uma empresa.
  - `rawg_id` passa a ser único por empresa, porque duas locadoras podem ter o mesmo jogo.
  - O papel passa a ser por empresa, e as regras de `403` das rotas mudam.

Para não fechar portas, os caminhos ficaram simples (`/games`, `/loans`). Um filtro como `?company_id=` pode ser acrescentado depois sem quebrar quem já usa a API.

## Pendências

- **ADR da autenticação:** a decisão por sessão no banco precisa ser registrada em ADR, como manda o [README dos ADRs](../adr/README.md).
- **Modelagem a atualizar:** entram a tabela `sessions` e os gêneros no MVP. Hoje a modelagem marca `genres` e `game_genres` como v2.
- **Duração da sessão:** falta decidir quanto tempo a sessão vale sem uso antes de expirar.
- **Quem cadastra jogos:** este documento considera que só o admin cadastra e altera jogos e gêneros.
- **Reativação:** não há rota para reativar conta desativada ou jogo retirado do acervo. Por causa do `rawg_id` único, um jogo retirado não pode ser cadastrado de novo.
- **Tamanhos de campo:** os limites de caracteres (nome até 100, senha com no mínimo 8, gênero até 50) são uma sugestão inicial.
