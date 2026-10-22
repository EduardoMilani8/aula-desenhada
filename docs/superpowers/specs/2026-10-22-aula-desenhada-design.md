# Aula Desenhada — Documento de Design

**Data:** 2026-10-22
**Status:** aprovado na conversa, aguardando revisão do documento escrito

## 1. Objetivo

Muita gente só entende um assunto quando ele é desenhado. O **Aula Desenhada** recebe uma
videoaula (link, arquivo de áudio/vídeo ou texto) e gera um quadro no Excalidraw com
diagramas que explicam o conteúdo, divididos em **seções ligadas a trechos do vídeo**
("02:10–07:45 · Fase clara da fotossíntese").

O usuário assiste ao vídeo numa aba e acompanha o desenho em outra. Não há sincronização
em tempo real: o quadro é gerado inteiro de uma vez.

### Critérios de sucesso

- Uma pessoa sem conhecimento técnico abre o site, entra com Google, cola um link ou usa a
  extensão, e recebe o quadro sem instalar nada além da extensão (opcional).
- Cada seção do quadro indica o trecho do vídeo que ela cobre, com um link que abre o
  vídeo naquele minuto.
- O quadro é editável (Excalidraw) e exportável (`.excalidraw` e PNG).
- Custo de IA por vídeo de 1h: menos de US$ 0,05 no pior caso.
- O dono do site nunca é surpreendido na fatura: há cota por usuário e teto diário global.

### Fora do escopo (por enquanto)

- Pagamentos e planos pagos (última fase, depois do lançamento).
- App de celular.
- Desenho sincronizado em tempo real com a reprodução do vídeo.

## 2. Decisões já tomadas

| Decisão | Escolha | Motivo |
|---|---|---|
| Distribuição | Site hospedado | O público-alvo (estudantes) não deve precisar instalar nada nem criar chave de API. |
| Interpretação do texto | Claude Haiku 5.5 (US$ 0,10 / US$ 0,50 por milhão de tokens) | Entende aulas longas em português; menos de 1 centavo por vídeo de 1h. Modelos locais pequenos não estruturam bem aulas longas. |
| Transcrição (sem legenda) | Groq Whisper Large v3 Turbo (US$ 0,04 por hora de áudio) | Não exige GPU no servidor nem no usuário. |
| Legendas do YouTube | Extensão de navegador lê a legenda na aba do usuário | O YouTube bloqueia downloads vindos de IPs de datacenter ("Sign in to confirm you're not a bot"). Na aba do usuário isso não acontece. |
| Quem desenha | Código, não IA | A IA decide só o conteúdo (seções, nós, setas, tipo). Layout e estilo são determinísticos. Saída pequena = custo baixo e resultado previsível. |
| Pagamentos | Última fase | Primeiro validar que o produto ajuda as pessoas. |

## 3. Arquitetura

```
 [Extensão do navegador] ──┐
   (lê a legenda na aba    │
    do YouTube do usuário)  │
                            ▼
 [Site: React + Excalidraw] ⇄ [API: Python/FastAPI] ⇄ [Postgres (Supabase)]
                                    │
                                    ▼
                         [Worker de processamento]
                          ├─ Groq Whisper (só quando não há legenda)
                          └─ Claude Haiku 5.5 (seções + diagramas)
```

### Estrutura do repositório

```
aula-desenhada/
├── core/        # Biblioteca Python pura: transcrição → seções → diagramas (sem web, sem banco)
├── api/         # FastAPI: autenticação, jobs, cotas, endpoints
├── worker/      # Processo que consome a fila de jobs e chama o core
├── web/         # React + Vite + Excalidraw: interface e layout dos diagramas
├── extension/   # Extensão Chrome/Firefox (Manifest V3)
├── docs/
└── docker-compose.yml
```

O `core` não conhece HTTP, banco ou usuários. Isso permite testá-lo isoladamente e, no
futuro, reaproveitá-lo numa versão instalável ou num MCP sem reescrever nada.

## 4. Pipeline

### 4.1 Entrada (da mais confiável para a menos)

1. **Extensão** — envia `{video_id, titulo, url, segmentos: [{inicio, fim, texto}]}`.
2. **Upload de arquivo** — o navegador extrai e comprime o áudio com ffmpeg.wasm (Opus,
   ~32 kbps; 1h ≈ 15 MB) antes do envio. O worker transcreve com a Groq.
