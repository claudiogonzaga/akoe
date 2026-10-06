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
   - `GERAR_ATA` + `LINK_TEMPLATE_ATA`: monta o documento da ata a partir de um Doc modelo (ver abaixo).
   - `REDIGIR_ATA_COM_LLM`: redige a ata no próprio Colab, com um modelo aberto (ver abaixo).
5. Aguarde o término — ao final é exibido o link do Google Doc de cada pasta. Uma pasta que falhe (link inválido, sem permissão) não interrompe as demais: o erro aparece no resumo e as seguintes continuam.

### `CARIMBO_TEMPO` — carimbo de tempo

A transcrição é agrupada em blocos de duração fixa, cada um marcado pelo instante **cheio** em que o bloco começa — o que serve para localizar o trecho direto no áudio:

```
[00:00:00] Bom dia, damos início à audiência. Presentes o Ministério Público
e a defesa. Pode confirmar seu nome completo? Confirmo.

[00:03:00] O senhor presenciou os fatos? Presenciei, sim.
```

Opções: **a cada 1, 2, 3, 5 ou 10 minutos**; `Por segmento do modelo` (um carimbo por corte do Whisper — irregulares, de poucos segundos cada, o que fragmenta bastante o texto); ou `Sem carimbo de tempo` (parágrafo corrido). Padrão: a cada 1 minuto.

Os tempos vêm prontos do próprio modelo — **não há custo extra de processamento**. Blocos sem fala são omitidos. Em arquivos longos, que são fragmentados internamente, o tempo é deslocado para continuar coerente com o arquivo inteiro (o segundo fragmento começa em `00:10:00`, não em `00:00:00`).

### WhisperX (experimental)

Selecionar `whisperx-large-v3 (experimental)` troca o motor. Por baixo é o mesmo Whisper, com três acréscimos: detecção de voz antes de transcrever (menos alucinação em silêncio), processamento em lote (mais rápido) e tempos mais precisos.

⚠️ É **experimental** por um motivo concreto: o WhisperX é sensível às versões de `torch`/CUDA, que a Colab atualiza sem aviso. Pode falhar na instalação, exigir reiniciar o ambiente de execução, ou funcionar hoje e quebrar depois. Ele só é instalado se você selecionar essa opção — nos demais modelos nada muda. Se der errado, escolha um modelo comum (ex.: `large-v3`) e rode de novo.

Diarização (separar quem falou: `SPEAKER_00`, `SPEAKER_01`) **não** está implementada.

### Alucinação em laço do Whisper

O Whisper foi treinado com legendas e, em trechos de silêncio, música ou áudio ruim, tende a inventar créditos de legendagem e a entrar em laço, repetindo a mesma frase dezenas de vezes no lugar da fala real:

```
[00:03:00] Legenda Fulano de Tal Legenda Fulano de Tal Legenda Fulano de Tal…
```

O problema não é só o lixo no texto: enquanto está em laço o modelo consome o áudio sem transcrevê-lo, e o resultado parece "cortado pela metade".

Duas medidas contra isso:

1. **`condition_on_previous_text=False`** na chamada do modelo. Por padrão o Whisper alimenta cada janela de 30 s com o texto que ele mesmo acabou de produzir; basta uma repetição surgir para ela se auto-reforçar até o fim do arquivo. Desligar essa realimentação é o que corta o laço na origem. Os limiares de confiança, silêncio e taxa de compressão também ficaram explícitos no código, para não mudarem junto com a versão da biblioteca.
2. **Colapso de repetições** na montagem do texto, como rede de proteção: trechos repetidos três vezes ou mais em sequência viram uma ocorrência só, e o log avisa quantos foram afetados. A fala normal não é tocada — inclusive repetições legítimas como "não, não, não foi isso".

Se ainda aparecer em gravações muito ruidosas, o `whisperx-large-v3 (experimental)` tende a se sair melhor: ele detecta voz e descarta o silêncio **antes** de transcrever, que é justamente onde a alucinação nasce.

### Ata de audiência ou reunião

Marcando `GERAR_ATA` e colando o link de um Google Doc modelo em `LINK_TEMPLATE_ATA`, o Akoé monta na pasta, ao final de cada transcrição, um documento `ATA — <arquivo> (para a IA)`: uma cópia do modelo, com a transcrição anexada no lugar do marcador `[[TRANSCRICAO_BRUTA]]`.

