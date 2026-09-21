# Atualizações: sinaleiro físico, tomada Tuya e webcam local

Branch: `ml/ergonomia-zona-risco-labels` · Data: 21/09/2026

Este documento descreve o que foi adicionado ou alterado nesta rodada de mudanças. Há três frentes:

1. **Hardware de alerta:** o orquestrador passa a acender o sinaleiro (ESP32) e a cortar a energia por uma tomada Tuya quando há risco.
2. **Webcam do notebook:** ela pode ser cadastrada como câmera pelo frontend.
3. **Setor de teste:** foi criado um setor `teste` na configuração de análise.

---

## 1. Sinaleiro físico e tomada Tuya

**Arquivo novo:** [`orquestrador/hardware_alert.py`](../orquestrador/hardware_alert.py)

O módulo é portado da Fase 03 e adaptado ao orquestrador multi-setor. Cada setor roda o seu pipeline, e todos compartilham o mesmo hardware físico.

### Estados do sinaleiro

| Estado | Quando acende | Buzzer | Tomada |
|---|---|---|---|
| 🔴 Vermelho | Queda **ou** invasão de zona de risco (risco grave) | Soa | **Desliga** |
| 🟠 Laranja | EPI ausente (risco leve) | Soa | Não altera |
| 🟡 Amarelo | Estado neutro, aceso sozinho pelo firmware do ESP32 | Não soa | Não altera |
| 🟢 Verde | Tudo limpo: há pessoas e todas estão com EPI e fora da zona | Não soa | Religa, se estava cortada |

### Regras de decisão

- **Prioridade:** vermelho vence laranja quando os dois ocorrem juntos.
- **Vários setores, um só hardware:** basta **um** setor em risco para acender o alerta (OU). Para ir ao verde, **todos** os setores com gente precisam estar limpos (E).
- **Corte da tomada:** só queda ou invasão de zona desligam a tomada, e o comando é enviado uma única vez, na subida do risco. EPI ausente sozinho **não** desliga a tomada.
- **Religar:** a tomada só volta quando o sistema confirma "tudo limpo" por `CLEAR_FRAMES_TO_GREEN` (15) frames seguidos. Ela nunca volta ao sair do vermelho direto para o amarelo.
- **Heartbeat:** o estado do LED é reenviado a cada 2 s (`LED_REFRESH_INTERVAL`), abaixo do watchdog de 5 s do ESP32. A tomada não tem watchdog.
- **Falhas de comunicação:** são apenas registradas no log. Não derrubam o pipeline, porque os envios rodam em threads.

### Escolha da tomada (`PLUG_ATIVA`)

As tomadas ficam no dicionário `PLUGS`. Hoje há duas:

| Nome | Uso |
|---|---|
| `principal` | Tomada da apresentação final (padrão) |
| `teste` | AVATTO Wifi Smart Socket 16A, usada para testes em outra rede |

A tomada é escolhida pela variável de ambiente `PLUG_ATIVA` ou pelo valor padrão no código. Um nome inválido faz o orquestrador falhar ao iniciar, listando as opções válidas. Os logs mostram a tomada em uso, por exemplo `[TUYA:teste] desligar: ok`.

### Variáveis de ambiente

| Variável | Efeito | Padrão |
|---|---|---|
| `PLUG_ATIVA` | Qual tomada de `PLUGS` usar | `principal` |
| `ESP_IP` | IP do ESP32 (sinaleiro) | valor fixo no código |
| `PLUG_TESTE_IP` | IP da tomada `teste` | valor fixo no código |

Os IPs mudam a cada rede Wi-Fi. Para descobrir o IP de uma tomada Tuya, use `python -m tinytuya scan`.

**Exemplo (PowerShell) para testar com a tomada de teste:**

```powershell
$env:PLUG_ATIVA    = "teste"
$env:PLUG_TESTE_IP = "<ip da tomada nesse wifi>"
$env:ESP_IP        = "<ip do ESP32 nesse wifi>"
python main.py
```

Para a apresentação, suba o orquestrador **sem** definir essas variáveis. Ele usa a `principal` e os IPs padrão.

### Como o `main.py` usa o módulo

Veja a seção 3.

---

## 2. Script de teste do ESP32