3. **Link** — o worker tenta `yt-dlp` (legenda primeiro; áudio se não houver). Funciona em
   muitos sites; no YouTube pode falhar por bloqueio. Em caso de falha, a interface sugere a
   extensão ou o upload.
4. **Texto colado** — vai direto para a interpretação; sem timestamps, as seções são
   "por assunto".

Todas as entradas produzem o mesmo formato interno:

```python
Transcricao = {
  "origem": "youtube" | "upload" | "link" | "texto",
  "idioma": "pt",
  "segmentos": [{"inicio": 12.4, "fim": 17.9, "texto": "..."}],  # inicio/fim None para texto
}
```

### 4.2 Interpretação (duas etapas com Haiku 5.5)

**Etapa A — Segmentação.** Recebe a transcrição completa (com marcações de tempo) e devolve:

```json
{"secoes": [{"titulo": "Fase clara", "inicio": 130.0, "fim": 465.0, "resumo": "..."}]}
```

**Etapa B — Diagrama por seção** (chamadas em paralelo). Recebe o trecho da seção e devolve:

```json
{
  "tipo": "fluxo | causa | hierarquia | comparacao | mapa",
  "nos":   [{"id": "n1", "texto": "Luz solar", "detalhe": "opcional, curto"}],
  "setas": [{"de": "n1", "para": "n2", "rotulo": "excita"}],
  "grupos": [{"titulo": "Cloroplasto", "nos": ["n2", "n3"]}]
}
```

Ambas usam **structured outputs** da API da Anthropic (`output_config.format` com JSON
Schema), garantindo JSON válido. Limites: até 12 nós por seção (legibilidade), rótulos de
até 40 caracteres.

Uma seção que falha é repetida uma vez. Se falhar de novo, ela é entregue como "seção
textual" (título + resumo) com botão "tentar de novo"; as demais continuam.

### 4.3 Desenho (no navegador)

1. **Layout** com ELK.js (`elkjs`), algoritmo escolhido por tipo:
   - `fluxo` → `layered`, direção para a direita
   - `causa` → `layered`, direção para baixo
   - `hierarquia` → `mrtree`
   - `comparacao` → duas colunas lado a lado (layout manual simples)
   - `mapa` → `radial`
2. **Estilo** por tipo (cores, formas, espessura) — tabela de estilos no `web/`.
3. **Conversão** para elementos Excalidraw com `convertToExcalidrawElements`.
4. **Moldura da seção**: cada seção vira um `frame` do Excalidraw com o título
   "📍 02:10–07:45 · Fase clara". As molduras são empilhadas verticalmente.

O desenho acontece no cliente porque o Excalidraw e o ELK são bibliotecas JavaScript, e
porque o servidor só precisa guardar o JSON (pequeno).

## 5. Dados (Postgres)

| Tabela | Campos principais |
|---|---|
| `usuarios` | id (Supabase Auth), email, criado_em |
| `videos` | id, chave_cache (ex.: `yt:<video_id>` ou hash do arquivo), titulo, url, duracao, transcricao (jsonb) |
| `jobs` | id, video_id, usuario_id, status (`na_fila`, `transcrevendo`, `segmentando`, `desenhando`, `pronto`, `erro`), progresso, erro, criado_em |
| `quadros` | id, video_id, secoes (jsonb), diagramas (jsonb), slug_publico, criado_em |
| `uso_diario` | data, usuario_id, videos, custo_estimado_usd |

**Cache global:** antes de criar um job, procura `videos.chave_cache`. Se já existe quadro
pronto, devolve na hora, sem custo e sem consumir cota.

**Fila:** a tabela `jobs` é a fila. O worker pega jobs com
`SELECT ... FOR UPDATE SKIP LOCKED`. Não precisa de Redis.

## 6. Contas, cotas e custos

- Login com Google via Supabase Auth. A API valida o JWT do Supabase.
- Cota por usuário: `COTA_DIARIA_VIDEOS` (padrão 5). Vídeos vindos do cache não contam.
- Teto global: `TETO_DIARIO_USD`. O worker soma o custo real de cada chamada (tokens
  reportados pela API × preço; minutos de áudio × preço da Groq). Ao atingir o teto, novos
  jobs ficam bloqueados até o dia seguinte e a interface avisa.
- Duração máxima de vídeo: `DURACAO_MAXIMA_MIN` (padrão 180).

