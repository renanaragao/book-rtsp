# RTSP 2.0 na Prática: Livro em Português baseado no RFC 7826

> Fonte normativa: **RFC 7826** — https://www.rfc-editor.org/info/rfc7826/  
> Este livro resume e explica o RFC inteiro com foco de implementação.  
> Os exemplos em C# são didáticos e curtos, para ilustrar decisões de implementação.

---

## Como ler este livro

- Quando houver conflito entre este texto e a RFC, a RFC prevalece.
- O foco aqui é **entender e implementar partes do protocolo**, não construir um servidor/cliente completo.
- Cada capítulo aponta para seções correspondentes da RFC 7826.

---

## Sumário completo (mapeado ao RFC)

1. Introdução e visão geral (RFC §1-§2)  
2. Convenções e terminologia (RFC §3)  
3. Parâmetros do protocolo (RFC §4)  
4. Modelo de mensagem RTSP (RFC §5-§9)  
5. Conexões, sessão, CSeq e confiabilidade (RFC §4.3, §5-§10)  
6. Capacidades e pipelining (RFC §11-§12)  
7. Métodos RTSP (RFC §13)  
8. Dados binários interleaved (RFC §14)  
9. Proxies (RFC §15)  
10. Caching e validação (RFC §16)  
11. Códigos de status (RFC §17)  
12. Cabeçalhos RTSP (RFC §18)  
13. Sintaxe ABNF e considerações de segurança (RFC §19-§21)  
14. IANA, interoperabilidade e estratégia de implementação (RFC §22 + apêndices)

---

## 1) Introdução e visão geral (RFC §1-§2)

O RTSP 2.0 é um protocolo de controle para sessões de mídia em tempo real.  
Ele coordena setup, início, pausa, redirecionamento e encerramento da entrega de mídia.

### Fluxo canônico de sessão

1. `OPTIONS` (descoberta de capacidades)
2. `DESCRIBE` (descrição de mídia)
3. `SETUP` (negociação de transporte + criação de sessão)
4. `PLAY` / `PAUSE` / parâmetros
5. `TEARDOWN`

### Snippet C#: enum de métodos RTSP

```csharp
public enum RtspMethod
{
    OPTIONS, DESCRIBE, SETUP, PLAY, PLAY_NOTIFY, PAUSE,
    TEARDOWN, GET_PARAMETER, SET_PARAMETER, REDIRECT
}
```

---

## 2) Convenções e terminologia (RFC §3)

No RFC, termos como **MUST**, **SHOULD** e **MAY** têm significado normativo (BCP 14).  
Na prática:

- **MUST**: obrigatório para conformidade
- **SHOULD**: fortemente recomendado; desvio exige justificativa
- **MAY**: opcional

### 2.1 Como interpretar linguagem normativa na implementação

Ao ler uma regra da RFC, transforme em decisão de código:

1. se for **MUST**, trate violação como erro de protocolo;
2. se for **SHOULD**, implemente como padrão e só desvie com motivo claro;
3. se for **MAY**, trate como recurso opcional, idealmente com feature flag/capability.

Exemplo mental:

- "A request MUST include Session" -> valide e rejeite sem `Session` quando aplicável;
- "Server SHOULD include Range" -> mantenha como padrão de resposta;
- "Client MAY use pipelining" -> suporte opcional, sem quebrar operação básica.

### 2.2 Terminologia essencial do RTSP

- **RTSP Session**: contexto lógico de controle criado tipicamente no `SETUP`.
- **Session ID**: identificador dessa sessão no header `Session`.
- **CSeq**: número sequencial da requisição para correlação de resposta.
- **Control URI**: URI RTSP alvo da operação (mídia individual ou agregado).
- **Aggregated Control**: controle de múltiplas mídias como uma única sessão lógica.
- **Range**: intervalo temporal da reprodução (`npt`, `clock`, etc.).
- **NPT (Normal Play Time)**: tempo relativo da mídia (ex.: `npt=10-30`).
- **Ready state**: sessão preparada para tocar, ainda sem envio ativo.
- **Play state**: sessão em envio ativo de mídia.
- **Interleaved**: mídia encapsulada no mesmo canal RTSP (geralmente sobre TCP).

### 2.3 Cliente, servidor e fluxo de responsabilidade

- **Cliente RTSP** decide *o que quer fazer* (`PLAY`, `PAUSE`, `TEARDOWN`).
- **Servidor RTSP** decide *como atende* dentro da RFC (ajuste de range, erro válido, timeout).
- O contrato entre eles é baseado em:
  - método + URI;
  - cabeçalhos obrigatórios;
  - estado atual da sessão.

### 2.4 Erros de terminologia que causam bugs

1. Confundir RTSP com transporte de mídia: RTSP controla, RTP/RTCP costuma transportar.
2. Tratar `Session` como conexão TCP: sessão lógica pode sobreviver a detalhes de conexão conforme implementação.
3. Ignorar contexto de estado: método válido em `Ready` pode ser inválido em outro estado.
4. Considerar `Range` como absoluto em todo cenário live: para live time-progressing, "agora" se move.

### Snippet C#: regra obrigatória com erro explícito

```csharp
static void Require(bool condition, string message)
{
    if (!condition) throw new InvalidOperationException(message);
}
```

---

## 3) Parâmetros do protocolo (RFC §4)

Este bloco cobre:

- versão RTSP (`RTSP/2.0`);
- URI/IRI de recursos RTSP;
- identificador de sessão;
- formatos de tempo (`npt`, `smpte`, `clock`);
- tags de feature;
- propriedades de mídia.

### 3.1 Session ID (ID de Sessão)

A **sessão RTSP** é o contexto lógico que conecta várias requisições de controle ao mesmo fluxo de mídia.  
Sem sessão, cada request seria isolada; com sessão, cliente e servidor compartilham estado.

Em termos práticos, a sessão serve para:

1. identificar **qual reprodução** está sendo controlada (`PLAY`, `PAUSE`, `TEARDOWN`);
2. manter **estado temporal** (pause point, range atual, modo Ready/Play);
3. manter **estado de transporte** negociado no `SETUP`;
4. permitir **expiração por inatividade** via `timeout`.

Fluxo típico:

1. cliente faz `SETUP`;
2. servidor responde com `Session: <id>;timeout=<segundos>` (quando aplicável);
3. cliente envia esse mesmo `Session` nas próximas requisições daquela sessão;
4. sessão termina em `TEARDOWN` ou timeout.

#### Como o servidor detecta inatividade após o SETUP

Quando o `SETUP` conclui, o servidor passa a monitorar a sessão por uma janela de inatividade, normalmente anunciada em:

```text
Session: <id>;timeout=<segundos>
```

Funcionamento típico no servidor:

1. ao criar a sessão, ele grava `lastActivity` com o horário atual;
2. toda requisição RTSP válida daquela sessão (com `Session` correto) atualiza `lastActivity`;
3. periodicamente, o servidor compara `agora - lastActivity` com `timeout`;
4. se ultrapassar o limite, considera a sessão inativa e libera recursos.

Na prática, isso significa que apenas concluir `SETUP` não mantém a sessão viva indefinidamente:  
é necessário tráfego de controle RTSP ao longo do tempo (por exemplo `PLAY`, `PAUSE`, `GET_PARAMETER`, `SET_PARAMETER` ou outras mensagens válidas daquela sessão, conforme implementação).

Exemplo de header:

```text
Session: QKyjN8nt2WqbWw4tIYof52;timeout=60
```

- `QKyjN8nt2WqbWw4tIYof52` = identificador único da sessão;
- `timeout=60` = servidor pode encerrar contexto após inatividade.

### Snippet C#: parser de `Session` com explicação linha a linha