É a mesma mecânica do **Graphé**: o documento reúne o prompt, o template e a transcrição, e **você o sobe no Gemini ou no Copilot**, que redigem a ata. O Akoé não envia nada para IA nenhuma nesse caminho.

O modelo precisa ter o marcador `[[TRANSCRICAO_BRUTA]]` — sem ele o documento é criado, mas a transcrição não é anexada, e o log avisa. O Akoé também preenche `[[ORIGEM_TRANSCRICAO]]`, com o nome do modelo usado. Os demais marcadores do seu template (data, participantes, número do procedimento) ficam como estão, para a IA ou para quem lavra a ata resolver — o Akoé não tem como saber esses dados e não inventa.

**O link do modelo fica guardado na sua conta.** Na primeira vez que você cola o link em `LINK_TEMPLATE_ATA`, o Akoé o grava em `Meu Drive/Akoe/preferencias.json`; nas próximas, com `GERAR_ATA` marcado e o campo em branco, ele usa o que está guardado. Um link novo colado no campo substitui o antigo. Como é o Drive que guarda, vale em qualquer computador onde você entre com a mesma conta Google — e, ao contrário do que ficaria no notebook, não vai para o GitHub. Para esquecer, apague o arquivo. Se o Drive estiver indisponível ou sem permissão de escrita, o recurso só deixa de funcionar: a transcrição segue normalmente.

Só o link do modelo é guardado: as pastas mudam a cada uso, e os demais campos (modelo, ação nos arquivos, carimbo, caixas de marcar) não têm como distinguir "deixei no padrão" de "escolhi o padrão", então um valor guardado acabaria vencendo uma escolha sua sem você perceber.

**`REDIGIR_ATA_COM_LLM` (opcional)** dispensa o Gemini: baixa o `Qwen/Qwen2.5-7B-Instruct` em 4 bits e redige a ata no próprio Colab, criando um segundo documento `(redigida pela IA)` na pasta. Três coisas a saber:

- Roda **depois** de toda a transcrição, nunca junto: o Whisper e o modelo de texto não cabem juntos na memória da T4, então o Whisper é descarregado antes.
- Transcrições longas não cabem na janela do modelo (o teto é 28 mil tokens, algo como 1 h a 1 h 30 de fala). Acima disso ele avisa e deixa o documento para o Gemini, em vez de cortar a transcrição pela metade.
- A qualidade fica bem abaixo da do Gemini. O documento para subir no Gemini continua sendo montado de qualquer forma.

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
- O documento consolidado é nomeado `[modelo] Transcrições de <primeiro_arquivo> e outros` — por exemplo `[large-v3] Transcrições de audiencia.mp4 e outros`. Como o nome do modelo entra no título, cada modelo gera seu próprio documento na pasta.

### Privacidade das saídas

As saídas de execução ficam salvas dentro do `.ipynb`. Como este repositório é público, há quatro camadas para que nomes de arquivos e IDs de pasta não vazem por ali:

1. **Limpeza automática ao final**: terminada a transcrição, o notebook apaga todo o log de progresso e reimprime só o resumo com os links dos documentos. O log com nomes de arquivos deixa de existir na saída salva. (O documento já está na pasta do Drive, então o log não tem mais utilidade.)
2. **Máscara no log**: enquanto roda, o log mostra `25.………p4` em vez de `25.05.26 - Reunião - TAC concurso público.mp4`. Isso cobre o caso em que a execução é interrompida por um erro no meio — aí a limpeza final não chega a rodar. O Google Doc consolidado continua com os nomes completos: a máscara vale só para o que aparece na tela. Controlada pela variável `MASCARAR_SAIDAS` no código (não aparece no formulário); mude para `False` se quiser o log legível.
3. **Hook de pre-commit**: zera as saídas de qualquer `.ipynb` antes de cada commit feito daqui. Ative uma vez por clone:

   ```bash
   git config core.hooksPath .githooks
   ```

4. **Ao salvar do Colab direto para o GitHub** (`Arquivo → Salvar uma cópia no GitHub`), o hook **não** roda — ele é local. Como a limpeza automática já removeu o log, o risco aqui é pequeno; em caso de execução interrompida por erro, use antes `Editar → Limpar todas as saídas`. Se você nunca salva do Colab para o GitHub, esse caso não te afeta.
