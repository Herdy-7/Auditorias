<!-- Converted in full from Relatório_técnico_final_—_segurança_e_performance.docx. -->

# Relatório técnico final — segurança e performance

Projeto: /home/ubuntu/bigbox-saas<br>
Escopo: revisão estática de código-fonte, configuração, dist/public já existente e medições de preview informadas.<br>
Data: 25 de setembro de 2026<br>
Alterações no projeto: nenhuma. A auditoria não editou código, configuração, lockfile ou artefatos de build.

## Conclusão executiva

Não foi confirmado nenhum problema de segurança classificado como crítico no escopo revisado. A aplicação já possui controles importantes no servidor: segredos ficam em variáveis de ambiente, operações administrativas exigem autorização, dados de leads são criptografados em repouso e a entrega de produção aplica cabeçalhos de segurança.

Os riscos mais relevantes são de exposição excessiva de dados por contratos públicos e de desempenho da experiência pública. O catálogo público devolve registros integrais, inclusive metadados de origem e sourceSnapshot; a criação de lead devolve um objeto de vendedor maior do que a tela precisa. No desempenho, catálogo e orçamento transferem o catálogo completo, a imagem crítica do detalhe só é descoberta após uma cadeia JavaScript → API → mídia externa e a maioria das imagens ainda depende de bigbox.net.br.

A maior oportunidade de impacto é reduzir os contratos públicos a DTOs com allowlist, paginar/projetar consultas do catálogo no servidor, entregar imagens responsivas sob domínio controlado e declarar corretamente o cache de assets com hash. A correção deve ser acompanhada de testes de catálogo, detalhe, relacionados, orçamento, recuperação pós-reload e administração.

> Leitura dos estados de evidência: Confirmado significa que a estrutura foi observada no código, no dist existente ou nas medições fornecidas. Condicional significa que o código permite o comportamento, mas depende de variáveis de ambiente ou de validação em produção. Hipótese a validar é uma consequência técnica plausível que requer teste de navegador ou infraestrutura antes de virar fato operacional.

## Contagem consolidada

| Classe | Quantidade | Observação |
| --- | --- | --- |
| Crítico confirmado | 0 | Não houve achado crítico confirmado no escopo estático. |
| Alto | 8 | Priorizar dados públicos, payloads de catálogo, imagem crítica, terceiros e cache de assets. |
| Médio | 10 | Corrigir após os itens altos ou no mesmo ciclo quando houver dependência. |
| Baixo | 7 | Planejar com testes de regressão apropriados. |
| Informativo | 1 | Sem exposição de segredo, mas com melhoria de higiene de build. |
| Total de achados | 26 | Achados consolidados nas seções B–G. |

## Metodologia e limitações

A revisão cobriu client, server, schema Drizzle, Vite, scripts e dist/public, excluindo node_modules e .git das buscas recursivas. Foram aprovados, em modo somente leitura, pnpm check (tsc --noEmit) e pnpm check:public-bundle; este último informou “Public bundle security check passed (24 files scanned)”. Não foi executado pnpm build, porque emptyOutDir: true reescreveria dist/public e isso contrariaria a exigência de não alterar o projeto. O dist inspecionado é posterior às fontes relevantes observadas. 1

Não houve login real, criação de lead real, auditoria de infraestrutura externa, teste de TLS/CDN, inspeção de cabeçalhos na publicação, Lighthouse em produção, emulação controlada de mobile ou dados de campo de LCP, CLS e INP. Assim, tamanhos gzip são medidas locais dos arquivos e não prova de que o servidor publica gzip ou Brotli; tempos de preview validam ordem de carregamento e comportamento observado, mas não substituem RUM ou CrUX. Campos efetivamente presentes em sourceSnapshot e tabelas dependem do banco.

# A. Problemas críticos de segurança

Nenhum problema crítico foi confirmado nesta auditoria estática. Esta conclusão não substitui pentest autenticado, revisão de infraestrutura, testes de autorização horizontal/vertical e validação de configuração de produção.

# B. Problemas importantes de segurança

## B-01 — DTO público do catálogo expõe metadados operacionais desnecessários

Classificação e estado: médio; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/server/db.ts; endpoints públicos catalog.products, catalog.productBySlug e catalog.relatedProducts em server/routers.ts.

Trecho/evidência: routers.ts:253–282 declara catalog.* como publicProcedure. Em db.ts:236–264, o retorno inclui {...product, category, colors}; em db.ts:294–348, inclui {...row.product, category, colors, images}; e db.ts:351–376 reaproveita a lista em relacionados. O schema em schema.ts:129–158 define sourceId, sourceUrl, sourceProductUrl, sourceImageUrl, sourceSnapshot e timestamps para produto; schema.ts:168–186 define URL e timestamps para imagens.

Problema: os endpoints públicos retornam o registro integral por spread. Isso inclui campos de origem, snapshot de coleta, flags operacionais e datas que não são necessários ao catálogo público.

Risco/impacto: qualquer visitante pode enumerar URLs e metadados de fornecedores, além de conteúdo presente em sourceSnapshot. Como o snapshot não é controlado por uma allowlist no contrato HTTP, novos dados internos podem passar a ser expostos sem uma mudança explícita na API pública.

Solução recomendada: retornar DTOs públicos com allowlist explícita. Excluir source*, sourceSnapshot, timestamps, flags e todo campo não consumido pela interface. Manter apenas URLs públicas de mídia aprovadas e campos necessários à listagem, detalhe e relacionados.

Risco de afetar funcionalidade: médio. Produtos ou imagens podem deixar de renderizar se o cliente usar hoje fallbacks como sourceImageUrl ou sourceUrl. Validar catálogo, detalhe, relacionados e orçamento após reduzir o contrato.

## B-02 — Resposta pública de criação de lead devolve o registro integral do vendedor

Classificação e estado: médio; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/server/routers.ts, endpoint público lead.create; retorno de vendedor em server/db.ts; confirmação em client/src/pages/Quote.tsx.

Trecho/evidência: routers.ts:284–306 define lead.create como publicProcedure e devolve seller: result.seller. db.ts:649–655 retorna assignedSeller integral. A tabela sellers contém, conforme schema.ts:216–229, id, nome, WhatsApp, e-mail, estado ativo, ordem de rodízio e timestamps. Quote.tsx:274–278 e 300–312 precisam apenas de nome e whatsappUrl.

