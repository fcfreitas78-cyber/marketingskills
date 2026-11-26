# Pipeline de Produção com FFmpeg

Comandos de referência para montar, legendar e exportar a VSL quando o FFmpeg estiver disponível. Verifique antes com `ffmpeg -version`. Ajuste caminhos e durações ao projeto.

## Estrutura de Pastas

```
vsl-projeto/
├── roteiro.md
├── imagens/        # cena01.png, cena02.png...
├── clipes/         # cena01.mp4... (gerados ou animados)
├── audio/          # narracao.mp3, trilha.mp3, sfx/
├── legendas/       # narracao.srt
└── export/         # vsl_16x9.mp4, vsl_9x16.mp4, vsl_1x1.mp4
```

## 1. Animar Imagem Estática (Ken Burns)

Sem gerador de vídeo, transforme cada imagem em um clipe com zoom lento:

```bash
ffmpeg -y -loop 1 -i imagens/cena01.png -t 5 \
  -vf "scale=3840:-2,zoompan=z='min(zoom+0.0008,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=150:s=1920x1080:fps=30,format=yuv420p" \
  -c:v libx264 -preset medium -crf 18 clipes/cena01.mp4
```

`d` = duração em frames (5 s × 30 fps = 150). Use `s=1080x1920` para 9:16.

## 2. Concatenar Clipes

```bash
for f in clipes/cena*.mp4; do echo "file '$PWD/$f'"; done > lista.txt
ffmpeg -y -f concat -safe 0 -i lista.txt -c copy video_mudo.mp4
```

Se os clipes tiverem codecs ou resoluções diferentes, reencode em vez de `-c copy`.

## 3. Transição Suave Entre Dois Clipes

```bash
ffmpeg -y -i a.mp4 -i b.mp4 \
  -filter_complex "[0:v][1:v]xfade=transition=fade:duration=0.4:offset=4.6,format=yuv420p" \
  -c:v libx264 -crf 18 ab.mp4
```

`offset` = duração do primeiro clipe menos a duração da transição.

## 4. Mixar Narração e Trilha com Ducking

```bash
ffmpeg -y -i video_mudo.mp4 -i audio/narracao.mp3 -i audio/trilha.mp3 \
  -filter_complex "[2:a]volume=0.25[bg];[bg][1:a]sidechaincompress=threshold=0.03:ratio=8:attack=20:release=300[duck];[1:a][duck]amix=inputs=2:duration=first:normalize=0,loudnorm=I=-14:TP=-1.5:LRA=11[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -ar 48000 -shortest com_audio.mp4
```

`loudnorm I=-14` atende à maioria das plataformas sociais e do YouTube.

## 5. Gerar Legendas

Se houver transcrição disponível (Whisper, ElevenLabs STT), gere `legendas/narracao.srt` a partir do áudio final, para que o tempo bata com a narração. Revise ortografia e quebre linhas em até ~32 caracteres (9:16) ou ~42 (16:9).

## 6. Queimar Legendas com Estilo

```bash
ffmpeg -y -i com_audio.mp4 \
  -vf "subtitles=legendas/narracao.srt:force_style='FontName=Montserrat,FontSize=14,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BorderStyle=1,Outline=3,Shadow=0,Alignment=2,MarginV=120'" \
  -c:v libx264 -crf 18 -preset medium -c:a copy legendado.mp4
```

Para destacar palavra por palavra, gere um arquivo `.ass` com tags `{\c&H00D7FF&}` nas palavras-chave.

## 7. Tela Final com CTA

```bash
ffmpeg -y -f lavfi -i color=c=0x111111:s=1920x1080:d=5:r=30 \
  -vf "drawtext=text='Clique no botão abaixo':fontcolor=white:fontsize=72:x=(w-text_w)/2:y=(h-text_h)/2" \
  -c:v libx264 -pix_fmt yuv420p tela_final.mp4
```

Prefira uma tela final desenhada (Canva, Figma, Remotion) com logo e cores da marca quando disponível.

## 8. Exportar por Plataforma

Exporte cada formato a partir de uma montagem feita na proporção correta. Quando for preciso derivar de um master 16:9, use crop centralizado e confira se sujeito e textos não foram cortados:

```bash
# 16:9 master
ffmpeg -y -i final.mp4 -c:v libx264 -profile:v high -pix_fmt yuv420p -crf 18 -r 30 \
  -c:a aac -b:a 192k -ar 48000 -movflags +faststart export/vsl_16x9.mp4

# 9:16 (crop central de um master 1920x1080)
ffmpeg -y -i final.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" \
  -c:v libx264 -pix_fmt yuv420p -crf 20 -r 30 -c:a aac -b:a 192k -movflags +faststart export/vsl_9x16.mp4

# 1:1
ffmpeg -y -i final.mp4 -vf "crop=ih:ih,scale=1080:1080" \
  -c:v libx264 -pix_fmt yuv420p -crf 20 -r 30 -c:a aac -b:a 192k -movflags +faststart export/vsl_1x1.mp4
```

Legendas devem ser queimadas **depois** do reenquadramento, com margens próprias para cada proporção.

## 9. Checklist de Finalização

- [ ] Áudio sem cortes, picos ou ruído; narração inteligível sobre a trilha
- [ ] Legendas sincronizadas e sem erros de ortografia
- [ ] Textos dentro da área segura em cada formato
- [ ] Personagens e estilo consistentes entre cenas
- [ ] Todas as alegações têm comprovação real
- [ ] CTA visível, legível e com a ação correta
- [ ] `ffprobe` confirma resolução, fps, codec H.264 e áudio AAC
- [ ] Duração final confere com o planejado para cada versão

```bash
ffprobe -v error -show_entries stream=codec_name,width,height,r_frame_rate -show_entries format=duration -of compact export/vsl_9x16.mp4
```
