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

A implementação precisa suportar pelo menos os formatos necessários ao recurso negociado em `Accept-Ranges`.

### Snippet C#: parser simples de `Range: npt=start-stop`

```csharp
public readonly record struct NptRange(double? Start, double? Stop);

static NptRange ParseNpt(string rangeValue)
{
    // Espera "npt=10-30", "npt=10-", "npt=-30"
    var raw = rangeValue["npt=".Length..];
    var split = raw.Split('-', 2);
    double? start = split[0].Length == 0 ? null : double.Parse(split[0], System.Globalization.CultureInfo.InvariantCulture);
    double? stop  = split[1].Length == 0 ? null : double.Parse(split[1], System.Globalization.CultureInfo.InvariantCulture);
    return new NptRange(start, stop);
}
```

---

## 4) Modelo de mensagem RTSP (RFC §5-§9)

RTSP define:

- tipos de mensagem (requisição, resposta, mensagens específicas);
- cabeçalhos gerais e por método;
- corpo de mensagem e comprimento;
- linha de requisição e status line.

### Snippet C#: builder de requisição RTSP

```csharp
using System.Text;

static string BuildRequest(string method, string uri, int cseq, IDictionary<string, string>? headers = null, string? body = null)
{
    var sb = new StringBuilder();
    sb.AppendLine($"{method} {uri} RTSP/2.0");
    sb.AppendLine($"CSeq: {cseq}");

    if (headers is not null)
        foreach (var (k, v) in headers)
            sb.AppendLine($"{k}: {v}");

    if (!string.IsNullOrEmpty(body))
        sb.AppendLine($"Content-Length: {Encoding.UTF8.GetByteCount(body)}");

    sb.AppendLine();
    if (!string.IsNullOrEmpty(body)) sb.Append(body);
    return sb.ToString();
}
```

### Snippet C#: leitura de status line

```csharp
static int ParseStatusCode(string statusLine)
{
    var parts = statusLine.Split(' ', StringSplitOptions.RemoveEmptyEntries);
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

Regras práticas:

- se uma requisição de controle exigir sessão e vier sem `Session`, espere erro;
- em sessão agregada, use a URI de controle agregada para operações agregadas;
- `TEARDOWN` encerra ou reduz o escopo da sessão conforme URI e estado atual.

### 5.2 Como funciona o CSeq

`CSeq` é o número sequencial da requisição RTSP e é obrigatório para correlação de mensagens.

Boas práticas de implementação:

1. incrementa 1 por requisição enviada na conexão;
2. resposta deve retornar o mesmo `CSeq` da requisição correspondente;
3. em pipelining, `CSeq` evita ambiguidade quando várias respostas chegam em sequência.

### Snippet C#: gerador de CSeq thread-safe

```csharp
using System.Threading;

public sealed class CSeqGenerator
{
    private int _value;

