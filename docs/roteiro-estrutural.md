# Roteiro — diagrama estrutural (containers)

Processo seguido: descrição inicial → prompt de lacunas/perguntas → respostas
(usando conhecimento real do código-fonte do KAM) → roteiro revisado →
suposições declaradas → PlantUML/Mermaid → checklist.

## 1. Descrição inicial (linguagem natural, como um usuário do sistema a daria)

> O KAM (Kaue Access Manager) é um sistema de autenticação e autorização
> multi-tenant que atende várias aplicações internas de uma mesma organização
> (hoje: Custos e InfoCI, com outras futuras). Um administrador acessa um
> painel web para cadastrar tenants (clientes), empresas, filiais, usuários e
> permissões. As aplicações clientes usam o KAM para autenticar os usuários
> dessas aplicações e verificar o que cada um pode fazer, evitando que cada
> aplicação implemente seu próprio controle de acesso do zero.

## 2. Lacunas identificadas (rodando o prompt de lacunas da aula sobre a descrição acima)

1. Não está claro se o KAM roda sempre como serviço HTTP separado ou se também
   pode ser embutido como biblioteca dentro do backend de outra aplicação.
2. Não está definido se há uma única tecnologia de banco de dados ou mais de
   uma suportada.
3. Não está claro como uma aplicação cliente se autentica junto ao KAM para
   operações de manutenção (ex.: sincronizar seu catálogo de permissões) — é a
   mesma sessão do usuário final ou um canal separado?
4. Não está definido se o token de sessão trafega em header, cookie ou corpo
   da requisição.
5. Não está claro se existe algum tipo de fila/mensageria entre os
   componentes ou se a comunicação é só síncrona via HTTP.
6. Não está definido se o isolamento entre tenants acontece só por filtro
   lógico (`tenantId`) ou por banco/schema físico separado.
7. Não está claro se o painel administrativo é uma aplicação separada do
   backend ou parte do mesmo processo.
8. Não está definido o papel de um usuário "master" (cross-tenant) versus um
   usuário comum de um tenant.
9. Não está claro se existe algum motor/pacote compartilhado reaproveitável
   por outras aplicações além do próprio painel do KAM.

## 3. Perguntas de esclarecimento

1. O KAM é sempre um serviço HTTP à parte, ou pode rodar embutido no processo
   de outra aplicação?
2. Quantas tecnologias de banco de dados o KAM precisa suportar
   simultaneamente?
3. Existe um canal de autenticação diferente para uso "máquina a máquina"
   (deploy/sincronização), além da sessão de usuário?
4. O token de sessão viaja em cookie, header ou body?
5. O isolamento de tenant é só lógico (coluna `tenantId`) ou físico
   (schema/banco por tenant)?
6. O painel administrativo é um frontend separado do backend?
7. Existe um papel "master" que atravessa tenants, fora do RBAC comum?
8. O motor de autenticação/RBAC é uma biblioteca própria, reaproveitável por
   outras aplicações?

## 4. Roteiro revisado (respostas obtidas a partir do código-fonte real do KAM)

- **Escopo**: visão de containers do KAM operando como **serviço HTTP
  standalone** (o modo "biblioteca" fica fora — ver Lacunas remanescentes).
- **Nível**: C4 — Containers.
- **Limites e responsabilidades**: o Painel Admin só consome a API; toda
  regra de negócio e verificação de permissão fica na API; o banco é só
  persistência, sem lógica de autorização nele.
- **Integrações externas**: aplicações consumidoras (Custos, InfoCI,
  EntidadeMais) integram de **duas formas distintas**: (1) usuários finais
  dessas aplicações confiam no JWT emitido pelo KAM, validado via JWKS
  público — não fazem login na aplicação consumidora, o token é do KAM; (2)
  canal separado de **máquina**, autenticado por um segredo de deploy
  (`X-Deploy-Secret`), usado pela aplicação para sincronizar o próprio
  catálogo de permissões — não usa sessão de usuário nenhuma.
- **Restrições**: sessão nunca trafega em header/body/query, só cookie
  (cookie-only); banco de dados **dual** (MySQL ou SQL Server, escolhido por
  variável de ambiente — nunca os dois ao mesmo tempo numa mesma instância);
  não misturar, no mesmo diagrama, o modo standalone com o modo biblioteca
  (são duas topologias de deploy diferentes).
- **Lacunas remanescentes (aceitas conscientemente, fora do escopo deste
  diagrama)**: o modo "biblioteca" (`KamModule` importado por outro backend
  Nest) não aparece — merece um diagrama de containers próprio, onde o KAM
  deixa de ser um container HTTP separado e passa a ser módulos dentro do
  container do app hospedeiro; o pacote compartilhado `access-manager-core`
  não vira container próprio porque não é implantado separadamente — só
  existe como biblioteca compilada dentro do container "API".

## 5. Suposições assumidas antes do diagrama

1. O diagrama representa exclusivamente o modo (a) "serviço HTTP standalone".
2. O banco de dados é representado como um único container lógico "Banco de
   Dados Relacional", com nota de que o motor é escolhido por ambiente
   (MySQL ou SQL Server) — não são dois containers simultâneos.
3. "Aplicação Consumidora" representa qualquer uma das aplicações clientes
   (Custos, InfoCI, EntidadeMais) de forma genérica, já que a relação
   estrutural com o KAM é a mesma para todas.
4. O pacote `access-manager-core` não aparece como container próprio, só é
   citado como nota dentro do container "API".

## 6. Checklist de revisão

- [x] O diagrama está no nível C4 Container, sem misturar componentes internos?
- [x] As integrações externas (aplicações consumidoras) estão marcadas
      visualmente como externas?
- [x] Os dois canais de integração (validação de sessão vs. sincronização de
      permissões) aparecem separados, sem serem confundidos como uma coisa só?
- [x] O diagrama evita expor endpoints/rotas — só mostra dependências
      relevantes?
- [x] Fica claro que o banco é dual (MySQL/SQL Server) sem desenhar dois
      containers simultâneos?
- [x] O modo "biblioteca" foi deliberadamente excluído, e essa exclusão está
      registrada (não é omissão silenciosa)?
- [x] O pacote compartilhado `access-manager-core` não aparece como container
      próprio (não é implantado separadamente)?
- [x] As suposições estão explícitas e diferenciadas de fatos confirmados no
      código?
