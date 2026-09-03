<img src="akoe-logo.png" width="340" alt="Akoé">

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/claudiogonzaga/akoe/blob/main/Akoe.ipynb)


Notebook Colab que transcreve automaticamente todos os arquivos de áudio e vídeo de **até 5 pastas** do Google Drive usando o modelo **Whisper** (OpenAI) ou modelos compatíveis do HuggingFace.

## O que o notebook faz

1. **Autentica** no Google Drive por sessão (`auth.authenticate_user()`), sem persistir token em disco.
2. **Lê** todos os arquivos de mídia (áudio/vídeo) de cada pasta do Drive informada por link — até 5, processadas em sequência.
3. **Transcreve** cada arquivo com o Whisper local (ou um modelo HuggingFace, ex.: `pierreguillou/whisper-medium-portuguese`). Arquivos longos (acima de `LIMITE_DURACAO_S`, padrão 20 min) são automaticamente fragmentados em pedaços de `CHUNK_DURACAO_S` (padrão 10 min) via ffmpeg para evitar estouros de memória — as partes são transcritas separadamente e concatenadas.
4. **Consolida** as transcrições de cada pasta em um Google Doc criado nela mesma, com um sumário no topo (✅ transcritos / ⏳ pendentes) — atualizado a cada arquivo.
5. **Retoma de onde parou**: se o documento consolidado já existir, apenas os arquivos ainda não transcritos são processados.
6. (Opcional) Salva os áudios extraídos dos vídeos em uma subpasta `Áudios Extraídos`.
7. (Opcional) Move o arquivo original para a lixeira do Drive depois de transcrito (reversível por 30 dias — não é hard-delete).

A transcrição segue um *prompt* de **transcritor jurídico**: integral, com identificação de interlocutores quando possível e marcação `[áudio ininteligível]` para trechos não compreendidos.

## Como usar

1. Clique no badge **Open in Colab** acima.
2. Em `Ambiente de execução → Alterar tipo de ambiente`, selecione **GPU** (recomendado para `large-v3`).
3. Execute a célula principal e **autorize** o acesso ao Google Drive quando solicitado (a cada nova sessão do Colab).
4. Ajuste os parâmetros do formulário:
   - `modelo_whisper`: `tiny`, `base`, `small`, `medium`, `large`, `large-v2`, `large-v3`, um modelo HuggingFace (`org/modelo`) ou `whisperx-large-v3 (experimental)` — ver abaixo.
   - `PASTA_1` a `PASTA_5`: links das pastas do Google Drive com os áudios/vídeos. Preencha da primeira em diante; as que ficarem em branco são ignoradas, e a mesma pasta repetida é lida uma vez só. **O modo de entrada é automático**: com pelo menos um link, lê do Drive; com **todos em branco, abre o seletor de upload** do seu computador (nesse caso nada toca o Drive e a transcrição é baixada de volta ao final).
   - `ACAO_ARQUIVOS`: o que fazer depois de transcrever — quatro combinações entre manter/apagar a mídia original e manter/apagar o áudio extraído (ver tabela abaixo).
   - `CARIMBO_TEMPO`: de quanto em quanto tempo marcar o instante na transcrição (ver abaixo).
5. Aguarde o término — ao final é exibido o link do Google Doc de cada pasta. Uma pasta que falhe (link inválido, sem permissão) não interrompe as demais: o erro aparece no resumo e as seguintes continuam.

### `CARIMBO_TEMPO` — carimbo de tempo

A transcrição é agrupada em blocos de duração fixa, cada um marcado pelo instante **cheio** em que o bloco começa — o que serve para localizar o trecho direto no áudio:

```
[00:00:00] Bom dia, damos início à audiência. Presentes o Ministério Público
e a defesa. Pode confirmar seu nome completo? Confirmo.

[00:03:00] O senhor presenciou os fatos? Presenciei, sim.
```

Opções: **a cada 1, 2, 3, 5 ou 10 minutos**; `Por segmento do modelo` (um carimbo por corte do Whisper — irregulares, de poucos segundos cada, o que fragmenta bastante o texto); ou `Sem carimbo de tempo` (parágrafo corrido). Padrão: a cada 3 minutos.

Os tempos vêm prontos do próprio modelo — **não há custo extra de processamento**. Blocos sem fala são omitidos. Em arquivos longos, que são fragmentados internamente, o tempo é deslocado para continuar coerente com o arquivo inteiro (o segundo fragmento começa em `00:10:00`, não em `00:00:00`).

### WhisperX (experimental)

Selecionar `whisperx-large-v3 (experimental)` troca o motor. Por baixo é o mesmo Whisper, com três acréscimos: detecção de voz antes de transcrever (menos alucinação em silêncio), processamento em lote (mais rápido) e tempos mais precisos.

⚠️ É **experimental** por um motivo concreto: o WhisperX é sensível às versões de `torch`/CUDA, que a Colab atualiza sem aviso. Pode falhar na instalação, exigir reiniciar o ambiente de execução, ou funcionar hoje e quebrar depois. Ele só é instalado se você selecionar essa opção — nos demais modelos nada muda. Se der errado, escolha um modelo comum (ex.: `large-v3`) e rode de novo.

Diarização (separar quem falou: `SPEAKER_00`, `SPEAKER_01`) **não** está implementada.

### `ACAO_ARQUIVOS` — o que sobra depois de transcrever

