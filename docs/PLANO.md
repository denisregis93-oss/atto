# Plano do sistema Atto

## Decisões tomadas

| # | Tema | Decisão |
|---|------|---------|
| 1 | Sistema | Novo, no repositório `atto` |
| 2 | Acesso | Todos acessam: secretaria, presidente e conselheiros têm login |
| 3 | Mensagens | Fase 1 só com e-mail. WhatsApp e SMS depois |
| 4 | Atas | Assinadas à mão. O PDF digitalizado é enviado ao sistema e vira o arquivo oficial |
| 5 | Banco | Supabase, projeto **Tarefas** (região São Paulo, plano Pro) |
| 6 | Confirmação de presença | Link direto no e-mail, sem login. Login só para abrir documentos |

## Interface (escolhida no protótipo)

| Tema | Escolha |
|------|---------|
| Menu | Barra lateral (no celular vira barra de abas embaixo) |
| Tela inicial | Foco na próxima reunião: contagem regressiva, quórum, material e o que falta preparar |
| Deliberações e ações | Quadro por status: atrasada, vencendo, em dia, concluída |
| Página da reunião | Abas: Pauta, Documentos, Presença, Ata, Deliberações |
| Cor institucional | Grafite e ouro (`#7a5a10` no tema claro, `#dcb860` no escuro) |
| Densidade | Compacta |
| Tema | Segue o aparelho (claro ou escuro) |
| Tipografia | Manrope (texto e interface) e Source Serif 4 (títulos) |

Cores de status, fixas em todo o sistema: verde = em dia, âmbar = vencendo, vermelho = atrasado, cinza = concluído.

## Ainda em aberto

1. Conselheiros podem enviar arquivos ou só ler? Sugestão: só a secretaria e os relatores enviam.
2. O link de envio para relatores entra no MVP ou na fase 2?

## Arquitetura

- **Frontend:** site estático e leve (Svelte ou HTML + Alpine.js), publicado em hospedagem estática.
- **Supabase (projeto Tarefas):**
  - Postgres com RLS (regras de acesso por perfil);
  - Auth com e-mail e verificação em duas etapas;
  - Storage para documentos;
  - Edge Functions para envio de e-mail;
  - pg_cron para lembretes.
- **E-mail:** Resend ou SMTP da empresa, com convite `.ics` anexo.

### Convivência no projeto Tarefas

O projeto Tarefas já tem tabelas de outros sistemas (financeiro, orçamento, CRM e outros) e 14 logins. Para isolar o Conselho:

- Todas as tabelas usam o prefixo `cons_`, e o bucket de arquivos é próprio (`conselho-docs`).
- O acesso depende de estar na tabela `cons_membros`. Ter login em outro sistema do projeto **não** dá acesso ao Conselho.
- As tabelas de outros sistemas (por exemplo `profiles`) não são alteradas.
- **Risco aceito:** quem administra o projeto Tarefas consegue ver os dados do Conselho. Se isso não for aceitável, o mesmo código roda em um projeto separado, sem mudanças.

### Repositório público

O repositório `atto` é público. O código fica visível, mas os dados não, porque são protegidos pelo RLS. Apenas a chave pública (anon) do Supabase pode aparecer no código. Nenhuma chave secreta entra no repositório.

## Upload de documentos

### Quem envia

| Quem | O que pode enviar |
|------|-------------------|
| Secretaria | Qualquer arquivo de qualquer reunião |
| Relator (diretor, sem login) | Recebe por e-mail um link de envio válido só para o seu item da pauta. Envia sem ver o restante do material |
| Conselheiro | Só lê. Não envia (decisão a confirmar) |

### Como funciona

1. **Na reunião:** a pessoa abre a aba Documentos, escolhe o item da pauta, arrasta os arquivos (ou clica para escolher) e acompanha a barra de progresso.
2. **No navegador:** o arquivo vai direto para o Supabase Storage, no bucket privado `conselho-docs`. Arquivos grandes usam envio retomável: se a internet cair, o envio continua de onde parou.
3. **Validação:**
   - formatos aceitos: PDF, Word, Excel, PowerPoint, PNG e JPG;
   - tamanho máximo de 50 MB por arquivo;
   - qualquer outro formato é recusado com uma mensagem clara.
