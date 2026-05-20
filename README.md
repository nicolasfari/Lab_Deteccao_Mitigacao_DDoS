# Detecção de Ataques DDoS na Web — TryHackMe Writeup

## Descrição
Investigação de um ataque DDoS usando Splunk. Registros de acesso à web coletados durante o período do ataque suspeito. Esses registros contêm uma mistura de tráfego normal de usuários e solicitações potencialmente maliciosas

---

## Conceitos

### DoS vs DDoS
- **DoS** — ataque de uma única máquina sobrecarregando o servidor
- **DDoS** — ataque distribuído usando uma botnet de dispositivos comprometidos, muito mais difícil de bloquear

### Tipos de ataque
| Tipo | Descrição |
|------|-----------|
| Slowloris | Envia requisições HTTP parciais para ocupar conexões |
| HTTP Flood | Volume massivo de requisições HTTP |
| Cache Bypass | Ignora a CDN forçando o servidor de origem a responder |
| Consulta excessiva | Requisições que consomem muitos recursos do servidor |
| Abuso de login | Sobrecarga da autenticação com tentativas repetidas |

### Endpoints mais visados
- `/login` — autenticação consome CPU e banco de dados
- `/search` — consultas complexas ao banco de dados
- `/api` — essencial para conteúdo dinâmico
- `/register` — requer gravação no banco de dados
- `/checkout` — gerenciamento de sessão e pagamento

---

## Investigação — DDoS no Splunk

### O site sofreu um ataque DDoS. Os logs foram carregados no Splunk para investigação. O objetivo foi analisar logs de acesso web coletados durante o ataque para identificar o endpoint alvo, os IPs da botnet, o User-Agent usado e o pico de requisições.

### 1. Identificar o endpoint mais atacado
<img width="1407" height="801" alt="image" src="https://github.com/user-attachments/assets/8feeff5e-3501-427e-a7a0-45c45ca5b831" />


- Conta quantas vezes cada URI foi solicitada e ordena do maior para o menor. O resultado mostrou `/search` com 81% de todas as requisições, claramente o alvo do ataque. 
- O atacante escolheu `/search` porque cada busca exige uma consulta complexa ao banco de dados, consumindo muito mais recursos do que uma página estática.

### 2. Identificar o IP principal da botnet
<img width="1400" height="797" alt="image" src="https://github.com/user-attachments/assets/cbadb934-afe0-4cb0-83f0-6530c6aee002" />


- Filtro para apenas as requisições para `/search` e contou por IP de origem. O IP `203.0.113.7` liderou com 50 requisições, o nó mais ativo da botnet.

### 3. Mapear o tamanho da botnet
<img width="1403" height="814" alt="image" src="https://github.com/user-attachments/assets/494cd6e6-2daf-46e3-a231-a730132bf364" />


Filtro todos os IPs da faixa suspeita `203.0.113.x` e contou as requisições por IP. Resultado: **60 IPs únicos** todos coordenados na mesma faixa, confirmando uma botnet organizada.

### 4. Identificar o User-Agent do ataque
<img width="1401" height="675" alt="image" src="https://github.com/user-attachments/assets/ca9afd10-1667-4d2e-aded-110b8dd8cc27" />


- Listo os User-Agents mais frequentes. O `Java/1.8.0_181` foi o mais usado com **135 requisições**, confirmando automação.
- Versões antigas do Java são frequentemente usadas em ferramentas de ataque automatizado.

### 5. Medir o pico do ataque
<img width="1406" height="652" alt="image" src="https://github.com/user-attachments/assets/bdf418a2-6746-4c8c-a0b3-01a998befe95" />


- Pesquisando sobre o volume de requisições por segundo. Pico de **207 requisições por segundo** durante o ataque.

### 6. Identificar o primeiro cliente legítimo impactado
- Filtrando por `status=503` e ordenando por tempo foi possível identificar o primeiro cliente legítimo a receber erro 503. Marcando o momento exato em que o servidor começou a ceder.


<img width="1400" height="805" alt="image" src="https://github.com/user-attachments/assets/8c87b178-0419-4569-8b1f-14dc8dcebbe1" />

---

## Defesas

| Defesa | Como ajuda |
|--------|-----------|
| Rate limiting no WAF | Limita requisições por IP por minuto |
| CAPTCHA | Bloqueia bots em endpoints críticos |
| CDN | Absorve volume de tráfego antes do servidor |
| Validação de entrada | Impede consultas abusivas ao banco de dados |
| Bloqueio geográfico | Bloqueia IPs fora do público alvo |

### Técnica de bypass identificada
- Atacantes podem contornar CDNs adicionando query strings aleatórias: /products?a=abcd1234. Isso força o servidor de origem a responder pois a CDN não tem essa URL em cache.

---

## Lições Aprendidas
- O Splunk permite identificar rapidamente o endpoint mais atacado, os IPs da botnet e o pico de requisições usando queries simples
- `Java/1.8.0_181` como User-Agent em volume é sinal de ferramenta de ataque automatizado
- Botnets distribuídas usam múltiplos IPs para parecerem origens diferentes
- Endpoints como `/search` são visados por consumirem mais recursos
- Erros 503 em cadeia marcam o momento em que o servidor cedeu

---

## Referências
- TryHackMe: https://tryhackme.com
- MITRE ATT&CK T1498 — Network Denial of Service: https://attack.mitre.org/techniques/T1498/
- MITRE ATT&CK T1499 — Endpoint Denial of Service: https://attack.mitre.org/techniques/T1499/