```csharp
// Retorna duas informações do header Session:
// 1) o ID obrigatório da sessão
// 2) o timeout opcional em segundos
static (string Id, int? TimeoutSec) ParseSession(string value)
{
    // Divide o header por ';'.
    // Ex.: "abc123;timeout=60" vira ["abc123", "timeout=60"].
    // Remove entradas vazias e espaços laterais.
    var parts = value.Split(
        ';',
        StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

    // A primeira parte é o ID da sessão.
    var id = parts[0];

    // Timeout começa nulo porque é opcional no header.
    int? timeout = null;

    // Percorre parâmetros extras (a partir do segundo item).
    foreach (var p in parts.Skip(1))
    {
        // Verifica se o parâmetro atual começa com "timeout=",
        // sem diferenciar maiúsculas/minúsculas.
        if (p.StartsWith("timeout=", StringComparison.OrdinalIgnoreCase) &&
            // Tenta converter o texto depois de "timeout=" para inteiro.
            int.TryParse(p["timeout=".Length..], out var t))
        {
            // Se conversão funcionar, guarda o timeout encontrado.
            timeout = t;
        }
    }

    // Devolve o par (ID da sessão, timeout opcional).
    return (id, timeout);
}
```

### 3.2 Range e tempos

A header `Range` diz **qual trecho temporal da mídia** o cliente quer reproduzir/controlar.  
Sem `Range`, em muitos cenários o servidor usa o ponto atual da sessão (pause point).

No RTSP 2.0, tempo de mídia pode aparecer em formatos diferentes (por exemplo `npt`, `clock`, `smpte`), e o suporte é negociado/indicado por `Accept-Ranges`.

#### O que é NPT

`NPT` (**Normal Play Time**) é o formato mais comum e representa tempo relativo da apresentação, em segundos.

Exemplos:

- `Range: npt=10-30` -> toca do segundo 10 ao 30;
- `Range: npt=10-` -> toca do segundo 10 até o fim (ou até novo controle);
- `Range: npt=-30` -> sem início explícito, com fim no segundo 30 (semântica depende do contexto/método);
- `Range: npt=now-` -> em conteúdo live/time-progressing, começa no "agora".

#### Por que `Range` é importante

1. controle fino de seek e retomada;
2. interoperabilidade entre cliente/servidor sobre o tempo efetivo entregue;
3. tratamento correto de erro (`457 Invalid Range`) quando o trecho pedido é inválido.

Em respostas de `PLAY`/`PAUSE`, o servidor pode devolver um `Range` ajustado para refletir o trecho real que será/foi entregue.

### Snippet C#: parser simples de `Range: npt=start-stop`

```csharp
// Estrutura imutável para transportar o resultado do parse.
// Start e Stop são opcionais (null quando não vierem no Range).
public readonly record struct NptRange(double? Start, double? Stop);

static NptRange ParseNpt(string rangeValue)
{
    // Espera formatos como:
    // "npt=10-30", "npt=10-", "npt=-30"
    // Remove o prefixo "npt=" e guarda só "10-30", "10-", "-30".
    var raw = rangeValue["npt=".Length..];

    // Divide em no máximo 2 partes usando '-':
    // "10-30" -> ["10", "30"]
    // "10-"   -> ["10", ""]
    // "-30"   -> ["", "30"]
    var split = raw.Split('-', 2);

    // Se o início vier vazio, vira null; senão converte para double.
    // InvariantCulture evita erro com vírgula/ponto dependendo da localidade.
    double? start = split[0].Length == 0
        ? null
        : double.Parse(split[0], System.Globalization.CultureInfo.InvariantCulture);

    // Mesmo raciocínio para o fim da faixa.
    double? stop = split[1].Length == 0
        ? null
        : double.Parse(split[1], System.Globalization.CultureInfo.InvariantCulture);

    // Retorna o intervalo NPT normalizado.
    return new NptRange(start, stop);
}
```

---

## 4) Modelo de mensagem RTSP (RFC §5-§9)

RTSP define uma gramática de mensagem bem rígida.  
Na prática, toda implementação precisa dominar 4 peças:

1. **Request-Line** (método, URI, versão), ex.: `PLAY rtsp://... RTSP/2.0`;
2. **Headers** (metadados como `CSeq`, `Session`, `Range`);
3. **Linha em branco** separando cabeçalhos do corpo;
4. **Body** opcional (com `Content-Length` consistente).

### 4.1 Requisição RTSP: estrutura mínima

Exemplo mental:

```text
PLAY rtsp://example.com/media RTSP/2.0
CSeq: 42
Session: abcd1234
Range: npt=10-20

```

Se houver corpo, ele vem após a linha em branco e deve bater com `Content-Length`.

### Snippet C#: builder de requisição RTSP (linha por linha)

```csharp
// Necessário para StringBuilder e Encoding.
using System.Text;

// Monta uma requisição RTSP completa em formato texto.
// method: PLAY/PAUSE/SETUP...
// uri: recurso RTSP alvo
// cseq: número sequencial da requisição
// headers: cabeçalhos extras opcionais
// body: corpo opcional
static string BuildRequest(string method, string uri, int cseq, IDictionary<string, string>? headers = null, string? body = null)
{
    // Buffer eficiente para construir string em múltiplas etapas.
    var sb = new StringBuilder();

    // Primeira linha obrigatória da requisição RTSP.
    sb.AppendLine($"{method} {uri} RTSP/2.0");

    // Header obrigatório para correlação de mensagens.
    sb.AppendLine($"CSeq: {cseq}");

    // Se houver cabeçalhos adicionais, escreve um por linha.
    if (headers is not null)
        foreach (var (k, v) in headers)
            sb.AppendLine($"{k}: {v}");

    // Se houver corpo, calcula o tamanho em bytes UTF-8 e envia Content-Length.
    if (!string.IsNullOrEmpty(body))
        sb.AppendLine($"Content-Length: {Encoding.UTF8.GetByteCount(body)}");

    // Linha em branco obrigatória: separa cabeçalhos do corpo.
    sb.AppendLine();

    // Escreve o corpo somente quando existe.
    if (!string.IsNullOrEmpty(body)) sb.Append(body);

    // Converte o buffer inteiro em string final da mensagem.
    return sb.ToString();
}
```

### 4.2 Resposta RTSP: leitura da Status-Line

A primeira linha da resposta normalmente é algo como:

```text
RTSP/2.0 200 OK
```

Para lógica de controle, o campo mais importante é o código numérico (`200`, `457`, `460`...).

### Snippet C#: leitura de status line (linha por linha)

```csharp
// Extrai o código de status da primeira linha da resposta RTSP.
// Retorna -1 se a linha estiver inválida.
static int ParseStatusCode(string statusLine)
{
    // Quebra por espaços, ignorando espaços duplicados.
    // "RTSP/2.0 200 OK" -> ["RTSP/2.0", "200", "OK"]
    var parts = statusLine.Split(' ', StringSplitOptions.RemoveEmptyEntries);

    // Confere se existe pelo menos o segundo token (código),
    // tenta converter para inteiro e retorna esse valor.
    // Se falhar, retorna -1 para sinalizar parse inválido.
    return parts.Length >= 2 && int.TryParse(parts[1], out var code) ? code : -1;
}
```

---

## 5) Conexões, sessão, CSeq e confiabilidade (RFC §4.3, §5-§10)

Pontos relevantes:

- reuso de conexão;
- timeout de sessão/conexão;
- demonstração de liveness;
- sobrecarga e comportamento sob falha.

### 5.1 Como funciona a sessão RTSP

A sessão RTSP nasce normalmente no `SETUP` bem-sucedido.  
Nesse momento, o servidor devolve `Session: <id>[;timeout=<s>]`, e esse ID passa a identificar o contexto de controle.

Na prática, a sessão:

1. agrupa o estado de reprodução (Ready/Play);
2. associa recursos e parâmetros de transporte negociados;
3. define janela de inatividade (`timeout`) para expiração.

#### Sessão agregada vs sessão "normal" (não agregada)

