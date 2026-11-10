# APS 3 - Telemetria e Alertas dos AGVs

Aluno: **Carlos Tokarski Borges Neto**
Modelagem de Software - Prof. Sidney L. da Silva
Projeto: Sistema de Logística de Entregas Autônomas (AGVs)

## Funcionalidade escolhida

Minha parte é a **telemetria dos AGVs**, ou seja, como o sistema recebe os dados que os veículos mandam e o que faz com eles.

Funciona assim:
1. o AGV manda posição, bateria e velocidade a cada 1 segundo pelo broker MQTT;
2. o Serviço de Telemetria recebe, confere se a mensagem está certa e, ao mesmo tempo, salva no banco e verifica a bateria;
3. se a bateria estiver abaixo de 15%, sai um alerta e o AGV fica como `BATERIA_BAIXA` (ele para de receber missões);
4. se estiver tudo normal, só atualiza a posição no mapa do operador;
5. se o AGV ficar mais de 5 segundos sem mandar nada, ele é marcado como `OFFLINE`.

Essa parte é importante porque o despacho e o replanejamento de rotas dependem de saber onde cada AGV está e quanta bateria ele tem.

## Operações de cada participante

| Participante | Operações |
|---|---|
| Broker MQTT | `publicarTelemetria(TelemetriaDTO)` |
| Serviço de Telemetria | `onTelemetria(TelemetriaDTO)`, `verificarSemSinal(agvId)` |
| Banco de Dados | `salvarLeitura(TelemetriaDTO)`, `atualizarStatusAgv(agvId, status)` |
| RabbitMQ | `publicarAlerta(AlertaDTO)`, `publicarPosicao(PosicaoDTO)` |
| API de Orquestração | `onAlerta(AlertaDTO)`, `onPosicao(PosicaoDTO)` |
| Aplicação Web | `mostrarAlerta(AlertaDTO)`, `atualizarMapa(PosicaoDTO)` |

## Artefato 1 - Diagrama de Sequência

Setas: `->>` síncrona, `-)` assíncrona (MQTT, RabbitMQ e WebSocket), `-->>` retorno.

```mermaid
sequenceDiagram
    participant AGV
    participant Broker as Broker MQTT
    participant Tel as Serviço de Telemetria
    participant BD as Banco de Dados
    participant Fila as RabbitMQ
    participant API as API de Orquestração
    participant Web as Aplicação Web

    loop a cada 1 segundo
        AGV-)Broker: publicarTelemetria(TelemetriaDTO)
        Broker-)Tel: onTelemetria(TelemetriaDTO)
        Tel->>BD: salvarLeitura(TelemetriaDTO)
        BD-->>Tel: ok

        alt bateria < 15
            Tel-)Fila: publicarAlerta(AlertaDTO)
            Fila-)API: onAlerta(AlertaDTO)
            API->>BD: atualizarStatusAgv(agvId, BATERIA_BAIXA)
            BD-->>API: AgvDTO
            API-)Web: mostrarAlerta(AlertaDTO)
        else bateria >= 15
            Tel-)Fila: publicarPosicao(PosicaoDTO)
            Fila-)API: onPosicao(PosicaoDTO)
            API-)Web: atualizarMapa(PosicaoDTO)
        end
    end

    opt AGV ficou mais de 5s sem mandar nada
        Tel->>Tel: verificarSemSinal(agvId)
        Tel-)Fila: publicarAlerta(AlertaDTO tipo OFFLINE)
        Fila-)API: onAlerta(AlertaDTO)
        API->>BD: atualizarStatusAgv(agvId, OFFLINE)
        BD-->>API: AgvDTO
        API-)Web: mostrarAlerta(AlertaDTO)
    end
```

## Artefato 2 - Diagrama de Atividades

O fork divide em duas tarefas que rodam em paralelo (salvar a leitura e verificar a bateria) e o join espera as duas terminarem.