Problema: a mutação pública repassa ao navegador campos internos do vendedor, embora a tela só necessite de informações mínimas para a confirmação e o link de contato.

Risco/impacto: criação de leads sintéticos pode permitir a coleta de e-mails, IDs internos, estado ativo, ordem de distribuição e datas dos vendedores. O rate limit por IP reduz volume, mas não elimina a exposição ou a coleta distribuída.

Solução recomendada: devolver DTO mínimo: seller com nome público ou null, publicId caso necessário e whatsappUrl calculada no servidor. Não devolver e-mail, ID interno, telefone bruto, flags, ordem de rodízio ou timestamps.

Risco de afetar funcionalidade: médio. Preservar displaySuccess.seller?.name e whatsappUrl. Qualquer uso futuro que precise de outro campo deve ter contrato próprio, autorizado e documentado.

## B-03 — Coletor de depuração de desenvolvimento pode persistir PII em logs locais

Classificação e estado: médio; confirmado para vite serve; não confirmado na entrega de produção.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/public/__manus__/debug-collector.js; plugin em /home/ubuntu/bigbox-saas/vite.config.ts.

Trecho/evidência: vite.config.ts:77–150 registra POST /__manus__/logs somente no modo serve e injeta o script em desenvolvimento nas linhas 82–97. debug-collector.js:21–46 define redação; 469–487 e 551–570 capturam request/response de fetch; 618–672 faz o mesmo para XHR. A redação cobre chaves como password, token, secret, authorization, cookie e session, mas não os campos de lead contactName, companyName, whatsapp, cnpj, email e notes, definidos em routers.ts:133–180. Foram identificados .manus-logs/networkRequests.log com 641.559 bytes e .manus-logs/sessionReplay.log com 895.788 bytes, sem leitura de seu conteúdo.

Problema: em preview ou desenvolvimento acessível pelo navegador, o coletor intercepta corpos de requisição e resposta e os persiste sem autenticação no diretório de logs. A redação atual não cobre PII dos leads.

Risco/impacto: PII de leads e, potencialmente, respostas administrativas podem ficar em console, replay e logs locais. A existência dos arquivos confirma que o mecanismo produz registros. O script está em dist/public, mas não foi observado como referência de HTML ou chunk de produção; portanto, não há confirmação de vazamento na entrega estática produtiva.

Solução recomendada: desabilitar o coletor em fluxos com dados reais ou exigir ativação autenticada, temporária e auditável. Usar allowlist de rotas/campos, mascarar explicitamente lead.create e dados administrativos antes da persistência, restringir permissões de .manus-logs e expurgar registros existentes conforme política de retenção.

Risco de afetar funcionalidade: médio. Reduz capacidade de reproduzir erros de QA. Preservar telemetria técnica agregada e um modo de diagnóstico autorizado reduz esse impacto.

## B-04 — Confirmação de orçamento persiste dados além do necessário em sessionStorage

Classificação e estado: baixo; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Quote.tsx.

Trecho/evidência: Quote.tsx:40–62 lê bigbox.quote.success.v1; 206–247 monta nextSuccess a partir do resultado e o grava em sessionStorage. A resposta em routers.ts:297–306 inclui seller e whatsappMessage; o vendedor vem integral de db.ts:649–655.

Problema: o objeto completo de sucesso é serializado para recuperação após reload. Ele inclui o vendedor excessivo e whatsappMessage, que incorpora observações do usuário.

Risco/impacto: aumenta a janela de exposição de dados de contato e observações durante a sessão. Qualquer JavaScript comprometido na mesma origem pode ler esse armazenamento.

Solução recomendada: persistir somente DTO de recuperação com publicId, nome público do vendedor e itens sanitizados. Excluir whatsappMessage, notas e objeto integral de vendedor. Remover a chave depois de mostrar a confirmação recuperada.

Risco de afetar funcionalidade: baixo. Não remover a persistência inteira sem alternativa, pois ela sustenta o fluxo de recuperação com ?recover=1.

# C. Problemas de performance

## C-01 — Catálogo baixa todos os produtos e pagina somente no cliente

Classificação e estado: alto; confirmado em preview e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Catalog.tsx; /home/ubuntu/bigbox-saas/server/db.ts, listPublicProducts.

Trecho/evidência: no preview, products e categories foram agrupados em uma chamada de 645.902 B para 153 produtos, enquanto a grade mostrava 15 cards. Catalog.tsx usa products.slice; a consulta devolve produtos e cores para todo o conjunto.

Problema: filtros, paginação e recorte visual são executados após transferir e processar o catálogo integral.

Risco/impacto: aumenta transferência, parse, memória e trabalho de renderização, especialmente em celular e CPU lenta. O custo pode retornar em refetches.

Solução recomendada: criar endpoint resumido e paginado por cursor/limit, aplicar filtros, contagem e ordenação no banco e reservar descrição, galeria e campos completos para productBySlug.

Risco de afetar funcionalidade: médio. Preservar busca, categoria, paginação na URL, ordenação e contagem de resultados.

## C-02 — Orçamento baixa o catálogo completo para preencher um único seletor

Classificação e estado: alto; confirmado em preview e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Quote.tsx.

Trecho/evidência: em /orcamento, catalog.products começou aos 554 ms e transferiu 643.069 B. O primeiro <select> tinha 154 opções. Quote.tsx chama catalog.products sem projeção ou filtro.

Problema: a página de conversão transfere e renderiza todo o catálogo antes de o visitante selecionar um produto.

Risco/impacto: consumo de cerca de 643 KB na etapa comercial, lista extensa e maior atraso de disponibilidade em mobile.

Solução recomendada: expor endpoint leve de opções pesquisáveis; resolver cores e detalhes somente após a seleção. Preservar resolução direta de ?produto= e considerar combobox com busca remota se o catálogo crescer.

Risco de afetar funcionalidade: médio. Manter pré-seleção por URL, validação de cor, múltiplos itens e fallback de indisponibilidade.

## C-03 — Imagem principal do detalhe é descoberta somente após cadeia de API

Classificação e estado: alto; confirmado quanto à ordem de carga; impacto de campo depende de rede e origem externa.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/ProductDetail.tsx.

Trecho/evidência: no detalhe medido, productBySlug começou aos 539 ms, durou 1.029 ms e só depois a imagem externa de 1.138 × 1.138 px entrou no DOM. fetchPriority="high" existe, mas é aplicado após a resposta do produto.