**Sessão agregada** é quando várias mídias/trilhas (ex.: áudio + vídeo) são controladas como um único conjunto lógico, por uma **URI de controle agregada**.

**Sessão não agregada ("normal")** é quando o controle acontece por mídia/trilha individual, cada uma com seu contexto de controle direto na URI da trilha.

Diferenças práticas:

1. **Escopo do comando**
   - Agregada: `PLAY/PAUSE/TEARDOWN` no controle agregado afeta o conjunto inteiro.
   - Não agregada: o comando afeta apenas a trilha alvo.
2. **URI usada**
   - Agregada: use a URI agregada para operações agregadas.
   - Não agregada: use a URI da mídia/trilha.
3. **Erros de operação**
   - Em sessão agregada, tentar operar uma única trilha com método que exige agregado pode gerar `460 Only Aggregate Operation Allowed`.
4. **Sincronização**
   - Agregada facilita manter áudio e vídeo sincronizados sob um único comando de controle.

**Exemplo mental (agregada):**

- Recurso: filme com duas trilhas (`audio` e `video`), com URI agregada `rtsp://exemplo.com/filme`.
- Cliente envia `PLAY rtsp://exemplo.com/filme RTSP/2.0`.
- Resultado: áudio e vídeo iniciam juntos sob o mesmo contexto.
- Cliente envia `PAUSE rtsp://exemplo.com/filme RTSP/2.0`.
- Resultado: as duas trilhas pausam juntas.

**Exemplo mental (não agregada):**

- Recurso: mesmas trilhas, mas controle por URI individual:
  - `rtsp://exemplo.com/filme/audio`
  - `rtsp://exemplo.com/filme/video`
- Cliente envia `PAUSE` apenas para `.../audio`.
- Resultado: somente o áudio pausa; vídeo pode continuar (dependendo da política da aplicação).
- Para pausar tudo, o cliente precisa enviar comando para cada trilha relevante.

Regras práticas:

- se uma requisição de controle exigir sessão e vier sem `Session`, espere erro;
- em sessão agregada, use a URI de controle agregada para operações agregadas;
- `TEARDOWN` encerra ou reduz o escopo da sessão conforme URI e estado atual.

### 5.2 Como funciona o CSeq

`CSeq` (**Command Sequence**) é o contador de requisições RTSP dentro da conexão de controle.  
Ele existe para o cliente e o servidor saberem exatamente **qual resposta pertence a qual requisição**.

Boas práticas de implementação:

1. incrementa 1 por requisição enviada na conexão;
2. resposta deve retornar o mesmo `CSeq` da requisição correspondente;
3. em pipelining, `CSeq` evita ambiguidade quando várias respostas chegam em sequência.
4. se o `CSeq` da resposta não bater, trate como erro de protocolo/correlação.

Exemplo mental rápido:

1. cliente envia `PLAY` com `CSeq: 41`;
2. cliente envia `PAUSE` com `CSeq: 42` (pipelining ou logo em seguida);
3. servidor responde fora de ordem por latência;
4. o cliente usa o `CSeq` para ligar cada resposta ao request certo.

### Snippet C#: gerador de CSeq thread-safe (linha por linha)

```csharp
// Traz o tipo Interlocked para incremento atômico.
using System.Threading;

// Classe fechada para gerar CSeq sem herança.
public sealed class CSeqGenerator
{
    // Campo interno que guarda o último CSeq emitido.
    private int _value;

    // Permite iniciar contador de um valor conhecido (default 0).
    public CSeqGenerator(int startAt = 0) => _value = startAt;

    // Incrementa de forma atômica e retorna o novo valor.
    // "atômico" evita duplicidade quando múltiplas threads enviam requests.
    public int Next() => Interlocked.Increment(ref _value);
}
```

### Snippet C#: validando CSeq da resposta (linha por linha)

```csharp
// Verifica se a resposta pertence à requisição esperada.
// Retorna true quando bate; false quando houver mismatch.
static bool IsExpectedCSeq(IReadOnlyDictionary<string, string> responseHeaders, int expectedCSeq)
{
    // Tenta ler o header "CSeq" da resposta.
    if (!responseHeaders.TryGetValue("CSeq", out var cseqRaw))
        return false; // Sem CSeq: não dá para correlacionar com segurança.

    // Converte o CSeq recebido para inteiro.
    if (!int.TryParse(cseqRaw, out var receivedCSeq))
        return false; // Valor inválido também é erro de protocolo.

    // Confere igualdade entre recebido e esperado.
    return receivedCSeq == expectedCSeq;
}
```

### Snippet C#: controle local de timeout por sessão

```csharp
public sealed class SessionLease
{
    public string SessionId { get; }
    public DateTimeOffset LastActivityUtc { get; private set; }
    public int TimeoutSec { get; }

    public SessionLease(string sessionId, int timeoutSec)
    {
        SessionId = sessionId;
        TimeoutSec = timeoutSec;
        LastActivityUtc = DateTimeOffset.UtcNow;
    }

    public void Touch() => LastActivityUtc = DateTimeOffset.UtcNow;
    public bool IsExpired() => DateTimeOffset.UtcNow - LastActivityUtc > TimeSpan.FromSeconds(TimeoutSec);
}
```

### 5.3 Transporte de live streaming no RTSP

No **live streaming**, RTSP controla *quando e como* receber mídia, enquanto os pacotes de mídia seguem pelo transporte negociado.

Fluxo típico para live:

1. `DESCRIBE` para descobrir faixas e capacidades;
2. `SETUP` com `Transport` (UDP unicast/multicast ou interleaved TCP);
3. `PLAY` para iniciar entrega ao vivo;
4. `PAUSE`/`PLAY` para pausar e retomar (quando o tipo de mídia permitir);
5. `TEARDOWN` para liberar recursos.

#### Tipos de transporte no SETUP

- **UDP unicast**: servidor envia RTP/RTCP para **um cliente específico** (IP/porta daquele cliente).  
  É comum em cenários 1:1 e costuma ter baixa latência.

- **UDP multicast**: servidor envia um único fluxo para um **grupo multicast**, e vários clientes recebem ao mesmo tempo.  
  É eficiente para 1:n, mas depende de suporte de rede (roteamento/switches/políticas).

- **Interleaved TCP**: RTP/RTCP vai encapsulado no **mesmo canal TCP do RTSP** (frames interleaved, `$`).  
  Controle e mídia passam pela mesma conexão.

**Quando cada um leva vantagem:**

1. **UDP unicast**  
   Vantagem quando você quer baixa latência por cliente e tem controle de rede/portas.  
   Bom para sessões individuais (1:1), especialmente em LAN ou ambientes administrados.

2. **UDP multicast**  
   Vantagem quando muitos clientes precisam do **mesmo stream ao mesmo tempo** (1:n).  
   Reduz tráfego e CPU no servidor, porque um único fluxo atende vários receptores.

3. **Interleaved TCP**  
   Vantagem quando há NAT/firewall restritivo, rede corporativa rígida ou falha recorrente em UDP.  
   Simplifica conectividade por usar uma única conexão TCP RTSP já permitida.

Pontos importantes da RFC para live:

- para conteúdo time-progressing, o ponto "agora" é dinâmico;
- após pausa em alguns cenários live, retomar pode exigir `Range: npt=now-`;
- em redes restritas por NAT/firewall, interleaved no canal RTSP pode simplificar conectividade;
- sincronização de mídia normalmente depende de informações como `RTP-Info` e relógio RTP/NPT.

**Por que interleaved ajuda com NAT/firewall:**  
em UDP unicast/multicast, além da conexão de controle RTSP, a rede precisa liberar portas UDP extras para RTP/RTCP (e manter mapeamentos NAT ativos). Em muitos ambientes corporativos isso é bloqueado ou instável.  
Com interleaved TCP, tudo passa pela conexão RTSP já estabelecida (normalmente uma porta TCP permitida), reduzindo regras de firewall e problemas de mapeamento NAT.