    public CSeqGenerator(int startAt = 0) => _value = startAt;
    public int Next() => Interlocked.Increment(ref _value);
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

Pontos importantes da RFC para live:

- para conteúdo time-progressing, o ponto "agora" é dinâmico;
- após pausa em alguns cenários live, retomar pode exigir `Range: npt=now-`;
- em redes restritas por NAT/firewall, interleaved no canal RTSP pode simplificar conectividade;
- sincronização de mídia normalmente depende de informações como `RTP-Info` e relógio RTP/NPT.

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

Use `OPTIONS` e cabeçalhos de feature para descobrir extensões suportadas.

### 6.2 Pipelining

Permite enviar múltiplas requisições sem aguardar resposta individual imediata, respeitando correlação por `CSeq`.

### Snippet C#: correlacionando respostas por CSeq

```csharp
var pending = new Dictionary<int, TaskCompletionSource<string>>();

void Register(int cseq, TaskCompletionSource<string> tcs) => pending[cseq] = tcs;

void OnResponseReceived(int cseq, string rawResponse)
{
    if (pending.Remove(cseq, out var tcs))
        tcs.TrySetResult(rawResponse);
}
```

---

## 7) Métodos RTSP (RFC §13)

Este é o núcleo operacional do protocolo.

### 7.1 OPTIONS (§13.1)

**Para que serve:** descobrir métodos e capacidades suportadas pelo servidor/recurso.  
**Quando usar:** início da sessão e diagnóstico de interoperabilidade.  
**Esperado na resposta:** `Public` com lista de métodos (e possivelmente indicações de features).

```csharp
var options = BuildRequest("OPTIONS", "rtsp://example.com/media", 1);
```

### 7.2 DESCRIBE (§13.2)

**Para que serve:** obter descrição do recurso (tipicamente SDP).  
**Quando usar:** antes de `SETUP`, para saber trilhas e URIs de controle.  
**Cabeçalhos importantes:** `Accept` (ex.: `application/sdp`).  
**Erros comuns:** tipo de descrição não suportado.

```csharp
var describe = BuildRequest("DESCRIBE", "rtsp://example.com/media", 2,
    new Dictionary<string, string> { ["Accept"] = "application/sdp" });
```

### 7.3 SETUP (§13.3)

**Para que serve:** negociar transporte e criar/associar sessão RTSP.  
**Quando usar:** após `DESCRIBE`, para cada mídia/trilha necessária.  
**Cabeçalhos importantes:** `Transport`, `Accept-Ranges`.  
**Esperado na resposta:** `Session`, `Transport` selecionado, além de cabeçalhos de capacidades temporais.

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

```csharp
static bool IsPlayNotify(string method) =>
    method.Equals("PLAY_NOTIFY", StringComparison.OrdinalIgnoreCase);
```

### 7.6 PAUSE (§13.6)

**Para que serve:** interromper imediatamente e fixar pause point.  
**Quando usar:** sessão em `Play` (e idempotência prática em certos cenários já em `Ready`).  
**Esperado na resposta:** `Range` com ponto atual/trecho remanescente.  
**Atenção live:** para mídia time-progressing sem retenção, retomada pode exigir `npt=now-`.

```csharp
var pause = BuildRequest("PAUSE", "rtsp://example.com/media", 5,
    new Dictionary<string, string> { ["Session"] = "abcd1234" });
```

### 7.7 TEARDOWN (§13.7)

**Para que serve:** encerrar entrega e liberar recursos.  
**Quando usar:** final de reprodução, troca de conteúdo, limpeza de sessão.  
**Comportamento:** pode destruir sessão inteira ou remover mídia específica (dependendo de URI/estado e se sessão é agregada).

```csharp
var teardown = BuildRequest("TEARDOWN", "rtsp://example.com/media", 6,
    new Dictionary<string, string> { ["Session"] = "abcd1234" });
```

### 7.8 GET_PARAMETER (§13.8)

**Para que serve:** consultar parâmetros de sessão/recurso (estado, métricas etc.).  
**Quando usar:** monitoramento, keepalive em implementações que adotam esse padrão, telemetria de sessão.

```csharp
var getParameter = BuildRequest("GET_PARAMETER", "rtsp://example.com/media", 7,
    new Dictionary<string, string> { ["Session"] = "abcd1234" },
    body: "stream_state\r\njitter\r\npacket_loss\r\n");
```

### 7.9 SET_PARAMETER (§13.9)

**Para que serve:** atualizar parâmetros controláveis do recurso/sessão.  
**Quando usar:** ajustes em runtime (por exemplo volume, preferências de entrega, sinalizações de app).  
**Atenção:** validar conteúdo e permissões para evitar inconsistência de estado.

```csharp
var setParameter = BuildRequest("SET_PARAMETER", "rtsp://example.com/media", 8,
    new Dictionary<string, string>
    {
        ["Session"] = "abcd1234",
        ["Content-Type"] = "text/parameters"
    },
    body: "volume: 0.6\r\n");
```

### 7.10 REDIRECT (§13.10)

**Para que serve:** instruir o cliente a migrar para outro endpoint/URI.  
**Quando usar:** manutenção, balanceamento, reorganização de serviço.  
**Cabeçalho-chave:** `Location` com destino de redirecionamento.

```csharp
static Uri ParseRedirectTarget(IReadOnlyDictionary<string, string> headers) =>
    new(headers["Location"]);
```

---

## 8) Dados binários interleaved (RFC §14)

Quando RTP/RTCP é interleaved no mesmo canal RTSP, frames binários precisam de demultiplexação correta.

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

---

## 9) Proxies (RFC §15)

Proxies precisam:

- encaminhar mensagens preservando semântica;
- lidar com extensões/capabilities;
- multiplexar e demultiplexar corretamente.

### Snippet C#: exemplo simplificado de roteamento por URI

```csharp
static string PickUpstream(Uri requestUri) =>
    requestUri.Host.EndsWith(".interna.example", StringComparison.OrdinalIgnoreCase)
        ? "rtsp://upstream-a.internal"
        : "rtsp://upstream-b.internal";
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

Para interoperabilidade, o parser deve ser estrito no núcleo sintático e tolerante somente onde a RFC permitir.

### 13.2 Considerações de segurança

Boas práticas:

1. validar tamanho de cabeçalhos e corpo;
2. limitar recursos por sessão/conexão;
3. tratar redirecionamento com política explícita;
4. evitar parsing permissivo para campos críticos.

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