**Arquivo novo:** [`orquestrador/test_esp32_led.py`](../orquestrador/test_esp32_led.py)

O script testa o firmware do sinaleiro direto pelo HTTP, sem o orquestrador. Ele roda uma bateria de testes:

- **Servidor:** `GET /` responde 200.
- **LEDs:** vermelho, laranja e verde ligam e desligam.
- **Validação de entrada:** corpo vazio, JSON inválido, `led` inválido e `state` inválido devolvem 400.
- **Exclusão mútua:** ligar vermelho depois de verde é aceito.
- **Watchdog:** teste opcional com `--watchdog`.

Opções úteis:

```
python test_esp32_led.py --ip 10.0.0.5            # outro IP
python test_esp32_led.py --only vermelho --loop   # pisca um LED até Ctrl+C (útil com multímetro)
python test_esp32_led.py --delay 4                # pausa maior entre os passos
```

O IP padrão agora vem da variável `ESP_IP`, igual ao `hardware_alert.py`. Antes, os dois arquivos tinham IPs diferentes.

---

## 3. Alterações no orquestrador ([`orquestrador/main.py`](../orquestrador/main.py))

1. **Integração com o hardware:**
   - `_run_sector` chama `hardware_alert.report_sector(...)` a cada frame processado, com três informações: EPI ausente confirmado, queda ou zona confirmadas, e se há pessoa presente.
   - Ao encerrar o pipeline de um setor, chama `hardware_alert.remover_setor(setor)`. Isso evita que a flag antiga bloqueie o verde para sempre.
2. **Webcam local:** em `_make_resolve_fn`, uma `streamUrl` puramente numérica (`"0"`, `"1"`, ...) é convertida em `int`. Assim a câmera é aberta via `cv2.VideoCapture` local, na mesma convenção do `CAMERA_SOURCE` de ambiente.
3. **Correção de reinício em loop:** em `_start_sector`, o cálculo de `cam_ids` agora inclui a `streamUrl`, como o `_sector_manager` já fazia. Antes, os dois cálculos nunca coincidiam. O gerenciador via "mudança" e reiniciava o pipeline a cada checagem (30 s), mesmo sem nenhuma câmera ter mudado.

---

## 4. Frontend: cadastro de webcam local

**Arquivos:**
- [`ButtonAddCam.jsx`](../frontend/src/features/cameraPage/components/ButtonAddCam.jsx)
- [`ButtonEditCam.jsx`](../frontend/src/features/cameraPage/components/ButtonEditCam.jsx)

- **Botão novo:** "Usar webcam do notebook (em vez de câmera de rede)", com ícone `Webcam` do `lucide-react`.
- **Campo do índice:** com o botão ativo, o campo de IP/URL dá lugar a "Índice do dispositivo". O padrão é `0`, a webcam principal. Com mais de uma câmera USB, tente 1, 2, e assim por diante.
- **Como é salvo:** a câmera fica com `ip: "local"` e `streamUrl` igual ao índice, por exemplo `"0"`. É essa convenção que o orquestrador reconhece (seção 3, item 2).
- **Edição:** ao editar, uma `streamUrl` numérica reabre o modal com o botão ativado e o índice preenchido.
- **Validação:** sem webcam local, o IP/URL passa a ser obrigatório. O formulário não envia mais com o campo vazio.

---

## 5. Configuração de análise

**Arquivo:** [`orquestrador/analise_config.json`](../orquestrador/analise_config.json)

Foi adicionado o setor **`teste`**, sem EPIs obrigatórios e com ergonomia ativa:

```json
"teste": { "epis": [], "ergonomia": true }
```

---

## 6. Dependências

**Arquivo:** [`requirements.txt`](../requirements.txt)

- Adicionado `tinytuya>=1.13.0`, usado para controlar a tomada Tuya.

---

## 7. Arquivos que não fazem parte desta atualização

Estes arquivos estão na raiz, sem versionamento, e contêm dados de credencial ou de dispositivos Tuya. **Não devem ser commitados:**

- `devices.json`
- `tinytuya.json`
- `tuya-raw.json`
- `snapshot.json`

O próprio `hardware_alert.py` também guarda `local_key` das tomadas direto no código. Antes de subir, mova esses valores para um `.env`, que já está no `.gitignore`.