### Snippet C#: escolhendo transporte para live

```csharp
static string ChooseLiveTransport(bool preferTcpInterleaved, int rtpPort = 5000, int rtcpPort = 5001)
{
    if (preferTcpInterleaved)
        return "RTP/AVP/TCP;unicast;interleaved=0-1";

    return $"RTP/AVP;unicast;dest_addr=\":{rtpPort}\"/\":{rtcpPort}\"";
}
```

---

## 6) Capacidades e pipelining (RFC §11-§12)

### 6.1 Capability handling

Capability handling é a negociação explícita do que cliente e servidor realmente suportam.
Sem isso, o cliente pode enviar métodos/headers opcionais e receber falhas de interoperabilidade.

**Cabeçalhos-chave nessa negociação:**

- `Public`: lista de métodos aceitos pelo recurso/servidor (`OPTIONS`, `DESCRIBE`, `SETUP`, `PLAY`, etc.).
- `Supported`: lista de extensões/feature-tags que a outra ponta entende.
- `Require`: extensões obrigatórias para processar aquela requisição.
- `Proxy-Require`: semelhante ao `Require`, mas exigindo suporte também no caminho por proxies RTSP.

Fluxo prático recomendado:

1. Envie `OPTIONS` no início da sessão (ou quando mudar de recurso).
2. Leia `Public` para saber quais métodos são válidos antes de montar o fluxo.
3. Se uma funcionalidade for opcional, ative apenas quando aparecer em `Supported`.
4. Use `Require` só quando a operação depender daquela extensão (caso contrário, prefira fallback).

Se o servidor não suportar uma extensão exigida, a resposta tende a sinalizar erro de capability
(por exemplo, com indicação de recurso não suportado). Nesses casos, o cliente robusto deve
rebaixar a estratégia (fallback) ou ajustar o fluxo de métodos.

### Snippet C#: checando capacidades antes de habilitar uma feature

```csharp
static bool SupportsFeature(string? supportedHeader, string featureTag)
{
    if (string.IsNullOrWhiteSpace(supportedHeader))
        return false;

    return supportedHeader
        .Split(',', StringSplitOptions.TrimEntries | StringSplitOptions.RemoveEmptyEntries)
        .Any(tag => string.Equals(tag, featureTag, StringComparison.OrdinalIgnoreCase));
}
```

### 6.2 Pipelining

Pipelining permite enviar várias requisições RTSP na mesma conexão sem esperar a resposta de cada uma.
O ganho principal é reduzir latência de ida-e-volta (RTT), especialmente quando você faz sequências como
`OPTIONS` -> `DESCRIBE` -> múltiplos `SETUP`.

Em cliente real, três regras evitam bugs:

1. Cada requisição deve ter `CSeq` único e monotônico por conexão.
2. Toda resposta deve ser correlacionada pelo `CSeq`, nunca por "ordem esperada" no código de negócio.
3. Cada requisição pendente precisa de timeout/cancelamento para não vazar recursos.

Quando usar:
- links de alta latência (WAN, internet pública, VPN);
- sequência de inicialização com várias chamadas curtas.

Quando ter cuidado:
- operações com forte dependência de estado do servidor (ex.: enviar `PLAY` antes de confirmar `SETUP`);
- servidores legados que anunciam suporte parcial e se comportam mal com muitas pendências.

### Snippet C#: correlacionando respostas por CSeq

```csharp
// Guarda, por CSeq, a promessa da resposta pendente.
var pending = new Dictionary<int, TaskCompletionSource<string>>();

// Registra a requisição enviada para correlação futura.
void Register(int cseq, TaskCompletionSource<string> tcs) => pending[cseq] = tcs;

// Chamado quando uma resposta RTSP completa chega do socket/parser.
void OnResponseReceived(int cseq, string rawResponse)
{
    // Remove e recupera a pendência em um único passo (evita dupla conclusão).
    if (pending.Remove(cseq, out var tcs))
        // Conclui a Task de quem enviou a requisição correspondente.
        tcs.TrySetResult(rawResponse);
}
```

### Snippet C#: enviando requisição pipelined com `CSeq` incremental

```csharp
// Contador de CSeq por conexão RTSP.
int _nextCSeq = 0;

// Envia request sem bloquear a conexão e devolve Task da resposta correlacionada.
Task<string> SendPipelinedAsync(string method, string uri)
{
    // Gera CSeq único de forma atômica (thread-safe).
    var cseq = Interlocked.Increment(ref _nextCSeq);
    // Promessa da resposta; continuações assíncronas evitam bloquear I/O.
    var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);
    // Registra antes de enviar para não perder resposta que chegue muito rápido.
    Register(cseq, tcs);

    // Monta a mensagem RTSP com method/uri/CSeq.
    var request = BuildRequest(method, uri, cseq);
    // Escreve no socket da conexão atual.
    socket.Send(request);
    // Quem chamou aguarda esta Task até a resposta do mesmo CSeq.
    return tcs.Task;
}
```

---

## 7) Métodos RTSP (RFC §13)

Este é o núcleo operacional do protocolo: os métodos definem **o ciclo de vida da sessão**,
da descoberta de capacidades até o encerramento.

Em implementação prática, pense nesses métodos em três blocos:

1. **Descoberta e preparação**: `OPTIONS` e `DESCRIBE`.
2. **Negociação e execução**: `SETUP`, `PLAY`, `PAUSE`.
3. **Operação e encerramento**: `GET_PARAMETER`, `SET_PARAMETER`, `TEARDOWN`, `REDIRECT`.

Uma sessão robusta normalmente segue a ordem:
`OPTIONS` -> `DESCRIBE` -> `SETUP` (por trilha) -> `PLAY` -> (`PAUSE`/`PLAY` conforme necessário) -> `TEARDOWN`.

Dois princípios evitam grande parte dos erros:

- sempre correlacionar request/response por `CSeq` e validar `Session` quando aplicável;
- tratar métodos como transições de estado (por exemplo, `PLAY` antes de `SETUP` tende a falhar).

### 7.1 OPTIONS (§13.1)

**Para que serve:** descobrir métodos e capacidades suportadas pelo servidor/recurso.  
**Quando usar:** início da sessão e diagnóstico de interoperabilidade.  
**Esperado na resposta:** `Public` com lista de métodos (e possivelmente indicações de features).

`OPTIONS` é o ponto de entrada mais seguro para adaptar o cliente ao servidor real, em vez de
assumir suporte universal. Ele permite responder perguntas como:

- este endpoint aceita `GET_PARAMETER`/`SET_PARAMETER`?
- há suporte a extensões anunciadas por `Supported`?
- o método está disponível no recurso agregado, na trilha, ou em ambos?

Boas práticas para `OPTIONS`:

1. Enviar no URI agregado no início da sessão.
2. Repetir em URI de trilha quando houver dúvida de suporte por recurso.
3. Usar o resultado para habilitar/desabilitar fluxos opcionais do cliente.
4. Em ausência de `Public`, operar com fallback conservador e log explícito.

Erros típicos:

- assumir que todo servidor suporta todos os métodos da RFC;
- ignorar diferença entre capacidade global do servidor e capacidade por recurso específico;
- não ajustar o fluxo quando uma feature opcional não aparece nos headers de capability.

```csharp
// Monta um OPTIONS para descobrir métodos/capacidades do recurso.
var options = BuildRequest("OPTIONS", "rtsp://example.com/media", 1);
```

### 7.2 DESCRIBE (§13.2)

**Para que serve:** obter descrição do recurso (tipicamente SDP).  
**Quando usar:** antes de `SETUP`, para saber trilhas e URIs de controle.  
**Cabeçalhos importantes:** `Accept` (ex.: `application/sdp`).  
**Erros comuns:** tipo de descrição não suportado.