"Mídia original" é o áudio/vídeo que estava na pasta do Drive. "Áudio extraído" é o WAV 16 kHz mono gerado a partir de vídeos — guardá-lo cria uma cópia na subpasta `Áudios Extraídos`.

| Opção | Mídia original | Áudio extraído |
|---|---|---|
| **Manter mídia original e apagar áudio eventualmente extraído** (padrão) | mantida | não guardado |
| Apagar mídia original e manter áudio eventualmente extraído | vai p/ lixeira | guardado |
| Apagar tudo depois de transcrever | vai p/ lixeira | não guardado |
| Manter tudo depois de transcrever | mantida | guardado |

Apagar move para a lixeira do Drive (reversível por 30 dias), não é exclusão definitiva. No modo upload nada disso se aplica: os arquivos ficam só no disco temporário do Colab.

### Observações

- Se adicionar novos arquivos à pasta após uma execução, reexecute a célula (o notebook detecta automaticamente quais ainda faltam transcrever).
- O documento consolidado é nomeado `[modelo] Transcrições de <primeiro_arquivo> e outros`.

### Privacidade das saídas

As saídas de execução ficam salvas dentro do `.ipynb`. Como este repositório é público, há quatro camadas para que nomes de arquivos e IDs de pasta não vazem por ali:

1. **Limpeza automática ao final**: terminada a transcrição, o notebook apaga todo o log de progresso e reimprime só o resumo com os links dos documentos. O log com nomes de arquivos deixa de existir na saída salva. (O documento já está na pasta do Drive, então o log não tem mais utilidade.)
2. **Máscara no log**: enquanto roda, o log mostra `25.………p4` em vez de `25.05.26 - Reunião - TAC concurso público.mp4`. Isso cobre o caso em que a execução é interrompida por um erro no meio — aí a limpeza final não chega a rodar. O Google Doc consolidado continua com os nomes completos: a máscara vale só para o que aparece na tela. Controlada pela variável `MASCARAR_SAIDAS` no código (não aparece no formulário); mude para `False` se quiser o log legível.
3. **Hook de pre-commit**: zera as saídas de qualquer `.ipynb` antes de cada commit feito daqui. Ative uma vez por clone:

   ```bash
   git config core.hooksPath .githooks
   ```

4. **Ao salvar do Colab direto para o GitHub** (`Arquivo → Salvar uma cópia no GitHub`), o hook **não** roda — ele é local. Como a limpeza automática já removeu o log, o risco aqui é pequeno; em caso de execução interrompida por erro, use antes `Editar → Limpar todas as saídas`. Se você nunca salva do Colab para o GitHub, esse caso não te afeta.

---

# 🔊 Gerar Áudios com Gemini (texto → MP3)

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/claudiogonzaga/transcrevercomwhisper/blob/main/_gemini_2026_07__Gerar_Audios_de_Textos_de_1_Pasta_do_Drive.ipynb)

Notebook irmão que faz o caminho **inverso**: transforma arquivos de **texto** (`.txt`, `.md` ou **Google Docs**) de uma pasta do Google Drive — ou enviados por **upload** — em **áudios narrados (MP3)** com a API **Gemini TTS**, devolvendo o MP3 **na mesma pasta do Drive** (ou para download, no modo upload).

Serve para gerar **notas de áudio**, **notas de esclarecimento ao público**, **comunicados institucionais**, **boletins**, **aulas narradas** e **audiolivros** — o formulário permite escolher o tipo de conteúdo (ou escrever uma **instrução de narração personalizada**), a voz (8 vozes do Gemini), o tom, o ritmo, a velocidade final e a leitura pausada com silêncio entre frases.

## O que o notebook faz

1. **Lê** os arquivos de texto de uma pasta do Drive informada por link (ou recebe arquivos por upload, sem tocar no Drive).
2. **Narra** cada texto com a API `gemini-2.5-flash-preview-tts`, fatiando textos longos por parágrafo/frase e mandando a mesma instrução de estilo em cada trecho (voz consistente do começo ao fim).
3. **Normaliza o volume** entre os trechos, concatena tudo e codifica o **MP3** (ffmpeg, com ajuste de velocidade preservando o tom da voz).
4. **Devolve o MP3 na mesma pasta** do Drive (textos que já têm MP3 são pulados — pode rodar de novo sem medo).
5. **Retoma de onde parou**: os trechos gerados ficam em cache no seu Drive (`MyDrive/GerarAudioGemini/cache-trechos`) — se a cota diária da API esgotar no meio de um texto longo, basta rodar de novo no dia seguinte.

## Como usar

1. Clique no badge **Open in Colab** acima (não precisa de GPU).
2. Crie uma chave **gratuita** da API Gemini em [aistudio.google.com/apikey](https://aistudio.google.com/apikey) e salve-a nos **Secrets** do Colab (ícone 🔑 na barra lateral) com o nome `GEMINI_API_KEY` — ou apenas cole a chave quando o notebook pedir.
3. Preencha o formulário (modo de entrada, link da pasta, tipo de conteúdo, voz, tom, ritmo…) e execute a célula.
4. No modo Drive, **autorize** o acesso na primeira execução (mesmo token pickle do notebook do Whisper).

> A cota gratuita do Gemini TTS é pequena (poucas requisições por dia) — notas e comunicados curtos saem em uma rodada; textos muito longos podem precisar de mais de um dia, e a retomada por trecho cuida disso.