Problema: URL e metadados da mídia crítica só são descobertos após hidratação e consulta tRPC.

Risco/impacto: cria a sequência JavaScript → API → descoberta de imagem remota. Em rede móvel ou origem lenta, a tela permanece em “Carregando produto” e a mídia visual crítica inicia tarde.

Solução recomendada: entregar metadados críticos em SSR/loader para permitir preload da imagem, ou ao menos prefetch do detalhe em hover/foco do card. Hospedar ou proxificar mídia em CDN com srcset, sizes e fallback visual.

Risco de afetar funcionalidade: médio. Preservar deep links, produto não encontrado, galeria, compartilhamento e URLs legadas de mídia.

## C-04 — Cards carregam originais externos maiores que a área exibida

Classificação e estado: alto; confirmado em preview e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/components/ProductCard.tsx; /home/ubuntu/bigbox-saas/client/src/pages/Catalog.tsx.

Trecho/evidência: a grade instanciou 15 imagens de cards. Recursos externos apresentaram durações de 1,4 s a 8,9 s; imagens naturais variavam de 700 a 1.200 px. ProductCard usa somente src e loading="lazy" em área aproximada de 270 × 192 px.

Problema: os cards não usam variantes responsivas, srcset ou sizes. O lazy loading reduz competição inicial, mas não reduz o tamanho do original quando o card se aproxima da viewport.

Risco/impacto: maior transferência, decode, memória e degradação de scroll/INP em mobile. Não há fallback de erro consistente no card.

Solução recomendada: servir mídia em CDN/proxy com cache e variantes responsivas; adicionar AVIF/WebP, srcset, sizes, placeholder e onError consistente. Manter lazy loading e avaliar content-visibility em listas longas.

Risco de afetar funcionalidade: baixo a médio. Preservar alt, links, aparência e comportamento de produtos sem imagem; validar direitos e todas as URLs migradas.

## C-05 — Relacionados formam waterfall e usam seleção em memória

Classificação e estado: médio; confirmado em preview e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/ProductDetail.tsx; /home/ubuntu/bigbox-saas/server/db.ts.

Trecho/evidência: relatedProducts iniciou aos 1.654 ms, após productBySlug de 1.029 ms, e transferiu 17.009 B. ProductDetail usa enabled: Boolean(product); o servidor lista produtos da categoria antes de pontuar e limitar quatro.

Problema: a consulta secundária só inicia após o detalhe, apesar de a seção estar abaixo da dobra; a seleção no servidor percorre o conjunto da categoria antes de limitar.

Risco/impacto: competing work após conteúdo principal e custo crescente conforme a categoria aumenta.

Solução recomendada: carregar relacionados por IntersectionObserver próximo da viewport ou devolver quatro itens na resposta do detalhe por consulta dedicada, projeção mínima e limite no banco.

Risco de afetar funcionalidade: baixo. Manter a ordem atual por afinidade e garantir que falha em relacionados não afete o produto principal.

## C-06 — Elementos fixos concorrentes aumentam risco de atrito e INP no mobile

Classificação e estado: médio; confirmado no layout; impacto de campo é hipótese a validar.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/components/SiteShell.tsx; /home/ubuntu/bigbox-saas/client/src/pages/Catalog.tsx; /home/ubuntu/bigbox-saas/client/src/index.css.

Trecho/evidência: consentimento é bottom-4 z-50; WhatsApp é bottom-5 right-5 z-30; catálogo posiciona back-to-top na mesma área e filtro com fixed inset-0 z-50. A fonte é carregada por @import remoto.

Problema: em telas pequenas, consentimento, filtro, WhatsApp e back-to-top podem ocupar ou disputar a mesma área de toque.

Risco/impacto: CTAs podem se sobrepor e dificultar toques. Fonte e logos podem causar ajuste visual em rede fria. Não foi confirmado CLS no preview; o risco é de UX, INP e CLS em campo.

Solução recomendada: reservar áreas exclusivas; ocultar ou reposicionar ações flutuantes quando consentimento, menu ou filtro estiverem ativos; usar foco e scroll-lock no filtro. Declarar dimensões proporcionais de logos e revisar estratégia de fonte.

Risco de afetar funcionalidade: baixo a médio. Não ocultar WhatsApp permanentemente nem impedir consentimento; testar Android, iOS e teclado virtual.

## C-07 — Conteúdo abaixo da dobra é iniciado por timeout, não por visibilidade

Classificação e estado: baixo; confirmado em preview e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Home.tsx; /home/ubuntu/bigbox-saas/client/src/components/HeroProductRotation.tsx.

Trecho/evidência: Home aciona setLoadBelowFold após 800 ms. No preview, heroProducts iniciou aos 908 ms e produtos em destaque aos 1.662 ms, mesmo sendo dados abaixo da dobra.

Problema: o timeout busca conteúdo independentemente de o visitante demonstrar intenção de rolar até a seção.

Risco/impacto: pode competir com LCP e interação em mobile ou realizar tráfego sem uso.

Solução recomendada: substituir o timeout por IntersectionObserver com rootMargin calibrado. Manter fallback para navegação por teclado, âncora e rolagem rápida.

Risco de afetar funcionalidade: baixo. Preservar fallback e carregamento oportuno da seção.

## C-08 — Busca a cada tecla e listener de scroll fazem trabalho evitável

Classificação e estado: baixo; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Catalog.tsx.

Trecho/evidência: onChange alimenta diretamente selectSearch usado na query. handleScroll chama setShowBackToTop(window.scrollY > 520) em cada evento.

Problema: cada tecla pode gerar consulta/reconciliação, e o estado de back-to-top é oferecido repetidamente mesmo sem cruzar o limiar.

Risco/impacto: possível sequência de requests e trabalho durante rolagem, reduzindo margem para INP no mobile.

Solução recomendada: aplicar debounce de 250–400 ms ou useDeferredValue, ignorar respostas obsoletas e atualizar back-to-top apenas na transição de limiar ou via IntersectionObserver.

Risco de afetar funcionalidade: baixo. Preservar colagem, Enter e deep links com termo de busca.

# D. Problemas de imagens/assets

## D-01 — Catálogo depende majoritariamente de imagens em origem legada externa

Classificação e estado: alto; confirmado por inventário e código.