`DESCRIBE` é onde o cliente descobre a topologia real da mídia: quais trilhas existem,
quais codecs/formats são anunciados e quais URIs de controle devem ser usadas depois.
Em prática, quase toda decisão de `SETUP` nasce da leitura correta desse payload.

Boas práticas:

1. Sempre enviar `Accept` explícito para evitar ambiguidade de formato de descrição.
2. Validar se o payload retornado contém trilhas coerentes com o que o app espera (áudio/vídeo).
3. Diferenciar URI agregada da sessão e URIs de trilha para não enviar `SETUP` no alvo errado.
4. Falhar cedo quando descrição vier incompleta, em vez de seguir para `SETUP` inconsistente.

```csharp
var describe = BuildRequest("DESCRIBE", "rtsp://example.com/media", 2,
    new Dictionary<string, string> { ["Accept"] = "application/sdp" });
```

### 7.3 SETUP (§13.3)

**Para que serve:** negociar transporte e criar/associar sessão RTSP.  
**Quando usar:** após `DESCRIBE`, para cada mídia/trilha necessária.  
**Cabeçalhos importantes:** `Transport`, `Accept-Ranges`.  
**Esperado na resposta:** `Session`, `Transport` selecionado, além de cabeçalhos de capacidades temporais.

`SETUP` é o método que transforma "descrição" em "sessão ativa". Em sessões com múltiplas trilhas,
o cliente executa um `SETUP` por trilha; normalmente a primeira resposta cria a `Session` e as próximas
associam novas trilhas ao mesmo identificador.

Pontos críticos de interoperabilidade:

1. O `Transport` enviado é uma proposta; o servidor pode ajustar parâmetros na resposta.
2. O cliente deve consumir o `Transport` efetivo retornado (portas/canais realmente aceitos).
3. Quando UDP falhar por rede restrita, fallback para interleaved TCP costuma ser o caminho robusto.
4. Sem `Session` válida, não prossiga para `PLAY`.

```csharp
var setup = BuildRequest("SETUP", "rtsp://example.com/media/track1", 3,
    new Dictionary<string, string>
    {
        ["Transport"] = "RTP/AVP;unicast;dest_addr=\":5000\"/\":5001\"",
        ["Accept-Ranges"] = "npt, clock"
    });
```

### 7.4 PLAY (§13.4)

**Para que serve:** iniciar ou retomar a transmissão.  
**Quando usar:** sessão em `Ready` (ou atualização durante `Play` conforme cenário).  
**Cabeçalhos importantes:** `Session`, opcionalmente `Range` e `Seek-Style`.  
**Esperado na resposta:** `Range` efetivo (ajustado pelo servidor), e possivelmente `RTP-Info`.  
**Erro clássico:** `457 Invalid Range` quando início/fim pedido é inválido.

`PLAY` comanda o ponto temporal e o início efetivo de entrega. Sem `Range`, o servidor tende a aplicar
comportamento padrão do recurso (início natural, ou "agora" em cenários live). Com `Range`, o cliente
pede recorte temporal explícito, e o servidor pode ajustar esse recorte para limites válidos.

Checklist prático antes de enviar:

1. Confirmar `Session` ativa.
2. Confirmar que todas as trilhas necessárias já passaram por `SETUP`.
3. Definir estratégia de seek (`Seek-Style`) quando precisão temporal for relevante.
4. Tratar `RTP-Info` da resposta para sincronização inicial de decodificação.

**O que é `Seek-Style`:**  
é o header usado no `PLAY` (com `Range`) para expressar **como** o servidor deve escolher
o ponto real de início quando há acesso aleatório (random access) na mídia.

Políticas definidas na RFC:

1. `RAP` (Random Access Point): inicia no RAP anterior mais próximo do ponto pedido.  
   Melhor qualidade de decodificação inicial, pois começa em ponto seguro de referência.
2. `CoRAP` (Conditional RAP): usa RAP **se** ele estiver mais próximo do alvo do que do pause point atual;  
   caso contrário continua do pause point (na prática, pode virar comportamento `Next` na resposta).
3. `First-Prior`: inicia na primeira unidade de mídia imediatamente anterior ao tempo pedido.
4. `Next`: inicia na primeira unidade **após** o tempo pedido (útil para continuar sem sobreposição grande).

Notas de interoperabilidade:

- para mídias com propriedades de random access, o servidor deve retornar o `Seek-Style` aplicado na resposta `PLAY`;
- se cliente/servidor receber política desconhecida, deve ignorar e seguir com fallback padrão;
- `Next` só deve ser usado por solicitação explícita do cliente.

```csharp
var play = BuildRequest("PLAY", "rtsp://example.com/media", 4,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Range"] = "npt=10-30",
        ["Seek-Style"] = "RAP"
    });
```

### 7.5 PLAY_NOTIFY (§13.5)

**Para que serve:** notificação assíncrona servidor->cliente sobre eventos de reprodução  
(fim de stream, atualização de propriedades de mídia, troca de escala etc.).  
**Quando usar:** não é enviado pelo cliente; o cliente precisa saber consumir/tratar.

`PLAY_NOTIFY` fecha uma lacuna importante: nem todo evento relevante cabe no fluxo estritamente
request/response iniciado pelo cliente. O servidor pode avisar mudanças de estado da reprodução
sem esperar polling ativo.

**Como a notificação chega no cliente (ponto que costuma confundir):**

- em RTSP 2.0, a conexão de controle é **bidirecional**;
- isso significa que, na mesma conexão TCP já aberta pelo cliente, o servidor pode enviar requests RTSP;
- `PLAY_NOTIFY` é justamente um desses requests servidor->cliente.

Na prática, o cliente mantém **uma** conexão RTSP de controle e um loop de leitura que aceita dois tipos
de mensagem de entrada:

1. respostas às requisições que o cliente enviou (`CSeq` correlacionado);
2. requisições iniciadas pelo servidor (como `PLAY_NOTIFY`), que o cliente deve processar e responder.

Então, para `PLAY_NOTIFY`, **não é obrigatório abrir uma segunda conexão TCP** só para notificação.
O que muda é a lógica do parser/dispatcher do cliente: ele não pode assumir que toda mensagem recebida
é resposta.  

Observação: o transporte de mídia (RTP/RTCP) continua independente dessa regra e pode ser UDP ou
interleaved no mesmo TCP RTSP. Isso não altera o fato de que o `PLAY_NOTIFY` chega pelo canal de
controle RTSP.

Boas práticas de tratamento:

1. Processar de forma assíncrona, sem bloquear o loop de I/O principal.
2. Correlacionar com a sessão/trilha correta antes de atualizar estado local.
3. Tornar o tratamento idempotente, porque notificações podem repetir em cenários reais.
4. Registrar telemetria para diagnóstico de fim de stream e mudanças inesperadas.

```csharp
// Diferencia rapidamente se a mensagem recebida é request RTSP (ex.: PLAY_NOTIFY).
static bool IsServerRequest(string startLine) =>
    !startLine.StartsWith("RTSP/2.0 ", StringComparison.Ordinal);

// Trata requests vindos do servidor no mesmo canal de controle.
void OnServerRequestReceived(RtspRequest req)
{
    if (req.Method.Equals("PLAY_NOTIFY", StringComparison.OrdinalIgnoreCase))
    {
        HandlePlayNotify(req.Headers, req.Body);  // Atualiza estado local do player.
        SendResponse(req.CSeq, 200, "OK");        // Cliente responde ao request do servidor.
        return;
    }

    SendResponse(req.CSeq, 405, "Method Not Allowed");
}
```

### 7.6 PAUSE (§13.6)

**Para que serve:** interromper imediatamente e fixar pause point.  
**Quando usar:** sessão em `Play` (e idempotência prática em certos cenários já em `Ready`).  
**Esperado na resposta:** `Range` com ponto atual/trecho remanescente.  
**Atenção live:** para mídia time-progressing sem retenção, retomada pode exigir `npt=now-`.