4. **Caminho do arquivo:** `conselho-docs/{reuniao_id}/{item_id}/{uuid}-{nome}`.
5. **Registro:** uma linha em `cons_documentos` guarda nome, tipo, tamanho, versão, item, quem enviou e quando.
6. **Aviso aos conselheiros:** a secretaria envia vários arquivos e depois clica em "Avisar conselheiros". Sai **um único e-mail** com a lista do que é novo, e não um e-mail por arquivo.
7. **Leitura:**
   - o conselheiro abre o arquivo por um link temporário, válido por 60 segundos;
   - cada abertura é registrada, e isso alimenta o "lido por X de 7" e a trilha de auditoria;
   - o link copiado e enviado a outra pessoa expira logo e não funciona.
8. **Nova versão:** um envio sobre um documento existente cria a v2, a v3 e assim por diante. A anterior fica no histórico, e o contador de leitura recomeça.
9. **Ata assinada:**
   - depois da aprovação, a ata é impressa e assinada à mão;
   - em seguida é digitalizada e enviada como "Ata assinada";
   - a reunião passa para o status "Ata assinada", e esse PDF é o registro oficial.

### Segurança

- Bucket privado: nenhum arquivo tem endereço público.
- O RLS do Storage só permite leitura a membros ativos de `cons_membros`.
- Enviar e substituir é permitido só à secretaria e a links de relator válidos.
- Excluir arquivo é permitido só à secretaria. A exclusão é lógica e fica registrada na auditoria.
- Fase 2: marca d'água com o nome do leitor nos PDFs, e opção "somente visualizar" sem download.

### Capacidade

O plano Pro inclui 100 GB de armazenamento. Uma reunião típica tem 20 a 60 MB de material, ou seja, menos de 1 GB por ano.

## Modelo de dados (tabelas `cons_`)

- `cons_orgaos`: Conselho e comitês.
- `cons_membros`: pessoa, login, papel (`secretaria`, `presidente`, `conselheiro`, `suplente`), canais de contato.
- `cons_mandatos`: membro, órgão, cargo, início, fim, eleição de origem.
- `cons_eleicoes`: data, evento (AGO, AGE, RCA), descrição, documento.
- `cons_reunioes`: órgão, número, tipo, data, hora, formato, local, fase.
- `cons_pauta_itens`: reunião, ordem, título, tipo (deliberação, informativo, discussão), relator, minutos.
- `cons_conflitos`: item, membro, declaração.
- `cons_documentos`: reunião, item, caminho, nome, tipo, tamanho, versão, substitui_id, enviado_por, enviado_em, excluido_em.
- `cons_leituras`: documento, membro, quando.
- `cons_links_envio`: item, e-mail do relator, token, validade, usado_em.
- `cons_presencas`: reunião, membro, resposta, respondido_em, canal.
- `cons_atas`: reunião, texto, fase (rascunho, revisão, aprovada, assinada), documento assinado.
- `cons_ata_comentarios`: ata, membro, trecho, texto, resolvido.
- `cons_deliberacoes`: reunião, item, texto.
- `cons_acoes`: deliberação, título, responsável, prazo, status, concluída_em, comprovante.
- `cons_mensagens`: tipo, assunto, destinatários, enviado_em, entregues, abertos.
- `cons_auditoria`: quem, o quê, quando, detalhe.

## Fases

1. **MVP (~3–4 semanas):**
   - conselheiros, mandatos e eleições;
   - agenda e reuniões;
   - convocação automática por e-mail com `.ics` e confirmação de presença;
   - pauta;
   - upload de documentos com versões e leitura;
   - deliberações e ações;
   - ata com revisão e envio da ata assinada.
2. **Fase 2:**
   - lembretes automáticos;
   - pendências entrando sozinhas na pauta;
   - link de envio para relatores;
   - auditoria completa;
   - marca d'água;
   - WhatsApp.
3. **Fase 3:**
   - votação eletrônica e deliberação por escrito;
   - avaliação anual do conselho;
   - resumo de ata com IA.
