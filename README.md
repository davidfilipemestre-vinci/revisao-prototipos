# Revisão de Protótipos

Ferramenta interna da equipa de design para apresentar protótipos Figma a clientes de forma controlada (sem acesso ao ficheiro editável) e recolher os comentários deles organizados num só lugar, sem trocas de ficheiros por email.

## Como funciona, em resumo

- A equipa inicia sessão com a conta GitHub da empresa, cria **Projectos** (um por cliente, ex.: RTP, SIC), e dentro de cada projecto cria **Apresentações**, cada uma associada a um link de protótipo do Figma.
- Ao criar uma apresentação, copia-se um link específico e envia-se ao cliente por email. O cliente abre esse link **sem precisar de sessão nem conta**, navega no protótipo, e pode comentar directamente sobre o ecrã.
- Cada comentário fica automaticamente associado ao ecrã exacto do protótipo onde foi feito, através da detecção automática de navegação do Figma (Embed API).
- A distinção entre "comentário da equipa" e "comentário do cliente" não depende de login do cliente: é determinada por quem está autenticado no momento do comentário. Quem tem sessão iniciada é tratado como equipa; quem não tem, como cliente.
- Um sino no topo avisa a equipa, em qualquer ecrã, sempre que um cliente comenta, com histórico persistente e distinção visual entre visto e não visto.

## Arquitectura

Sistema totalmente estático, sem servidor próprio:

| Peça | Função |
|---|---|
| GitHub Pages | Aloja os ficheiros estáticos, dá o endereço fixo necessário para o Figma autorizar a detecção automática de ecrã |
| Supabase (Postgres + Auth + Realtime) | Base de dados, autenticação da equipa via GitHub, e actualizações em tempo real dos comentários |
| Figma Embed API | Incorpora o protótipo e avisa qual o ecrã actual, via uma aplicação OAuth privada registada em `figma.com/developers/apps` |

## Estrutura de ficheiros

```
index.html          Estrutura HTML, comum a todos os ecrãs
estilo.css           Todo o visual do sistema
sistema.js           Toda a lógica: autenticação, dados, ecrãs, notificações
logo-branco.webp      Logótipo usado no ecrã de autenticação
```

Não existem ficheiros de página separados (`login.html`, `projetos.html`, etc.). É uma aplicação de página única: o `sistema.js` troca o conteúdo dentro da `<div id="page">` consoante o ecrã, através das funções `renderLogin()`, `renderProjects()`, `renderDetail()` e `renderReview()`.

## Configuração necessária (feita uma única vez)

1. **GitHub Pages**: repositório com estes ficheiros na raiz, Pages activado a partir do ramo `main`.
2. **Aplicação Figma** (`figma.com/developers/apps`): aplicação privada, com "Allowed embed origins" a apontar para o endereço do GitHub Pages, e um scope mínimo seleccionado (ex.: `file_metadata:read`, não utilizado activamente, apenas para cumprir o requisito de publicação). **Tem de estar publicada** (mesmo que como privada), não basta ficar em rascunho, ou a detecção de ecrã não funciona.
3. **Supabase**: projecto criado, com o esquema e as regras de segurança correspondentes aos ficheiros SQL abaixo, e o fornecedor "GitHub" activado em Authentication → Providers, com uma aplicação OAuth própria do GitHub (diferente da do Figma) só para login.
4. As chaves `SUPABASE_URL`, `SUPABASE_ANON_KEY` e `FIGMA_CLIENT_ID`, definidas no topo do `sistema.js`.

## Migrações da base de dados (ordem de execução)

Estes blocos foram corridos, por esta ordem, no SQL Editor do Supabase:

1. **Esquema inicial**: tabelas `projects`, `presentations`, `comments`, com RLS activo e políticas base (equipa autenticada gere tudo, qualquer pessoa lê apresentações e comentários, autor do comentário obrigatoriamente correcto consoante sessão).
2. **Lista de acesso da equipa**: tabela `team_members` (email de cada pessoa autorizada), políticas ajustadas para verificar essa lista em vez de aceitar qualquer conta GitHub autenticada.
3. **Verificação robusta da equipa**: função `is_equipa()` (`security definer`), que consulta `auth.users` com privilégios elevados, evitando o erro de permissão que a verificação directa por `auth.jwt()` e por junção directa a `auth.users` provocava.
4. **Tempo real**: `comments` adicionada à publicação `supabase_realtime`, para os comentários aparecerem sozinhos em quem estiver a ver a mesma apresentação.
5. **Sinal de vida**: tabela `app_heartbeat`, para o banner que avisa a equipa se o sistema esteve muito tempo sem qualquer visita (risco de pausa do plano gratuito do Supabase).
6. **Comentários vistos**: coluna `seen_by_team` em `comments`, que sustenta o histórico persistente de notificações.

## Limitações e cuidados conhecidos

- **Plano gratuito do Supabase**: o projecto pausa automaticamente ao fim de 7 dias sem qualquer pedido à API. Qualquer visita ao sistema, mesmo de um cliente sem sessão, evita a pausa. O banner amarelo avisa a equipa se isso não acontecer há uns dias.
- **Sessão Figma activa no mesmo browser**: uma falha conhecida, do lado do Figma, faz o protótipo entrar em ciclo de carregamento quando quem o visualiza tem sessão Figma iniciada no mesmo browser. Contornado com o atributo `credentialless` no iframe; caso reapareça, a alternativa é rever num browser ou perfil sem sessão Figma iniciada.
- **Barra de nome do ficheiro do Figma**: removida através do parâmetro `hide-ui=1` no link do embed, não documentado oficialmente pelo Figma, mas confirmado a funcionar.
- **Detecção de ecrã**: depende inteiramente de a aplicação Figma estar publicada (mesmo que privada) e do endereço de origem estar correctamente registado. Sem isso, os comentários deixam de conseguir distinguir o ecrã onde foram feitos.
- **Notificações**: guardam-se apenas os últimos 30 comentários de clientes no sino; para consultar comentários mais antigos, é preciso abrir a apresentação em causa directamente.

## Transferência de conta

Está prevista a transferência do repositório da conta pessoal da autora original para a organização GitHub da empresa. Ao concluir essa transferência, é necessário actualizar, nos três sítios seguintes, o novo endereço:

1. "Allowed embed origins" na aplicação Figma.
2. "Redirect URLs" em Authentication → URL Configuration, no Supabase.
3. Confirmar que os três ficheiros (`index.html`, `estilo.css`, `sistema.js`) e o `logo-branco.webp` foram copiados para o novo repositório.