`PAUSE` preserva contexto de sessão sem liberar tudo como em `TEARDOWN`.
Isso permite retomar rápido com `PLAY`, desde que o modelo temporal do conteúdo permita.

Cuidados práticos:

1. Persistir localmente o ponto de pausa retornado pelo servidor.
2. Em live sem buffer de retenção, preparar fallback para retomada relativa ao "agora".
3. Evitar assumir que `PAUSE` congela estado de rede/transporte por tempo indefinido.
4. Se a pausa for longa, revalidar sessão/keepalive antes do `PLAY` de retomada.

```csharp
var pause = BuildRequest("PAUSE", "rtsp://example.com/media", 5,
    new Dictionary<string, string> { ["Session"] = "abcd1234" });
```

### 7.7 TEARDOWN (§13.7)

**Para que serve:** encerrar entrega e liberar recursos.  
**Quando usar:** final de reprodução, troca de conteúdo, limpeza de sessão.  
**Comportamento:** pode destruir sessão inteira ou remover mídia específica (dependendo de URI/estado e se sessão é agregada).

`TEARDOWN` é o encerramento explícito do ciclo. Em produção, enviar esse método de forma previsível
evita sessões órfãs no servidor e reduz consumo desnecessário de recursos.

Regras práticas:

1. Use URI agregada para encerrar sessão completa quando esse for o objetivo.
2. Use URI de trilha apenas quando quiser remover mídia específica.
3. Trate timeout/falha de rede com cleanup local mesmo sem resposta final.
4. Após `TEARDOWN` bem-sucedido, invalide `Session` no cliente para impedir reuso acidental.

```csharp
var teardown = BuildRequest("TEARDOWN", "rtsp://example.com/media", 6,
    new Dictionary<string, string> { ["Session"] = "abcd1234" });
```

### 7.8 GET_PARAMETER (§13.8)

**Para que serve:** consultar parâmetros de sessão/recurso (estado, métricas etc.).  
**Quando usar:** monitoramento, keepalive em implementações que adotam esse padrão, telemetria de sessão.

`GET_PARAMETER` é útil para observabilidade e manutenção de sessão sem alterar estado de mídia.
Dependendo do servidor, pode ser usado como heartbeat leve ou consulta de indicadores operacionais.

**Canal de conexão (comparação com `PLAY_NOTIFY`):**

- `GET_PARAMETER` usa o **mesmo canal TCP RTSP de controle** já aberto para `PLAY`/`PAUSE`/`TEARDOWN`.
- Não exige segunda conexão TCP só para observabilidade.
- Diferença principal:
  - `GET_PARAMETER`: modelo **pull** (cliente solicita, servidor responde).
  - `PLAY_NOTIFY`: modelo **push** (servidor inicia notificação assíncrona).

**Valores/formatos possíveis no `GET_PARAMETER`:**

1. **Sem parâmetros** (sem body e sem headers de consulta):  
   serve como keepalive da sessão (se o servidor responder `200 OK`).
2. **Via headers de consulta** (quando suportado pelo servidor):  
   a RFC lista `Accept-Ranges`, `Media-Range`, `Media-Properties`, `Range` e `RTP-Info`.
3. **Via body `text/parameters`** (mais comum e mais claro):  
   cliente envia nomes de parâmetros e o servidor responde os mesmos nomes preenchidos com valor.

Formato típico em body:

- Requisição: uma linha por parâmetro (`jitter`, `packet_loss`, `packets_received` etc.).
- Resposta: pares `nome: valor`.

Se o media type do body não for suportado, o esperado é `415 Unsupported Media Type`.
Se algum parâmetro não for entendido, o esperado é `451 Parameter Not Understood`.

**Coisas interessantes para fazer com `GET_PARAMETER`:**

1. **Telemetria de qualidade em tempo real**: consultar `jitter`, perda e pacotes recebidos para alimentar dashboard.
2. **Adaptação de UX/rede**: se jitter/perda subir, reduzir resolução/bitrate no app ou trocar estratégia de transporte.
3. **Detecção precoce de degradação**: acompanhar tendência antes de travar playback e atuar proativamente.
4. **Health-check de sessão**: heartbeat periódico sem mexer no estado de reprodução.

**Recuperação de observabilidade com `GET_PARAMETER` funciona assim:**

1. Cliente envia `GET_PARAMETER` com `Session` e body `text/parameters` listando métricas (ex.: `jitter`, `packet_loss`, `packets_received`).
2. Servidor responde `200 OK` no mesmo canal, com body contendo `nome: valor`.
3. Cliente faz parse dos pares e publica em logs/métricas/dashboard.
4. Repete em intervalo periódico (ex.: 1s, 2s, 5s), com timeout curto e backoff em falha.

**Exemplo de payload:**

**Request body**

```text
jitter
packet_loss
packets_received
```

**Response body**

```text
jitter: 0.38
packet_loss: 0.02
packets_received: 10234
```

Se o parâmetro não for suportado, o servidor pode responder `451 Parameter Not Understood`; se o tipo de body não for suportado, `415 Unsupported Media Type`.

Boas práticas:

1. Padronizar nomes de parâmetros consultados no cliente para facilitar compatibilidade.
2. Tratar ausência de parâmetro como dado "não suportado", não como erro fatal automático.
3. Separar chamadas de keepalive das chamadas de telemetria para controlar frequência.
4. Aplicar timeout curto para não competir com métodos críticos (`PLAY`/`PAUSE`/`TEARDOWN`).

```csharp
// Exemplo 1: consulta de métricas via body text/parameters.
var getParameter = BuildRequest("GET_PARAMETER", "rtsp://example.com/media", 7,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "stream_state\r\njitter\r\npacket_loss\r\n");
```

```csharp
// Exemplo 2: keepalive simples sem body (somente para renovar liveness da sessão).
var keepAlive = BuildRequest("GET_PARAMETER", "rtsp://example.com/media", 8,
    new Dictionary<string, string> { ["Session"] = "abcd1234" });
```

### 7.9 SET_PARAMETER (§13.9)

**Para que serve:** atualizar parâmetros controláveis do recurso/sessão.  
**Quando usar:** ajustes em runtime (por exemplo volume, preferências de entrega, sinalizações de app).  
**Atenção:** validar conteúdo e permissões para evitar inconsistência de estado.

`SET_PARAMETER` é o canal de controle de propriedades dinâmicas sem renegociar toda a sessão.
Ele é útil para comandos de aplicação, mas exige disciplina de validação para evitar estados parciais.

Pontos de implementação:

1. Definir contrato claro de payload (`Content-Type`, nomes, formato e faixas válidas).
2. Rejeitar valores inválidos com erro explícito, sem aplicar atualização parcial silenciosa.
3. Confirmar no cliente o estado aplicado (por resposta direta ou leitura posterior).
4. Restringir parâmetros sensíveis por política/autorização.

Exemplos práticos além de volume:

1. alternar trilha de áudio/idioma durante reprodução;
2. habilitar/desabilitar legenda;
3. ajustar bitrate alvo em cenários adaptativos;
4. sinalizar modo de latência (`low-latency`) para perfis de entrega;
5. enviar metadados de aplicação (por exemplo, identificação de dispositivo/cliente).

```csharp
// Exemplo 1: ajuste de volume.
var setParameter = BuildRequest("SET_PARAMETER", "rtsp://example.com/media", 8,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "volume: 0.6\r\n");
```

```csharp
// Exemplo 2: troca de trilha de áudio e idioma preferido.
var setAudioTrack = BuildRequest("SET_PARAMETER", "rtsp://example.com/media", 9,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "audio_track: 2\r\nlanguage: pt-BR\r\n");
```

```csharp
// Exemplo 3: habilita legenda e seleciona trilha.
var setSubtitle = BuildRequest("SET_PARAMETER", "rtsp://example.com/media", 10,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "subtitle_enabled: true\r\nsubtitle_track: 1\r\n");
```

