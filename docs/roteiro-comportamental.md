# Roteiro — diagrama comportamental (sequência de login)

## 1. Descrição inicial (linguagem natural)

> Um usuário acessa o Painel Admin e submete login e senha. A API KAM
> confere a senha e, se correta, emite os cookies de sessão. Se a senha
> estiver incorreta, o sistema registra a tentativa; depois de várias
> tentativas erradas seguidas, a conta fica temporariamente bloqueada, mesmo
> que a senha correta seja digitada depois.

## 2. Lacunas identificadas

1. Qual o limite de tentativas antes do bloqueio?
2. Por quanto tempo a conta fica bloqueada?
3. O bloqueio é por conta ou por IP/dispositivo?
4. O que acontece se a conta já estiver bloqueada e a senha estiver certa?
5. Existe verificação de "troca de senha obrigatória" (primeiro acesso) nesse
   mesmo fluxo?
6. O que exatamente é devolvido ao Painel em cada caminho (corpo da resposta,
   cookies, código HTTP)?

## 3. Perguntas de esclarecimento

1. O bloqueio é por conta (não por IP)?
2. O limite de tentativas e a duração do bloqueio são configuráveis ou fixos?
3. Uma senha correta durante o bloqueio deve ser recusada mesmo assim?
4. O fluxo de "esqueci minha senha" faz parte desta jornada?
5. A obrigatoriedade de troca de senha no primeiro acesso é tratada aqui ou é
   outra jornada?

## 4. Roteiro revisado (respostas obtidas a partir do código-fonte real do KAM)

- **Participantes**: Usuário, Painel Admin, API KAM (`POST /api/auth/login`),
  Banco de Dados.
- **Mensagens relevantes**: submissão de credenciais → leitura do usuário por
  `(tenantId, login)` → verificação de senha (bcrypt) → gravação do resultado
  (zera ou incrementa `tentativasLoginFalhas` / grava `bloqueadoAte`) →
  emissão de cookies (`Set-Cookie` de `access_token` e `refresh_token`) →
  resposta com o shape do usuário.
- **Condição de sucesso**: senha confere e a conta não está bloqueada →
  cookies emitidos, contador de tentativas zerado, `ultimoLoginEm`
  atualizado.
- **Falha realista modelada**: senha incorreta N vezes seguidas → bloqueio
  temporário (`bloqueadoAte`); uma tentativa subsequente **mesmo com a senha
  certa** é recusada enquanto o bloqueio não expirar — é por conta, não por
  IP (não há verificação de IP/dispositivo no código).
- **Fora de escopo**: fluxo de "esqueci minha senha" (não implementado no
  KAM hoje — existe só a infraestrutura de `PasswordResetToken`, sem
  endpoint) e troca de senha obrigatória no primeiro acesso
  (`mustChangePassword`) — são jornadas distintas; misturá-las aqui inflaria
  o diagrama sem necessidade.
- **Efeito colateral / por que não duplica**: cada tentativa de login sempre
  lê o estado atual (`tentativasLoginFalhas`, `bloqueadoAte`) do banco antes
  de decidir o próximo valor — não incrementa "às cegas" — então reenviar o
  mesmo POST não produz um efeito diferente do esperado (não há necessidade
  de uma chave de idempotência explícita nesta jornada, diferente do caso de
  pagamento da aula, porque aqui não existe operação que crie um registro
  novo a cada chamada).

## 5. Suposições assumidas antes do diagrama

1. O limite exato de tentativas e a duração do bloqueio são parâmetros de
   configuração — o diagrama não fixa um número, só o comportamento.
2. O bloqueio é avaliado **antes** de conferir a senha (uma conta bloqueada
   recusa mesmo com senha certa), e não depois.
3. A resposta de erro (401) não diferencia, no corpo, "senha errada" de
   "conta bloqueada" para o usuário final por padrão — a diferença é
   documentada aqui como suposição de segurança (evitar enumerar o motivo
   exato da falha), não como fato confirmado em todos os endpoints.

## 6. Checklist de revisão

- [x] Há um caminho de sucesso e pelo menos um caminho de falha realista?
- [x] A ordem das mensagens está clara (quem chama quem, em que ordem)?
- [x] O ponto onde o estado é lido antes de decidir o próximo valor está
      explícito (evita "efeito duplicado" por reenvio)?
- [x] O diagrama não mistura esta jornada com "esqueci minha senha" ou
      "troca de senha obrigatória"?
- [x] As suposições sobre parâmetros não confirmados (limite de tentativas,
      duração do bloqueio) estão declaradas, não fixadas como fato?
- [x] Fica claro que o bloqueio é avaliado antes da senha, não depois?
- [x] O diagrama evita detalhar formato exato de payload/erro além do
      necessário para entender o fluxo?