Arquivo e componente: client/src/components/ProductCard.tsx; server/routers.ts; server/_core/security.ts.

Trecho/evidência: o catálogo possui 153 produtos com imagem primária: 125 em https://bigbox.net.br e 28 em /manus-storage, ou 81,7% em origem externa. routers.ts:58–93 e 117–130 aceitam http://, https:// e /manus-storage/. A CSP permite https://bigbox.net.br em security.ts:50.

Problema: disponibilidade, latência, política de cache e otimização de 125 imagens dependem de host legado fora do controle direto da aplicação. O modelo aceita URLs arbitrárias http(s) para produto, categoria e galeria.

Risco/impacto: falhas ou lentidão na origem afetam cards, detalhe e rotação do herói. A falta de domínio controlado limita padronização de formato, cache, observabilidade e resposta a incidentes de mídia.

Solução recomendada: migrar mídia legada para storage/CDN próprio, gerar AVIF/WebP responsivos e restringir URLs a origens autorizadas. Manter allowlist e observabilidade durante a migração.

Risco de afetar funcionalidade: alto. Falhas de migração afetam cards, detalhe e herói. Planejar rollback e validar cada URL antes de cortar a origem legada.

## D-02 — Arquivos de catálogo são maiores que o necessário para cards e não têm variantes responsivas

Classificação e estado: médio; confirmado por medição e código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/components/ProductCard.tsx; /home/ubuntu/bigbox-saas/client/src/pages/ProductDetail.tsx.

Trecho/evidência: ProductCard.tsx:35–39 usa uma única URL sem srcset/sizes, em contêiner h-48. Entre 152 imagens primárias acessíveis, o maior arquivo foi bandeja-para-fast-food-modelo-lf400-img-01.png, com 1.116.927 bytes e 1.150 × 1.150 px. O total medido foi 16.757.753 bytes, média de 110.248 bytes; 48 imagens tinham lado de pelo menos 1.000 px, 23 excediam 200 KB, 2 excediam 500 KB e 1 excedia 1 MB.

Problema: uma única URL transfere originais para áreas pequenas. Lazy loading não adapta resolução ao viewport.

Risco/impacto: maior transferência e atraso visual ao aproximar cards da viewport, sobretudo em mobile e rede lenta.

Solução recomendada: gerar variantes 320/480/640 px em AVIF/WebP, servir com srcset/sizes e manter dimensões ou aspect-ratio explícitos. Priorizar a Bandeja Plástica para Refeitório Grande Reforçada.

Risco de afetar funcionalidade: médio. Preservar nitidez em detalhe, URLs antigas durante a transição e fallback para formatos não suportados.

# E. Problemas de JavaScript/bundle

## E-01 — vendor-react domina o JavaScript inicial

Classificação e estado: alto; confirmado no dist existente.

Arquivo e componente: /home/ubuntu/bigbox-saas/vite.config.ts; carregamento inicial em dist/public/index.html.

Trecho/evidência: index.html:12–15 referencia index-Uh9pX9JC.js e faz modulepreload de vendor-react-C0Jm7xYa.js. O chunk mede 403.438 B raw e 119.290 B gzip. vite.config.ts:179–184 agrupa react, react-dom, scheduler e wouter. Ele representa 71,5% do JavaScript inicial gzip (119.290 de 166.733 B) e 62,6% do total estático inicial JS+CSS gzip (119.290 de 190.690 B).

Problema: a rota pública inicial sempre baixa e processa um vendor grande antes da renderização de qualquer rota.

Risco/impacto: piora potencial de FCP, LCP, TBT e tempo de interatividade em dispositivos modestos ou redes lentas. É o maior componente mensurável do payload inicial.

Solução recomendada: em mudança autorizada, gerar análise por módulo com metafile Rollup ou visualizer e medir a contribuição de React DOM, React e Wouter. Só então avaliar remoção de duplicação/runtime ou estratégia compatível de carregamento.

Risco de afetar funcionalidade: não houve quebra atual. Não adiar React DOM sem validar hidratação/renderização, pois é runtime essencial.

## E-02 — CSS global bloqueante é o terceiro maior recurso inicial

Classificação e estado: médio; confirmado no dist existente.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/index.css.

Trecho/evidência: dist/public/index.html:16 carrega /assets/index-C0f91-oN.css, com 149.535 B raw e 23.957 B gzip. O total estático inicial JS+CSS é 723.519 B raw e 190.690 B gzip.

Problema: um único stylesheet global é bloqueante para renderização e pode conter regras de componentes ou rotas não visitados.

Risco/impacto: representa 12,6% dos bytes gzip iniciais de JS+CSS e pode atrasar pintura e LCP em primeira visita se possuir CSS sem uso.

Solução recomendada: medir cobertura de CSS nas páginas públicas e no admin; confirmar configuração de content/purge do Tailwind e remover estilos/componentes não usados. Manter CSS crítico da Home no caminho inicial.

Risco de afetar funcionalidade: nenhuma falha visual foi comprovada. Validar todas as rotas e estados de interação após purga ou divisão.

## E-03 — Rota Admin é lazy, mas concentra seis abas em um módulo grande

Classificação e estado: baixo; confirmado no dist existente.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/pages/Admin.tsx; lazy import em client/src/App.tsx.

Trecho/evidência: App.tsx:15 usa lazy(() => import("./pages/Admin")). Admin-Ca6Fq2DZ.js mede 140.921 B raw e 28.779 B gzip, é referenciado apenas pelo import dinâmico, e a fonte possui 2.403 linhas/88.660 B concentrando seis abas.

Problema: o primeiro acesso a /admin baixa e processa código de abas que podem não ser abertas na sessão.

Risco/impacto: maior latência apenas no primeiro acesso administrativo; a rota pública não paga esse custo.

Solução recomendada: se a UX administrativa justificar, dividir abas pesadas — catálogo, segurança, equipe e outras — em imports dinâmicos e manter shell/autenticação no chunk da rota.

Risco de afetar funcionalidade: baixo. Validar estados de abas, foco, queries e permissões após a divisão.

## E-04 — Declarações de patch/override do pnpm podem não ser reproduzidas em instalação futura

Classificação e estado: baixo; confirmado por aviso do pnpm e arquivos instalados.

Arquivo e componente: /home/ubuntu/bigbox-saas/package.json; pnpm-lock.yaml; dependência Wouter instalada.