```mermaid
flowchart TB
    subgraph R1[AGV]
        I((início))
        A1[Ler sensores: posição, bateria e velocidade]
        A2[Publicar telemetria no broker]
    end

    subgraph R2[Serviço de Telemetria]
        B1[Receber mensagem]
        D1{Mensagem válida?}
        B2[Descartar mensagem]
        FORK[ ]:::barra
        B3[Salvar leitura no banco]
        B4[Verificar limites de bateria]
        JOIN[ ]:::barra
        D2{Bateria baixa?}
        B5[Publicar alerta]
        B6[Publicar posição]
    end

    subgraph R3[API de Orquestração]
        C1[Mudar status do AGV para BATERIA_BAIXA]
        C2[Atualizar mapa no painel]
    end

    subgraph R4[Operador]
        E1[Ver alerta e decidir o que fazer]
        E2[Acompanhar AGV no mapa]
        F((fim))
    end

    I --> A1 --> A2 --> B1 --> D1
    D1 -->|"[dados incompletos ou timestamp antigo]"| B2 --> F
    D1 -->|"[dados completos e timestamp novo]"| FORK
    FORK --> B3 --> JOIN
    FORK --> B4 --> JOIN
    JOIN --> D2
    D2 -->|"[bateria < 15]"| B5 --> C1 --> E1 --> F
    D2 -->|"[bateria >= 15]"| B6 --> C2 --> E2 --> F

    classDef barra fill:#000,stroke:#000,color:#000
```

## Artefato 3 - Diagrama de Estados do AGV

```mermaid
stateDiagram-v2
    [*] --> Ligando

    state "Ligando" as Ligando
    Ligando : entry / conectarNoBroker()

    Ligando --> Ativo : autoTesteOk
    Ligando --> Manutencao : autoTesteFalhou / abrirChamado()

    state Ativo {
        [*] --> Disponivel
        Disponivel --> EmMissao : recebeuMissao
        EmMissao --> Disponivel : terminouMissao
    }

    Ativo --> BateriaBaixa : leituraRecebida [bateria < 15] / enviarAlerta()

    state "BateriaBaixa" as BateriaBaixa
    BateriaBaixa : entry / bloquearNovasMissoes()
    BateriaBaixa : exit / liberarNovasMissoes()

    BateriaBaixa --> Carregando : chegouNaEstacao
    Carregando --> Ativo : cargaCompleta [bateria >= 90]

    Ativo --> Offline : semSinal [tempo > 5s] / avisarOperador()
    Offline --> Ativo : sinalVoltou / sincronizar()
    Offline --> Manutencao : semSinal [tempo > 10min]

    Manutencao --> Ligando : consertado
    Manutencao --> Desativado : semConserto
    Desativado --> [*]
```

Para o AGV não ficar trocando de estado o tempo todo perto dos 15%, ele só volta a ficar `Ativo` depois de carregar até 90%. E leituras com timestamp mais antigo que a última são descartadas, para uma mensagem atrasada não "desfazer" um alerta.

## Artefato 4 - C4 Nível 3

```mermaid
flowchart TB
    agv["AGVs<br/>[Sistema externo]"]
    broker["Broker MQTT<br/>[Container: Mosquitto]"]

    subgraph tel[Serviço de Telemetria - Container: Node.js]
        consumer["TelemetriaConsumer<br/>[Componente: MQTT.js]"]
        validador["Validador<br/>[Componente]"]
        analisador["AnalisadorBateria<br/>[Componente]"]
        repo["LeituraRepository<br/>[Componente: pg]"]
        watchdog["VerificadorSemSinal<br/>[Componente: setInterval]"]
        pub["AlertaPublisher<br/>[Componente: amqplib]"]
    end

    bd[("Banco de Dados<br/>[Container: PostgreSQL]")]
    fila["RabbitMQ<br/>[Container: fila de eventos]"]
    api["API de Orquestração<br/>[Container: Node.js + NestJS]"]
    web["Aplicação Web<br/>[Container: React]"]

    agv -->|"MQTT (QoS 0)"| broker
    broker -->|MQTT| consumer
    consumer --> validador
    validador --> repo
    validador --> analisador
    analisador --> pub
    watchdog --> pub
    repo -->|SQL| bd
    pub -->|AMQP| fila
    fila -->|AMQP| api
    api -->|WebSocket| web

    classDef container fill:#1168bd,color:#fff
    classDef comp fill:#85bbf0,color:#000
    classDef ext fill:#999,color:#fff
    class broker,bd,fila,api,web container
    class consumer,validador,analisador,repo,watchdog,pub comp
    class agv ext
```

## Artefato 5 - ADR

[ADR 0001 - Usar MQTT para receber a telemetria dos AGVs](docs/architecture/adr-0001.md)

## Como os diagramas se conectam

- O **fork/join** das atividades (salvar leitura + verificar bateria) aparece na sequência como `salvarLeitura()` seguido do `alt` da bateria.
- O alerta de bateria (`publicarAlerta` → `onAlerta` → `atualizarStatusAgv(BATERIA_BAIXA)`) é a transição `Ativo → BateriaBaixa` da máquina de estados.
- O `opt` de 5 segundos sem sinal é a transição `Ativo → Offline`.