```csharp
// Exemplo 4: ajuste de bitrate alvo e modo de baixa latência.
var setAdaptiveProfile = BuildRequest("SET_PARAMETER", "rtsp://example.com/media", 11,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "target_bitrate_kbps: 1800\r\nlow_latency: true\r\n");
```

### 7.10 REDIRECT (§13.10)

**Para que serve:** instruir o cliente a migrar para outro endpoint/URI.  
**Quando usar:** manutenção, balanceamento, reorganização de serviço.  
**Cabeçalho-chave:** `Location` com destino de redirecionamento.

`REDIRECT` permite mover clientes de forma coordenada sem depender de falha abrupta.
É comum em cenários de manutenção planejada, drenagem de nó ou rebalanceamento de carga.

Fluxo recomendado no cliente:

1. Validar `Location` recebido (formato, esquema e destino permitido pela política).
2. Preparar nova conexão e repetir ciclo de setup no endpoint de destino.
3. Trocar para novo fluxo de mídia com mínima interrupção possível.
4. Encerrar sessão antiga com `TEARDOWN` quando a migração estiver estável.

```csharp
static Uri ParseRedirectTarget(IReadOnlyDictionary<string, string> headers) =>
    new(headers["Location"]);
```

---

## 8) Dados binários interleaved (RFC §14)

Quando RTP/RTCP é interleaved no mesmo canal RTSP, frames binários e mensagens RTSP texto
compartilham o mesmo stream TCP.  
O cliente/servidor precisa demultiplexar corretamente para não confundir mídia com controle.

Formato do frame interleaved:

1. byte 0: marcador `$` (`0x24`);
2. byte 1: canal (`channel id`);
3. bytes 2-3: tamanho do payload em big-endian;
4. bytes seguintes: payload RTP/RTCP com o tamanho informado.

Exemplo: em `interleaved=0-1`, normalmente canal `0` carrega RTP e canal `1` carrega RTCP
(a associação exata vem do `Transport` negociado no `SETUP`).

Regras práticas de parsing:

1. Se o próximo byte for `$`, leia exatamente 4 bytes de cabeçalho e depois `length` bytes de payload.
2. Se não for `$`, trate como mensagem RTSP textual (`Request`/`Response`).
3. Nunca assuma que um `recv()` devolve frame completo: acumule em buffer até completar.
4. Valide limites de tamanho para evitar consumo excessivo de memória em payload malformado.
5. Faça dispatch por canal para a pilha RTP/RTCP correta.

### Snippet C#: reconhecendo frame interleaved

```csharp
// Formato interleaved começa com '$', seguido por canal e tamanho.
static bool TryReadInterleavedHeader(ReadOnlySpan<byte> buf, out byte channel, out ushort length)
{
    channel = 0;
    length = 0;
    if (buf.Length < 4 || buf[0] != (byte)'$') return false;
    channel = buf[1];
    length = (ushort)((buf[2] << 8) | buf[3]);
    return true;
}
```

### Snippet C#: demultiplexando RTSP texto x frame interleaved

```csharp
// Processa o buffer de entrada e separa mensagens RTSP de frames interleaved.
void ConsumeIncomingBuffer(List<byte> buffer)
{
    while (buffer.Count > 0)
    {
        // Se não começar com '$', o próximo item é mensagem RTSP textual.
        if (buffer[0] != (byte)'$')
        {
            if (!TryExtractRtspMessage(buffer, out var rawRtsp))
                return; // Aguarda mais bytes para completar headers/body RTSP.

            HandleRtspMessage(rawRtsp);
            continue;
        }

        // Para frame interleaved, são necessários pelo menos 4 bytes de cabeçalho.
        if (buffer.Count < 4) return;

        var channel = buffer[1];
        var length = (buffer[2] << 8) | buffer[3];
        var total = 4 + length;

        // Aguarda até ter o frame completo no buffer.
        if (buffer.Count < total) return;

        var payload = buffer.GetRange(4, length).ToArray();
        buffer.RemoveRange(0, total);

        // Encaminha por canal para a lógica RTP/RTCP correspondente.
        HandleInterleavedFrame(channel, payload);
    }
}
```

---

## 9) Proxies (RFC §15)

Proxies RTSP ficam no meio entre cliente e servidor e precisam preservar a semântica do protocolo
em cada hop, sem "quebrar" estado de sessão, capacidades e transporte.

Papéis típicos de um proxy:

1. roteamento para upstream correto (sharding/região/política);
2. controle de acesso/autenticação no edge;
3. observabilidade e limitação de tráfego;
4. normalização mínima de mensagens para interoperabilidade.

Responsabilidades de interoperabilidade:

1. preservar correlação de mensagens (`CSeq`, `Session`) e ordem lógica por conexão;
2. encaminhar/negociar extensões com `Supported`, `Require`, `Proxy-Supported` e `Proxy-Require`;
3. tratar corretamente `REDIRECT`, `PLAY_NOTIFY` e outros métodos menos comuns;
4. em modo interleaved, demultiplexar/remultiplexar sem corromper framing binário.

Cuidados práticos:

- manter timeout e lifecycle de sessão coerentes entre os dois lados;
- evitar reescrever headers sem necessidade;
- registrar decisões de roteamento para diagnóstico;
- aplicar limites de tamanho/taxa para proteger upstream.

### Snippet C#: exemplo simplificado de roteamento por URI

```csharp
static string PickUpstream(Uri requestUri) =>
    requestUri.Host.EndsWith(".interna.example", StringComparison.OrdinalIgnoreCase)
        ? "rtsp://upstream-a.internal"
        : "rtsp://upstream-b.internal";
```

### Snippet C#: preservando headers críticos ao encaminhar

```csharp
// Headers que o proxy não deve perder durante o forward.
static readonly string[] CriticalHeaders =
{
    "CSeq", "Session", "Transport", "Range", "Require", "Supported",
    "Proxy-Require", "Proxy-Supported"
};

// Copia headers relevantes para o request de upstream.
static Dictionary<string, string> BuildForwardHeaders(IReadOnlyDictionary<string, string> incoming)
{
    var forwarded = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);

    foreach (var name in CriticalHeaders)
        if (incoming.TryGetValue(name, out var value))
            forwarded[name] = value;

    return forwarded;
}
```

---

## 10) Caching e validação (RFC §16)

RTSP define modelo de validação com datas e tags de corpo.  
Implementações com cache precisam invalidar após updates/deletes conforme regras da RFC.

### Snippet C#: chave de cache para DESCRIBE

```csharp
static string BuildDescribeCacheKey(string uri, string? etag, string? lastModified) =>
    $"{uri}|etag={etag ?? "-"}|lm={lastModified ?? "-"}";
```

---

## 11) Códigos de status (RFC §17)

Famílias:

- `1xx` informacional
- `2xx` sucesso
- `3xx` redirecionamento
- `4xx` erro do cliente
- `5xx` erro do servidor

Códigos muito frequentes na prática RTSP:

- `200 OK`
- `454 Session Not Found`
- `455 Method Not Valid in This State`
- `456 Header Field Not Valid for Resource`
- `457 Invalid Range`
- `459 Aggregate Operation Not Allowed`
- `460 Only Aggregate Operation Allowed`

### Snippet C#: mapeando status para decisão

```csharp
static string Classify(int code) => code switch
{
    >= 100 and < 200 => "Informational",
    >= 200 and < 300 => "Success",
    >= 300 and < 400 => "Redirection",
    >= 400 and < 500 => "ClientError",
    >= 500 and < 600 => "ServerError",
    _ => "Unknown"
};
```

---

## 12) Cabeçalhos RTSP (RFC §18)

O RFC 7826 define um conjunto extenso de cabeçalhos.  
Os mais críticos para começar:

- `CSeq`
- `Session`
- `Transport`
- `Range`
- `Accept-Ranges`
- `Media-Range`
- `Media-Properties`
- `RTP-Info`
- `Seek-Style`
- `Terminate-Reason`

### Snippet C#: parser genérico de cabeçalhos

```csharp
static Dictionary<string, string> ParseHeaders(IEnumerable<string> lines)
{
    var map = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
    foreach (var line in lines)
    {
        var idx = line.IndexOf(':');
        if (idx <= 0) continue;
        map[line[..idx].Trim()] = line[(idx + 1)..].Trim();
    }
    return map;
}
```

---

## 13) ABNF e segurança (RFC §19-§21)

### 13.1 ABNF e conformidade sintática

Para interoperabilidade real, ABNF não é detalhe: é o contrato que impede ambiguidades entre
implementações de cliente, servidor e proxy.

Diretriz prática: seja **estrito no que valida** e **explícito no erro**.
Aceitar mensagens malformadas "porque funciona" costuma gerar bugs difíceis de reproduzir.

Pontos que o parser deve validar sempre:

1. **Start-Line** válida (`Request-Line` ou `Status-Line`) com versão RTSP coerente.
2. **Headers** no formato `Nome: valor`, com término de linha correto.
3. **Content-Length** consistente com o corpo recebido (nem truncado, nem bytes sobrando).
4. **CSeq** presente e numérico em mensagens que exigem correlação.
5. **Session** quando o método/estado exigir contexto de sessão.

Onde ser tolerante (com cuidado):

- ordem dos headers;
- capitalização de nomes de headers (case-insensitive);
- espaços extras onde a gramática permitir.

Quando rejeitar:

- linha inicial inválida;
- header sem `:`;
- `Content-Length` inválido;
- campos críticos fora de política/estado.

Retorne código de erro compatível (por exemplo `400`, `451`, `455`, `456`, `457`) em vez de
"corrigir silenciosamente" entrada inválida.

### Snippet C#: validação mínima de Start-Line RTSP

```csharp
// Verifica se é uma Status-Line RTSP/2.0 com código numérico.
static bool IsValidStatusLine(string line)
{
    var parts = line.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    return parts.Length >= 3 &&
           parts[0].Equals("RTSP/2.0", StringComparison.Ordinal) &&
           int.TryParse(parts[1], out _);
}
```

### 13.2 Considerações de segurança

Boas práticas:

1. validar tamanho de cabeçalhos e corpo;
2. limitar recursos por sessão/conexão;
3. tratar redirecionamento com política explícita;
4. evitar parsing permissivo para campos críticos.

A superfície de ataque em RTSP aparece em três frentes:

1. **entrada malformada** (flood, header/body gigante, framing inconsistente);
2. **transição de estado indevida** (`PLAY`/`PAUSE`/`TEARDOWN` fora do estado esperado);
3. **controle de rota/sessão** (abuso de `REDIRECT`, sequestro de `Session`, replay de requests).

Controles recomendados em produção:

- limite por conexão (mensagens/s, bytes/s, requests pendentes);
- timeout de leitura/escrita e timeout de sessão inativa;
- validação de destino em `REDIRECT` (allowlist de host/esquema/porta);
- logging estruturado com `CSeq`, `Session`, método e status para auditoria;
- descarte explícito de mensagens que violem framing interleaved (`$`, canal, length).

Em `GET_PARAMETER`/`SET_PARAMETER`, trate payload como entrada não confiável:
valide nomes permitidos, tipos e faixas; rejeite parâmetros desconhecidos com erro explícito.

### Snippet C#: guarda contra mensagens excessivas

```csharp
static void EnforceMessageLimits(int headerBytes, int bodyBytes, int maxHeaderBytes = 32 * 1024, int maxBodyBytes = 256 * 1024)
{
    if (headerBytes > maxHeaderBytes) throw new InvalidOperationException("RTSP header too large.");
    if (bodyBytes > maxBodyBytes) throw new InvalidOperationException("RTSP body too large.");
}
```

---

## 14) IANA, interoperabilidade e estratégia de implementação (RFC §22 + apêndices)

### 14.1 Registros e extensões

A RFC 7826 organiza métodos, cabeçalhos e status em registros IANA, facilitando extensões compatíveis.

### 14.2 Estratégia incremental recomendada

1. Implementar parser/montagem de mensagens (`CSeq`, `Session`, `Transport`, `Range`).
2. Fechar fluxo principal `OPTIONS -> DESCRIBE -> SETUP -> PLAY -> PAUSE -> TEARDOWN`.
3. Adicionar `GET_PARAMETER`/`SET_PARAMETER` e tratamento completo de erros `4xx/5xx`.
4. Incluir interleaving (`$` frames), redirecionamento e comportamento com proxy/cache.

### Snippet C#: máquina de estados mínima de sessão

```csharp
public enum RtspSessionState { Init, Ready, Play, Terminated }

static RtspSessionState Apply(RtspSessionState state, string method, int statusCode) =>
    (state, method.ToUpperInvariant(), statusCode) switch
    {
        (RtspSessionState.Init, "SETUP", 200) => RtspSessionState.Ready,
        (RtspSessionState.Ready, "PLAY", 200) => RtspSessionState.Play,
        (RtspSessionState.Play, "PAUSE", 200) => RtspSessionState.Ready,
        (_, "TEARDOWN", 200) => RtspSessionState.Terminated,
        _ => state
    };
```

---

## Capítulo especial: PAUSE (RFC §13.6), em termos de implementação

Você citou corretamente o `PAUSE` como exemplo importante.  
Ele concentra uma regra operacional crucial: o servidor interrompe envio e devolve ponto de pausa em `Range`.

### Snippet C#: extraindo `Range` da resposta de PAUSE

```csharp
public readonly record struct PauseOutcome(bool Success, string? Range);

static PauseOutcome ParsePauseResponse(string raw)
{
    var lines = raw.Split("\r\n", StringSplitOptions.RemoveEmptyEntries);
    var ok = lines.FirstOrDefault()?.StartsWith("RTSP/2.0 200", StringComparison.Ordinal) == true;
    var range = lines.FirstOrDefault(l => l.StartsWith("Range:", StringComparison.OrdinalIgnoreCase))
                    ?.Substring("Range:".Length).Trim();
    return new PauseOutcome(ok, range);
}
```

### Snippet C#: retomando reprodução com fallback para `npt=now-`

```csharp
static string BuildResumeAfterPause(string uri, int cseq, string sessionId, string? pauseRange, bool forceNow)
{
    var headers = new Dictionary<string, string> { ["Session"] = sessionId };
    headers["Range"] = forceNow ? "npt=now-" : (pauseRange ?? "npt=now-");
    return BuildRequest("PLAY", uri, cseq, headers);
}
```

---

## Checklist de conformidade prática

1. Requisições sempre com `CSeq` consistente.
2. Controle de sessão com `Session` após `SETUP`.
3. Implementar semântica de estado (`Init`, `Ready`, `Play`).
4. Validar `Range` e devolver erro adequado (`457`) quando necessário.
5. Em `PLAY`/`PAUSE`, tratar `Range` de resposta como fonte de verdade do ponto de mídia.
6. Encerrar recursos corretamente em `TEARDOWN`.
7. Suportar comportamento mínimo de timeout/liveness.

---

## Referências diretas

- RFC 7826 (documento principal): https://www.rfc-editor.org/info/rfc7826/
- HTML navegável (seções): https://datatracker.ietf.org/doc/html/rfc7826
- Seção 13.3 (SETUP): https://datatracker.ietf.org/doc/html/rfc7826#section-13.3
- Seção 13.4 (PLAY): https://datatracker.ietf.org/doc/html/rfc7826#section-13.4
- Seção 13.6 (PAUSE): https://datatracker.ietf.org/doc/html/rfc7826#section-13.6
- Seção 13.7 (TEARDOWN): https://datatracker.ietf.org/doc/html/rfc7826#section-13.7