## 7. API (FastAPI)

| Método | Rota | Descrição |
|---|---|---|
| POST | `/jobs/transcricao` | Recebe transcrição pronta (extensão ou texto colado) |
| POST | `/jobs/link` | Recebe URL |
| POST | `/jobs/upload` | Recebe arquivo de áudio comprimido |
| GET | `/jobs/{id}` | Status e progresso (o site consulta a cada 2 s) |
| GET | `/quadros/{id}` | Seções + diagramas (JSON) |
| GET | `/p/{slug}` | Quadro público, sem login |
| POST | `/quadros/{id}/secoes/{n}/refazer` | Refaz uma seção que falhou |
| GET | `/me` | Usuário, cota restante, histórico |

## 8. Interface (web)

- **Início:** campo de link, área de soltar arquivo, aba "colar texto", botão "Desenhar".
- **Progresso:** etapas ao vivo ("Segmentando… Desenhando seção 3 de 8…").
- **Resultado:** quadro Excalidraw em tela cheia; barra lateral com seções e minutos;
  clicar leva o quadro até a seção; ▶ abre o vídeo naquele minuto (`&t=130s`) em nova aba;
  botões exportar `.excalidraw` / PNG e copiar link público.
- **Histórico:** vídeos já processados pelo usuário.

## 9. Extensão de navegador

- Manifest V3, compatível com Chrome e Firefox.
- Em `youtube.com/watch`, injeta um botão "🎨 Desenhar esta aula".
- Lê as faixas de legenda disponíveis no player (preferência: manual em português →
  automática em português → qualquer manual → qualquer automática), baixa no formato
  `json3` e converte para `segmentos`.
- Usa a sessão do site (token do Supabase) para enviar à API e abre o resultado.
- Sem legenda: avisa e oferece o upload no site.

## 10. Erros e casos-limite

| Situação | Comportamento |
|---|---|
| Link do YouTube bloqueado no servidor | Mensagem sugerindo a extensão ou o upload |
| Vídeo privado / link inválido | Mensagem clara, job marcado como `erro` |
| Vídeo acima da duração máxima | Recusado antes de gastar qualquer coisa |
| Cota do usuário esgotada | Mensagem com horário de renovação |
| Teto global atingido | Mensagem "voltamos amanhã"; jobs não são criados |
| Falha temporária Groq/Anthropic (429/5xx) | Retentativa com espera exponencial (até 3x) |
| Falha numa seção | Seção textual + botão "tentar de novo" |
| Worker reinicia no meio de um job | Job volta para a fila após tempo-limite de 10 min |

## 11. Testes

- **core:** testes unitários com transcrições de exemplo em `core/tests/fixtures/`;
  chamadas ao Haiku/Groq substituídas por respostas gravadas. Sem rede.
- **web:** testes do layout e da conversão JSON → Excalidraw (Vitest).
- **api:** testes de cota, teto, cache e permissões com Postgres de teste.
- **extensão:** teste do parser de legenda `json3`.
- **Ponta a ponta:** script manual com um vídeo curto real (gasta API, não roda no CI).
- **CI:** GitHub Actions rodando os testes sem rede.

## 12. Deploy

- **web:** Cloudflare Pages ou Vercel (estático).
- **api + worker:** um container Docker (Fly.io, Render ou VPS). Sem GPU.
- **banco + auth:** Supabase.
- **Local:** `docker compose up` sobe api, worker e Postgres; o web roda com `npm run dev`.
- Segredos via variáveis de ambiente: `ANTHROPIC_API_KEY`, `GROQ_API_KEY`,
  `SUPABASE_URL`, `SUPABASE_JWT_SECRET`, `DATABASE_URL`.

## 13. Fases de implementação

1. **core** — formato de transcrição, etapas A e B com Haiku, CLI de teste (`python -m core arquivo.txt`).
2. **web (desenho)** — layout ELK + conversão Excalidraw a partir de JSON de exemplo.
3. **api + worker + banco** — jobs, fila, cache, transcrição por Groq e yt-dlp.
4. **web (produto)** — login, envio, progresso, resultado, histórico, link público.
5. **extensão** — botão no YouTube, leitura de legendas, envio.
6. **cotas e custos** — cota por usuário, teto global, contabilização.
7. **deploy + CI** — publicação e testes automáticos.
8. **(depois)** pagamentos.