Trecho/evidência: pnpm check:public-bundle e pnpm check emitiram aviso de que pnpm.patchedDependencies e pnpm.overrides em package.json não são mais lidas pela versão instalada, 10.4.1. O lockfile contém patchedDependencies e wouter@3.7.1(patch_hash=4e16...); o arquivo instalado possui __WOUTER_ROUTES__ em esm/index.js:341–352.

Problema: o patch está presente agora, mas atualização ou regeneração futura de dependências pode omitir a forma declarada no package.json.

Risco/impacto: risco de manutenção e reprodutibilidade, inclusive de perder o comportamento que adiciona window.__WOUTER_ROUTES__.

Solução recomendada: em alteração autorizada, migrar configurações para o formato compatível com o pnpm usado e executar instalação limpa, build e testes de roteamento. Não alterar lockfile nesta auditoria.

Risco de afetar funcionalidade: nenhum atual observado; há risco futuro caso seja feita alteração incompleta de dependências.

## E-05 — Bundle contém caminhos absolutos de desenvolvimento

Classificação e estado: informativo; confirmado no bundle atual.

Arquivo e componente: /home/ubuntu/bigbox-saas/dist/public/assets/*.js.

Trecho/evidência: busca somente leitura encontrou 22 caminhos distintos sob /home/ubuntu/bigbox-saas/client/src/ em chunks como index-Uh9pX9JC.js e input-DLdb0JbW.js. Não foram encontrados *.map, *.map.gz ou diretivas sourceMappingURL.

Problema: metadados de localização de desenvolvimento são publicados apesar de não haver sourcemaps.

Risco/impacto: não expõe credenciais ou código de servidor, mas revela estrutura e caminhos do ambiente de build e facilita fingerprinting.

Solução recomendada: garantir build de produção sem metadados de localização e adicionar verificação de CI que rejeite /home/, jsxDEV e caminhos absolutos em dist/public.

Risco de afetar funcionalidade: redução de precisão em diagnósticos de localização de componentes no bundle publicado.

# F. Problemas de terceiros

## F-01 — CSP pode bloquear beacons de conversão do Google Ads

Classificação e estado: alto; hipótese técnica a validar em produção com CSP ativa.

Arquivo e componente: /home/ubuntu/bigbox-saas/server/_core/security.ts; /home/ubuntu/bigbox-saas/client/src/lib/googleAds.ts.

Trecho/evidência: security.ts:41–55 permite www.googletagmanager.com em script-src, mas connect-src aceita apenas self e o endpoint opcional de Umami; img-src não inclui destinos de Google Ads. googleAds.ts:51 e 99–106 carregam gtag e disparam conversão.

Problema: a origem do script é permitida, mas as origens de requisições/pixels de mensuração não estão explicitamente liberadas.

Risco/impacto: o script pode carregar após consentimento, enquanto requisições de conversão podem ser bloqueadas pela CSP, comprometendo atribuição de campanhas. Isso não foi comprovado com DevTools ou relatórios CSP e deve ser tratado como hipótese até o teste.

Solução recomendada: validar em produção com CSP ativa e adicionar somente hosts Google Ads comprovadamente necessários a connect-src, img-src e/ou frame-src. Considerar CSP report-only apenas durante a validação ou mensuração server-side.

Risco de afetar funcionalidade: alto para atribuição de marketing; não deve afetar o fluxo comercial. Não ampliar a CSP de forma genérica.

## F-02 — Umami pode iniciar antes de uma preferência específica de analytics

Classificação e estado: médio; condicional à presença das variáveis VITE de analytics.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/lib/analytics.ts; /home/ubuntu/bigbox-saas/client/src/main.tsx; /home/ubuntu/bigbox-saas/client/src/components/SiteShell.tsx.

Trecho/evidência: main.tsx:9–12 chama scheduleAnalyticsLoad incondicionalmente. analytics.ts:17–22 injeta ${ANALYTICS_ENDPOINT}/umami quando configurado. SiteShell.tsx:14–17 e 70 expõem controle apenas para consentimento de anúncios.

Problema: a telemetria Umami pode começar antes de consentimento ou sem uma preferência independente; a configuração efetiva não foi observada, pois os valores de ambiente não foram revelados.

Risco/impacto: se configurado, pode afetar conformidade, transparência e expectativa do visitante sobre telemetria.

Solução recomendada: definir finalidade e base legal do Umami. Quando consentimento for necessário, bloquear o carregamento até a escolha e oferecer gestão/revogação persistente, acompanhada de política de privacidade acessível.

Risco de afetar funcionalidade: médio. O site permanece funcional; a mudança pode reduzir cobertura de analytics de visitantes que recusarem.

## F-03 — Preferência de anúncios não possui caminho visível de reabertura e revogação

Classificação e estado: médio; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/components/SiteShell.tsx; /home/ubuntu/bigbox-saas/client/src/lib/googleAds.ts.

Trecho/evidência: SiteShell.tsx:70 exibe diálogo apenas quando adsConsent é null. googleAds.ts:81–96 grava bigbox-ads-consent no localStorage e atualiza gtag; não foi identificado controle visível de remoção, reabertura ou vínculo no diálogo para política de privacidade.

Problema: depois de aceitar ou recusar, a decisão persiste sem uma interface evidente para alterar ou revogar a escolha.

Risco/impacto: reduz transparência e torna difícil gerir consentimento posterior.

Solução recomendada: adicionar acesso permanente a “Preferências de privacidade”, link para política e mecanismo de alteração/revogação. Na revogação, atualizar Consent Mode e impedir novos eventos.

Risco de afetar funcionalidade: médio. Não bloqueia o fluxo comercial, mas exige validar persistência, transição de estados e carregamento de anúncios.

## F-04 — DM Sans é carregada por @import de Google Fonts

Classificação e estado: baixo; confirmado no código e build.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/index.css; dist/public/assets/index-C0f91-oN.css.

Trecho/evidência: index.css:1 e 5–32 usam a fonte externa. O CSS compilado tem 149.535 B raw, cerca de 24 KB gzip e 1.887 blocos de regras; não há @font-face local e a URL Google permanece no CSS.

Problema: há custo adicional de DNS/TLS/transferência e possibilidade de FOUT; @import é menos favorável que declarar links/preconnect no documento.

Risco/impacto: indisponibilidade não bloqueia conteúdo porque há system-ui e display=swap, mas a tipografia pode variar temporariamente e depender de terceiro.

Solução recomendada: hospedar e subsetar DM Sans localmente; manter font-display: swap; fazer preload apenas dos pesos necessários. Se o terceiro permanecer, preferir link e preconnect em vez de @import e revisar CSS não usado.

Risco de afetar funcionalidade: baixo. Validar pesos, caracteres PT-BR, métricas de layout e licenciamento antes de trocar a fonte.

## F-05 — Integração Google Maps está inativa e não está pronta para ativação segura

Classificação e estado: baixo; preventivo; componente não está no bundle atual.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/components/Map.tsx; /home/ubuntu/bigbox-saas/client/src/App.tsx; /home/ubuntu/bigbox-saas/server/_core/security.ts.

Trecho/evidência: não houve import/uso de MapView fora do próprio arquivo e o manifesto não lista chunk de mapa. Map.tsx:89–109 e 134–142 injetam script pelo proxy Forge com chave frontend e DEMO_MAP_ID; o Promise não resolve no onerror. A CSP não libera o host padrão forge.butterfly-effect.dev em script-src.

Problema: se conectado sem preparo, o mapa pode falhar silenciosamente ou ficar pendente. Uma chave frontend só é aceitável se tratada como pública e restringida por origem/API.

Risco/impacto: não gera tráfego no estado atual. Quando ativado, há risco de falha de UX e de uso indevido de chave sem restrições verificadas.

Solução recomendada: remover o componente inativo ou prepará-lo antes do uso: chave restrita por origem/API, mapId de produção, timeout/rejeição no loader, CSP estrita para hosts necessários e carregamento sob demanda.

Risco de afetar funcionalidade: baixo no estado atual; médio quando a tela for ativada.

# G. Problemas de cache/compressão

## G-01 — Assets com hash não recebem política explícita de cache longo no servidor estático

Classificação e estado: alto; confirmado no código; cabeçalhos finais de CDN/proxy não foram verificados.

Arquivo e componente: /home/ubuntu/bigbox-saas/server/_core/vite.ts.

Trecho/evidência: serveStatic usa express.static(distPath) e sendFile em vite.ts:50–66, sem maxAge, immutable ou Cache-Control explícitos. Os bundles têm hash; por exemplo, vendor-react mede 403.438 B raw/119.290 B gzip e vendor-data 99.784 B raw/27.804 B gzip.

Problema: o nome versionado suporta cache seguro, mas a aplicação não declara a política longa que permite ao navegador reaproveitar esses arquivos de maneira previsível. index.html também não possui política explícita nesse trecho.

Risco/impacto: visitas recorrentes podem revalidar ou baixar novamente bundles relevantes, piorando LCP e responsividade em rede móvel. Uma camada CDN/proxy pode compensar isso, mas isso não foi confirmado.

Solução recomendada: separar HTML de assets: index.html com no-cache ou revalidação curta e /assets com Cache-Control: public, max-age=31536000, immutable. Validar os cabeçalhos efetivos na camada publicada e nunca aplicar immutable ao HTML.

Risco de afetar funcionalidade: baixo. Cache imutável no HTML é o principal risco, pois poderia apontar para chunks removidos após deploy; manter o HTML revalidável evita isso.

## G-02 — Dados públicos são imediatamente stale e toda API tRPC recebe no-store

Classificação e estado: médio; confirmado no código.

Arquivo e componente: /home/ubuntu/bigbox-saas/client/src/main.tsx; /home/ubuntu/bigbox-saas/server/_core/index.ts.

Trecho/evidência: main.tsx instancia apenas new QueryClient(); a única política staleTime encontrada é de cinco minutos para heroProducts. server/_core/index.ts define Cache-Control: no-store para todo /api/trpc.

Problema: queries ficam stale imediatamente pelo padrão e podem ser refeitas em foco, remontagem ou reconexão. A API ainda proíbe cache HTTP de todos os dados, inclusive catálogo público.

Risco/impacto: consultas públicas grandes, como catálogo e orçamento, podem repetir rede e renderização durante a navegação.

Solução recomendada: definir staleTime e refetchOnWindowFocus por tipo de dado; invalidar explicitamente após mutações administrativas. Para endpoints públicos cuidadosamente projetados, avaliar cache HTTP curto ou CDN conforme dados e privacidade permitirem; manter no-store para dados pessoais, sessão e administração.

Risco de afetar funcionalidade: médio. Catálogo pode ficar alguns minutos desatualizado; mitigar com invalidação após alterações e atualização manual quando necessário.

## Compressão: conclusão limitada

Não foi confirmado um problema de compressão HTTP. Há medidas locais gzip dos artefatos, mas não houve inspeção dos cabeçalhos Content-Encoding de produção, nem validação de Brotli, CDN ou proxy reverso. A próxima verificação deve confirmar que HTML, JS, CSS, fontes e imagens têm política adequada por tipo de recurso, sem confundir tamanho gzip local com transferência efetiva de rede.

# Estimativa dos maiores responsáveis pelo tamanho inicial

A tabela descreve somente os recursos estáticos referenciados diretamente pelo index.html do dist existente. Não inclui API, AVIF remoto, fontes externas, chunks de rota carregados depois, nem cabeçalhos HTTP. Os percentuais usam o total de 190.690 B gzip para JS+CSS inicial.

| Recurso | Raw | Gzip local | Parcela do JS+CSS inicial | Observação |
| --- | --- | --- | --- | --- |
| vendor-react-C0Jm7xYa.js | 403.438 B | 119.290 B | 62,6% | Maior responsável; é modulepreload. |
| vendor-data | 99.784 B | 27.804 B | 14,6% | Pré-carregado no HTML. |
| index-C0f91-oN.css | 149.535 B | 23.957 B | 12,6% | CSS bloqueante. |
| vendor-ui | 52.436 B | 13.922 B | 7,3% | Pré-carregado no HTML. |
| index-Uh9pX9JC.js | 18.326 B | 5.717 B | 3,0% | Entry da aplicação. |
| Total JS inicial | 573.984 B | 166.733 B | — | Quatro arquivos JS estáticos. |
| Total JS+CSS inicial | 723.519 B | 190.690 B | 100,0% | Não inclui HTML de 1.176 B raw/641 B gzip. |

Após resolver o import dinâmico, a cadeia exclusiva da Home — Home, SiteShell, ProductCard, badge e colors — acrescenta 59.174 B raw e 13.264 B gzip. A estimativa prática para JS+CSS da Home é 782.693 B raw e 203.954 B gzip, sem AVIF remoto nem respostas da API. A imagem AVIF pré-carregada do herói mediu 41.324 B no inventário e 41.624 B de transferência observada no preview; a diferença é compatível com os métodos de medição distintos e não foi normalizada como métrica de produção.

O build público completo possui 19 chunks JS, 916.390 B raw/238.554 B gzip, e um CSS de 149.535 B raw/23.957 B gzip. Esse total não é carga inicial: inclui rotas lazy. A maior parte de bibliotecas grandes no disco, como lucide-react, date-fns, recharts, react-dom e superjson, não deve ser interpretada como bytes enviados, pois tree shaking e lazy loading evitam grande parte delas no dist atual. Sem metafile ou visualizer, a atribuição interna exata dos vendors é apenas estimada pelas regras de chunks, imports e assinaturas minificadas.

# Dados e segredos que não devem ir ao navegador

Nenhum dos valores abaixo foi encontrado no dist/public atual pelas buscas informadas. A lista é uma regra de fronteira: estes dados devem permanecer no servidor, em variáveis de ambiente ou armazenamento protegido, e não em bundle, HTML, URL, localStorage, sessionStorage, logs de browser ou respostas públicas.

- Acesso a banco e serviços internos: DATABASE_URL, URLs de conexão MySQL/PostgreSQL/Redis, usuários, senhas, certificados, tokens de serviço e endpoints internos não públicos.

- Criptografia e autenticação: JWT_SECRET, LEAD_ENCRYPTION_KEY, chaves de assinatura, sementes, tokens de sessão, hashes recuperáveis, chaves de rotação e qualquer segredo de cookie.

- Administração local: LOCAL_ADMIN_USERNAME, LOCAL_ADMIN_PASSWORD, hashes de senha, allowlist de IPs administrativos, tokens de recuperação e detalhes de lockout.

- OAuth e identidade: segredos de cliente OAuth, refresh tokens, tokens de acesso, OAUTH_SERVER_URL quando for endpoint interno/server-only, OWNER_OPEN_ID quando usado como identificador operacional não público e credenciais de provedores.

- Forge e storage: BUILT_IN_FORGE_API_KEY, credenciais do proxy de storage, chaves de upload, URLs assinadas reutilizáveis e qualquer token que autorize operações no Forge.

- Chaves de terceiros: credenciais AWS, Google Cloud, Stripe, Slack, GitHub e outros provedores. Uma chave frontend de mapa só pode existir no cliente se for explicitamente pública, limitada por origem e API, com escopo mínimo e quota monitorada.

- Dados pessoais e comerciais: nome de contato, empresa, WhatsApp, CNPJ, e-mail, observações de lead, texto de whatsappMessage, e-mail de vendedor, IDs internos, estado ativo, ordem de distribuição e timestamps internos.

- Metadados de origem: sourceSnapshot, URLs de fornecedor não destinadas ao público, sourceId, sourceUrl, sourceProductUrl, sourceImageUrl e qualquer informação operacional de coleta.

Os identificadores Google Ads AW-732141222 e de conversão encontrados são identificadores de mensuração, não segredos de autenticação. Mesmo assim, devem ser usados apenas após a decisão de consentimento aplicável. As referências VITE_ANALYTICS_ENDPOINT, VITE_ANALYTICS_WEBSITE_ID, VITE_FRONTEND_FORGE_API_KEY e VITE_FRONTEND_FORGE_API_URL não tiveram valores encontrados no bundle atual. Qualquer variável VITE_* deve ser tratada como pública por definição; não se deve colocar nela um segredo de servidor.

# H. O que já está correto e NÃO deve ser alterado

## Segurança, privacidade e controles de servidor

- Manter segredos no servidor. Segredos de banco, JWT, criptografia de leads, OAuth, Forge e credenciais locais são lidos por process.env; o script de build rejeita nomes e valores server-only em dist/public.

- Manter proteção administrativa no servidor. Endpoints administrativos usam adminProcedure e, em produção, exigem papel admin e IP permitido. Leads, notificações, allowlist de IP e tentativas de login não são rotas públicas.

- Manter criptografia de PII de leads. Leads são criptografados antes da persistência e decriptados apenas no fluxo administrativo.

- Manter o modelo de sessão local. Token aleatório de 32 bytes, persistência somente de SHA-256, senha com scrypt, comparação timing-safe, lockout, expiração e inatividade formam uma base adequada.

- Manter cookies protetivos. Cookies de sessão são HttpOnly; o cookie de admin local é SameSite=Strict; secure é ativado em HTTPS.

- Manter limites e não armazenamento de API sensível. tRPC usa Cache-Control: no-store; há limites por IP para login, OAuth e criação de lead, além de limites de corpo.

- Manter cabeçalhos de proteção em produção. CSP, HSTS em HTTPS, nosniff, proteção contra framing, Referrer-Policy e Permissions-Policy são controles corretos. A recomendação F-01 é ajustar apenas hosts comprovadamente necessários, sem enfraquecer a política.

- Manter o proxy de storage no servidor. Ele valida chave, bloqueia traversal, libera apenas chaves allowlisted ou referenciadas por produto e preserva credencial Forge no servidor.

- Manter a postura inicial de consentimento. Google Ads inicia como negado e só carrega após consentimento explícito. localStorage guarda somente granted/denied, não token.

## Build, roteamento e entrega inicial

- Manter a configuração de Vite de produção. root, outDir, emptyOutDir, aliases e os grupos manuais de vendor estão coerentes. O fato de emptyOutDir reescrever dist também justificou não executar build nesta auditoria.

- Manter lazy loading de rotas e Suspense acessível. Home, catálogo, detalhe, orçamento, institucionais e Admin usam lazy; o fallback tem role="status" e aria-live="polite". Admin não entra junto da Home.

- Manter o HTML inicial enxuto. O documento usa pt-BR, viewport, preload explícito do AVIF provável de LCP e fetchpriority="high"; não pré-carrega chunks de rotas.

- Manter ausência de sourcemaps públicos. Não há *.map, sourceMappingURL ou sourcemap habilitado na configuração observada. O problema E-05 não contradiz isso; ele trata apenas caminhos absolutos ainda presentes no minificado.

- Manter carregamento ocioso de analytics e Ads. Analytics é agendado para idle, até 3 s; Google Ads só é carregado após consentimento e também em idle. Esses scripts não estão no HTML inicial.

- Manter o escopo do coletor Manus como desenvolvimento. Ele é limitado a apply: serve; embora dist/public/__manus__/debug-collector.js exista — 25.168 B raw/5.693 B gzip — não é injetado no HTML ou chunks de produção. A correção B-03 deve reduzir PII em desenvolvimento, não transformar esse coletor em recurso produtivo.

- Manter nomes de assets com hash. Eles permitem cache longo e invalidação quando o conteúdo muda; a ação G-01 é completar a política de cabeçalhos correspondente.

- Manter httpBatchLink para produtos e categorias. O batching evita um round-trip adicional; o problema C-01 é o volume de dados, não o agrupamento de requisições.

## Imagens, acessibilidade e runtime

- Manter a estratégia de LCP do herói. AVIF pré-carregado, fetchPriority alto, fallback WebP/PNG e dimensões explícitas são corretos. Só considerar mudança do preload em navegadores sem AVIF após dados reais de audiência.

- Manter lazy loading nos cards. Cards usam loading="lazy"; a ação necessária é entregar origem e dimensões adequadas, não antecipar imagens abaixo da dobra.

- Manter postergação de catálogo destacado e rotação do herói como princípio. A rotação respeita prefers-reduced-motion, pausa com aba oculta e usa staleTime de cinco minutos. A melhora C-07 é trocar timeout fixo por visibilidade real.

- Manter cache de um ano para objetos versionados de storage. Os 12 assets em /manus-storage responderam public, max-age=31536000, immutable; o proxy preserva Content-Type e nosniff.

- Manter reserva de espaço de mídia. ProductDetail reserva área da imagem e usa dimensões; miniaturas usam lazy/async. ProductCard tem contêiner de altura fixa, reduzindo risco de CLS.

- Manter proteção de links externos. Links de WhatsApp e redes sociais com target="_blank" usam rel="noreferrer".

- Manter carregamento condicional do Admin. Consultas administrativas dependem de autenticação e aba, evitando baixar dados administrativos completos na rota pública.

- Manter MapView fora do bundle enquanto não for usado. Não há chunk nem carregamento de Google Maps hoje; não ativar o componente sem as correções preventivas de F-05.

Não houve long task no buffer desktop medido e não houve entrada de layout shift no detalhe naquele instante. Esses sinais são positivos, mas não provam ausência de problemas de campo; manter a instrumentação de Web Vitals por rota, conexão e dispositivo é a forma adequada de confirmar evolução.

# Plano de ação priorizado — sem executar mudanças

| Prioridade | Ação | Resultado esperado | Validação mínima antes de publicar |
| --- | --- | --- | --- |
| P0 | Trocar spreads dos endpoints públicos por DTOs com allowlist e reduzir retorno de vendedor. | Remove metadados de origem e dados de vendedor desnecessários do navegador. | Catálogo, detalhe, relacionados, orçamento, confirmação e recuperação pós-reload. Inspecionar payloads. |
| P0 | Restringir ou desabilitar o coletor de debug para fluxos com PII; mascarar lead/admin e expurgar logs. | Evita persistência local indevida de dados pessoais em preview/desenvolvimento. | Teste autorizado de diagnóstico sem corpo de lead nos logs; permissões e retenção verificadas. |
| P1 | Criar contratos paginados/projetados para catálogo e opções de orçamento. | Reduz cerca de 643–646 KB de payload integral por navegação observada. | Filtros, URL, ordenação, contagem, busca, pré-seleção ?produto=, múltiplos itens e cores. |
| P1 | Migrar/proxificar imagens legadas, gerar variantes responsivas e priorizar a maior PNG. | Menos bytes, decode e dependência de bigbox.net.br. | srcset/sizes, fallback, qualidade em cards/detalhe, direitos e rollback de URLs. |
| P1 | Declarar cache de /assets com hash e revalidação adequada de index.html. | Reuso previsível em visitas recorrentes sem quebrar deploy. | Cabeçalhos efetivos via CDN/proxy, navegação após deploy e teste de chunk antigo. |
| P1 | Testar Google Ads sob CSP de produção e liberar somente destinos comprovados. | Mantém CSP restritiva e confirma atribuição. | Console, network, relatórios CSP, evento de conversão após consentimento. |
| P2 | Definir cache de queries por tipo de dado e carregar relacionados/abaixo da dobra por visibilidade. | Menos refetch e menor competição com interação/LCP. | Atualização pós-admin, foco/reconexão, navegação rápida e offline/erro. |
| P2 | Criar preferências de privacidade reabertas/revogáveis e decidir política de Umami. | Transparência e controle persistente de tracking. | Aceitar, recusar, reabrir, revogar e confirmar ausência de novos eventos quando aplicável. |
| P2 | Medir composição de bundles, cobertura CSS e Admin por aba antes de dividir. | Evita otimizações especulativas do runtime essencial. | Build reproduzível, roteamento, autenticação, foco e comparação de payload. |
| P3 | Migrar patches/overrides pnpm ao formato suportado e preparar/remover MapView inativo. | Maior reprodutibilidade e menos débito de integração futura. | Instalação limpa, build, roteamento e, se mapa ficar, falha controlada/CSP/chave restrita. |

## Critério de encerramento recomendado

Considerar o ciclo P0 concluído somente quando os payloads públicos não contiverem campos source*, sourceSnapshot, dados internos de vendedores ou PII persistida pelo coletor de depuração; o conjunto de fluxos comerciais e administrativos deve permanecer coberto por testes de regressão. Considerar P1 concluído somente após confirmar, no ambiente publicado, paginação efetiva, variantes de imagem, cabeçalhos de cache corretos e entrega de conversão Google Ads sob a CSP final.

Para acompanhar ganho real, registrar Web Vitals por rota, dispositivo e classe de conexão antes e depois de cada grupo de mudanças. Comparar bytes transferidos de catálogo/orçamento, LCP do detalhe de produto, taxa de erros de imagem, taxa de conversão e violações CSP; não assumir que redução de bytes locais equivale automaticamente a melhora de campo.

## Referências
