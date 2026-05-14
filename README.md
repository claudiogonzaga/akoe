# Transcrever com Whisper

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/claudiogonzaga/transcrevercomwhisper/blob/main/_whisper_2026_02__Transcrever_Mi_dias_de_1_Pasta_do_Drive.ipynb)

Notebook Colab que transcreve automaticamente todos os arquivos de áudio e vídeo de uma pasta do Google Drive usando o modelo **Whisper** (OpenAI) ou modelos compatíveis do HuggingFace.

## O que o notebook faz

1. **Autentica** no Google Drive e armazena o token em pickle (`/content/drive/MyDrive/token_colab.pickle`), reutilizando-o em execuções futuras (renova automaticamente se expirado).
2. **Lê** todos os arquivos de mídia (áudio/vídeo) de uma pasta do Drive informada por link.
3. **Transcreve** cada arquivo com o Whisper local (ou um modelo HuggingFace, ex.: `pierreguillou/whisper-medium-portuguese`). Arquivos longos (acima de `LIMITE_DURACAO_S`, padrão 20 min) são automaticamente fragmentados em pedaços de `CHUNK_DURACAO_S` (padrão 10 min) via ffmpeg para evitar estouros de memória — as partes são transcritas separadamente e concatenadas.
4. **Consolida** as transcrições em um único Google Doc na própria pasta, com um sumário no topo (✅ transcritos / ⏳ pendentes) — atualizado a cada arquivo. Pode usar um **template** do Drive (Certidão de mídias de audiência, Ata de reunião) ou criar Doc em branco (modo "Outro").
5. **Retoma de onde parou**: se o documento consolidado já existir, apenas os arquivos ainda não transcritos são processados.
6. (Opcional) Salva os áudios extraídos dos vídeos em uma subpasta `Áudios Extraídos`.
7. (Opcional) Move o arquivo original para a lixeira do Drive depois de transcrito (reversível por 30 dias — não é hard-delete).

A transcrição segue um *prompt* de **transcritor jurídico**: integral, com identificação de interlocutores quando possível e marcação `[áudio ininteligível]` para trechos não compreendidos.

## Como usar

1. Clique no badge **Open in Colab** acima.
2. Em `Ambiente de execução → Alterar tipo de ambiente`, selecione **GPU** (recomendado para `large-v3`).
3. Execute a célula principal e **autorize** o acesso ao Google Drive na primeira execução. O token será salvo em `MyDrive/token_colab.pickle` e reutilizado nas próximas.
4. Ajuste os parâmetros do formulário:
   - `modelo_whisper`: `tiny`, `base`, `small`, `medium`, `large`, `large-v2`, `large-v3` ou um modelo HuggingFace (`org/modelo`).
   - `PASTA_DOCUMENTOS`: link da pasta do Google Drive com os áudios/vídeos.
   - `ACAO_ARQUIVO_ORIGINAL`: `Apagar` (move para a lixeira do Drive após transcrição bem-sucedida) ou `Manter`.
   - `ACAO_AUDIO_EXTRAIDO`: `Salvar em "Áudios Extraídos"` ou `Não salvar` (aplicável apenas a vídeos).
   - `TIPO_DOCUMENTO`: `Mídias de audiência`, `Ata de reunião` ou `Outro` (default).
   - `LINK_TEMPLATE_AUDIENCIA` / `LINK_TEMPLATE_ATA`: URL do Google Doc usado como template para cada tipo. O template deve conter um placeholder `{{TRANSCRIÇÕES}}` ou `[[TRANSCRIÇÕES]]` (case-insensitive, tolera com/sem acento) onde o bloco de transcrições será inserido. O documento original do template não é alterado — uma cópia é criada na pasta dos áudios.
5. Aguarde o término — o link do Google Doc consolidado é exibido ao final.

### Observações

- Se adicionar novos arquivos à pasta após uma execução, reexecute a célula (o notebook detecta automaticamente quais ainda faltam transcrever).
- A célula final (limpeza de cache) só deve ser executada em caso de erro de memória da GPU.
- O documento consolidado é nomeado `[modelo] Transcrições de <primeiro_arquivo> e outros`.
