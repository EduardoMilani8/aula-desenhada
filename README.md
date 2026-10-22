# Aula Desenhada 🎨

Transforma videoaulas em diagramas. Você manda o link, um arquivo de áudio/vídeo ou um texto,
e recebe um quadro no [Excalidraw](https://excalidraw.com) que explica o conteúdo desenhado,
dividido em seções ligadas aos minutos do vídeo.

> Feito para quem só entende quando vê desenhado.

## Como vai funcionar

1. Cole o link da aula (ou use a extensão no YouTube, ou envie um arquivo).
2. O Aula Desenhada transcreve o áudio, divide a aula por assunto e desenha cada parte
   como fluxograma, causa e efeito, hierarquia, comparação ou mapa mental.
3. Assista ao vídeo numa aba e acompanhe o quadro na outra. Cada seção mostra o trecho do
   vídeo que ela cobre ("📍 02:10–07:45 · Fase clara").

## Status

🚧 Em design. Veja o [documento de design](docs/superpowers/specs/2026-10-22-aula-desenhada-design.md).

## Tecnologias

- **Interpretação:** Claude Haiku 5.5
- **Transcrição:** legendas do YouTube (via extensão) ou Groq Whisper
- **Backend:** Python + FastAPI + Postgres (Supabase)
- **Frontend:** React + Vite + Excalidraw + ELK.js
- **Extensão:** Chrome/Firefox (Manifest V3)
