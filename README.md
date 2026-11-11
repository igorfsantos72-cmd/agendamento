# Agenda ALPALE Saúde Ocupacional

Site de agendamento de exames ocupacionais.

- `index.html` — painel da equipe (Firebase Auth, e-mail e senha). A aba **Ações** só aparece depois do login.
- `agendar.html` — página pública do cliente. Não usa a coleção `acoes_pendentes` e não deve ser alterada para o fluxo do assistente.

Firebase do projeto: `agendamento-alpale`.

## Ações pendentes

O assistente lê a caixa de pedidos da clínica, decide a resposta e grava uma proposta na coleção Firestore `acoes_pendentes`. A equipe aprova, pede ajuste ou recusa no painel. Só depois disso o assistente envia e-mail ou grava um agendamento.

O assistente entra com um usuário próprio do Firebase Auth (e-mail e senha), pelo endpoint REST `signInWithPassword`, e usa o `idToken` nas chamadas do Firestore. Não há senha nem token neste repositório.

### Campos do documento

| Campo | Quem grava | Conteúdo |
| --- | --- | --- |
| `tipo` | assistente | `resposta_email`, `agendamento` ou `remarcacao`. Outro texto curto também aparece no painel. |
| `remetente` | assistente | E-mail ou nome de quem pediu. |
| `empresa` | assistente | Empresa do pedido. |
| `assunto` | assistente | Assunto do e-mail. |
| `resumo` | assistente | Pedido em uma ou duas frases. |
| `rascunho` | assistente; a equipe pode editar ao pedir ajuste | Texto integral proposto. |
| `motivo` | assistente | Por que esta ação foi escolhida. |
| `dataSugerida` | assistente | Data `AAAA-MM-DD`, se houver. |
| `medicoSugerido` | assistente | Nome do médico, se houver. |
| `horarioSugerido` | assistente | Horário `HH:MM`, se houver. |
| `kitUrl` | assistente | Link http(s) do kit, se houver. Pode omitir. |
| `referencia` | assistente | Id opcional da mensagem na caixa de entrada, para o assistente achar o fio. |
| `status` | os dois | Ver ciclo abaixo. |
| `comentario` | equipe | Obrigatório em `ajustar`. Opcional em `aprovado` e `recusado`. |
| `criadoEm` | assistente | Data ISO 8601 (`2026-11-11T14:00:00Z`) ou timestamp do Firestore. |
| `respondidoPor` | painel | E-mail da conta que decidiu. |
| `respondidoEm` | painel | Data ISO 8601 da decisão. |
| `rascunhoOriginal` | painel | Primeiro rascunho do assistente, gravado só se a equipe editar o texto num ajuste. |
| `historico` | painel | Lista das decisões: `status`, `comentario`, `respondidoPor`, `respondidoEm`, `rascunhoAlterado`. |
| `enviadoEm` | assistente | Opcional. Preenchido quando o status passa a `enviado`. |

A coleção nasce no primeiro documento. Não é preciso criá-la à mão.

### Ciclo do `status`

1. **`pendente`** — o assistente criou a proposta. Ela aparece em Ações pendentes, da mais nova para a mais antiga. Nada é enviado.
2. **`aprovado`** — a equipe aceitou o rascunho atual. O assistente pode enviar o e-mail ou gravar o agendamento e, ao terminar, muda o status para `enviado`.
3. **`ajustar`** — a equipe não aceitou como está. `comentario` vem preenchido e `rascunho` pode ter sido reescrito. Isso **não** autoriza envio. O assistente lê o comentário e o rascunho, monta a nova proposta, grava esse texto, deixa o comentário para contexto e volta `status` para `pendente`. A equipe precisa aprovar a nova versão.
4. **`recusado`** — a equipe recusou. O assistente não envia e não agenda.
5. **`enviado`** — o assistente concluiu a ação depois de um `aprovado`.

`aprovado`, `ajustar`, `recusado` e `enviado` ficam no filtro **Histórico**. Se o assistente devolver um item de `ajustar` para `pendente`, ele volta para a fila.

O painel só grava a decisão se o documento ainda estiver `pendente`, para duas pessoas não responderem o mesmo item.

### Regras de segurança

As regras em produção estão no console do Firebase, não neste repositório. O arquivo `regras/acoes_pendentes.rules` é o bloco novo. Ele precisa ser **colado dentro** do `match /databases/{database}/documents` já publicado, sem apagar as regras de `agendamentos`, `contadores`, `configuracoes` nem as regras do Storage.

Efeito do bloco: `acoes_pendentes` só é lida e gravada por usuário autenticado (equipe e assistente). A página pública continua sem acesso a essa coleção.

### Exemplos REST

A `WEB_API_KEY` é o campo `apiKey` já publicado em `index.html`. Ela identifica o app; não é a senha do assistente. `BOT_EMAIL` e `BOT_SENHA` ficam só no ambiente do assistente.

O `idToken` dura cerca de uma hora (`expiresIn`).

```bash
export WEB_API_KEY="cole_a_apiKey_de_index.html"
export BOT_EMAIL="assistente@exemplo.com"
export BOT_SENHA="senha-fora-do-repositorio"

ID_TOKEN=$(curl -s -X POST \
  "https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=${WEB_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"${BOT_EMAIL}\",\"password\":\"${BOT_SENHA}\",\"returnSecureToken\":true}" \
  | jq -r .idToken)
```

Criar uma proposta:

```bash
curl -s -X POST \
  "https://firestore.googleapis.com/v1/projects/agendamento-alpale/databases/(default)/documents/acoes_pendentes" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "tipo": {"stringValue": "resposta_email"},
      "remetente": {"stringValue": "rh@empresa.com"},
      "empresa": {"stringValue": "Paraty"},
      "assunto": {"stringValue": "Pedido de periódico"},
      "resumo": {"stringValue": "O RH pede periódico para dois funcionários na quinta de manhã."},
      "rascunho": {"stringValue": "Olá,\n\nPodemos receber a equipe na quinta, às 07:00, com a Dra. Gizelle.\n\nAlpale Saúde Ocupacional"},
      "motivo": {"stringValue": "O pedido tem empresa, tipo de exame e uma data. A quinta de manhã é turno da Dra. Gizelle."},
      "dataSugerida": {"stringValue": "2026-11-19"},
      "medicoSugerido": {"stringValue": "Dra. Gizelle"},
      "horarioSugerido": {"stringValue": "07:00"},
      "referencia": {"stringValue": "inbox-msg-123"},
      "status": {"stringValue": "pendente"},
      "comentario": {"stringValue": ""},
      "criadoEm": {"timestampValue": "2026-11-11T14:00:00Z"}
    }
  }'
```

A resposta traz `name` com o id do documento, no final do caminho.

Consultar o que a equipe já respondeu (`aprovado`, `ajustar`, `recusado`, `enviado`):

```bash
curl -s -X POST \
  "https://firestore.googleapis.com/v1/projects/agendamento-alpale/databases/(default)/documents:runQuery" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "structuredQuery": {
      "from": [{"collectionId": "acoes_pendentes"}],
      "where": {
        "fieldFilter": {
          "field": {"fieldPath": "status"},
          "op": "IN",
          "value": {
            "arrayValue": {
              "values": [
                {"stringValue": "aprovado"},
                {"stringValue": "ajustar"},
                {"stringValue": "recusado"},
                {"stringValue": "enviado"}
              ]
            }
          }
        }
      }
    }
  }'
```

Cada item vem em `document.fields`. Texto fica em `stringValue`. Ordene no assistente por `criadoEm` ou `respondidoEm`; a consulta não pede índice composto.

Depois de cumprir um item `aprovado`, marque `enviado` (troque `ID_DO_DOCUMENTO`):

```bash
curl -s -X PATCH \
  "https://firestore.googleapis.com/v1/projects/agendamento-alpale/databases/(default)/documents/acoes_pendentes/ID_DO_DOCUMENTO?updateMask.fieldPaths=status&updateMask.fieldPaths=enviadoEm" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "status": {"stringValue": "enviado"},
      "enviadoEm": {"timestampValue": "2026-11-11T15:30:00Z"}
    }
  }'
```

Para devolver um ajuste à fila, faça outro `PATCH` com `status` = `pendente` e o novo `rascunho`. Não apague `historico`, `comentario`, `respondidoPor` nem `respondidoEm`.

### O que fazer uma vez no Firebase

1. Em Authentication → Users, crie o usuário do assistente com e-mail e senha. A senha não entra no git.
2. Em Firestore → Regras, cole o bloco de `regras/acoes_pendentes.rules` dentro das regras atuais e publique.
3. Abra o painel, entre com uma conta da equipe e confira a aba **Ações**. O contador mostra quantos itens estão `pendente`.
