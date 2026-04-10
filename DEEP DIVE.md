# Deep Dive: Treinamento Completo do Projeto steering-host-network

Este documento é um curso completo do zero ao intermediário sobre todas as tecnologias utilizadas neste projeto. **Não assume conhecimento prévio** - cada conceito é explicado desde o básico, com foco no **porquê** de cada escolha e os **ganhos práticos**.

---

## Índice

1. [Go (Golang) - Do Zero](#1-go-golang---do-zero)
2. [C eBPF - Explicado Detalhadamente](#2-c-ebpf---explicado-detalhadamente)
3. [Protobuf e gRPC](#3-protobuf-e-grpc)
4. [YAML](#4-yaml)
5. [Makefile](#5-makefile)
6. [Conceitos de Sistema](#6-conceitos-de-sistema)
7. [Integração no Projeto](#7-integração-no-projeto)

---

## 1. Go (Golang) - Do Zero

### 1.1 Por que Go? A Filosofia da Linguagem

Go foi criada no Google em 2007 por **Ken Thompson** (criador do Unix), **Rob Pike** (criador do UTF-8), e **Robert Griesemer**. Eles perceberam que:

- **C++ era muito complexo**: compilação lenta, muitos recursos, fácil de cometer erros
- **Java era verboso**: muito código para fazer coisas simples
- **Python/Ruby eram lentos**: interpretados, não adequados para sistemas de alta performance

**O que eles queriam?**
- Uma linguagem tão rápida quanto C
- Tão fácil de escrever quanto Python
- Com concorrência embutida (não uma biblioteca externa)
- Compilação rápida (segundos, não minutos)

**Por que isso importa para este projeto?**
- steerd precisa processar **milhões de pacotes por segundo**
- Precisa responder a eventos de saúde em **milissegundos**
- O código precisa ser **mantível** por uma equipe pequena

### 1.2 O que é `func`? Por que funções existem?

**Problema sem funções:**
```go
// Imagine um programa que precisa calcular média 3 vezes
func main() {
    // Primeira média
    soma1 := 10.0 + 20.0 + 30.0
    media1 := soma1 / 3.0
    
    // Segunda média
    soma2 := 15.0 + 25.0 + 35.0
    media2 := soma2 / 3.0
    
    // Terceira média
    soma3 := 5.0 + 10.0 + 15.0
    media3 := soma3 / 3.0
    
    fmt.Println(media1, media2, media3)
}
```

**Problemas:**
- Código repetido (3 vezes a mesma lógica!)
- Se precisar mudar a fórmula, tem que mudar em 3 lugares
- Fácil de errar (copiar e colar pode errar)

**Solução com funções:**
```go
// func = define uma função
// media = nome da função
// (a, b, c float64) = parâmetros (entradas)
// float64 = tipo de retorno (saída)
func media(a, b, c float64) float64 {
    return (a + b + c) / 3.0
}

func main() {
    m1 := media(10, 20, 30)  // Reutiliza a lógica
    m2 := media(15, 25, 35)
    m3 := media(5, 10, 15)
    
    fmt.Println(m1, m2, m3)
}
```

**Ganhos:**
- **Reutilização**: Escreve uma vez, usa quantas vezes quiser
- **Manutenção**: Mudou a fórmula? Muda em um lugar só
- **Legibilidade**: `media(10, 20, 30)` é mais claro que `(10+20+30)/3.0`
- **Testabilidade**: Pode testar a função isoladamente

**Analogia do professor**: Pense em uma função como uma **máquina**:
- Você coloca ingredientes (parâmetros) na entrada
- A máquina processa
- Você recebe o produto (retorno) na saída
- Pode usar a mesma máquina quantas vezes quiser

### 1.3 O que é `func main()`? Por que é especial?

**Todo programa precisa de um ponto de partida.** Quando você executa `./steerd`, o sistema operacional precisa saber por onde começar.

```go
package main  // Pacote especial = programa executável

import "fmt"

// main() = ponto de entrada
// O Go SEMPRE começa executando esta função
func main() {
    fmt.Println("Iniciando steerd...")
    
    // Inicializar componentes
    cfg := carregarConfig()
    engine := criarEngine(cfg)
    
    // Executar
    engine.Run()
}
```

**Por que `package main` é especial?**
- Outros pacotes (`package config`, `package engine`) são **bibliotecas**
- `package main` é **executável** - gera um binário que você pode rodar
- Apenas `package main` pode ter `func main()`

**O que acontece quando você compila?**
```bash
go build -o steerd ./cmd/steerd
```
1. Go encontra `package main`
2. Procura `func main()`
3. Gera código de inicialização do sistema
4. Cria binário executável

**Sem `main()`**: Go não sabe por onde começar, erro de compilação!

### 1.4 Por que usar Pacotes?

**Problema sem organização:**
```
projeto/
├── main.go  (5000 linhas!)
```
- Arquivo gigante, impossível de navegar
- Tudo no mesmo lugar, sem separação de responsabilidades
- Difícil de testar partes isoladas

**Solução com pacotes:**
```
projeto/
├── cmd/
│   └── steerd/
│       └── main.go          # Ponto de entrada
├── internal/
│   ├── config/
│   │   └── config.go        # Configuração
│   ├── decision/
│   │   └── engine.go        # Lógica de decisão
│   ├── health/
│   │   └── monitor.go      # Monitoramento
│   └── ebpf/
│       └── loader.go        # Carregamento eBPF
```

**Ganhos:**
- **Organização**: Cada pasta = uma responsabilidade
- **Encapsulamento**: `internal` só pode ser usado por este projeto
- **Reutilização**: `config` pode ser usado por `steerd` e `steerctl`
- **Testes isolados**: Pode testar `config` sem carregar eBPF

**Como importar pacotes:**
```go
package main

import (
    "github.com/aziontech/steering-host-network/internal/config"
    "github.com/aziontech/steering-host-network/internal/decision"
)

func main() {
    // Usa tipos de outros pacotes
    cfg := config.Load("config.yaml")
    engine := decision.NewEngine(cfg)
}
```

### 1.5 Por que Structs? O problema que elas resolvem

**Problema: variáveis soltas**
```go
func main() {
    // Dados de um prefixo
    var prefixo string = "192.168.1.0/24"
    var classID int = 0
    var score int = 85
    var degraded bool = false
    var lastSeen time.Time = time.Now()
    
    // Dados de outro prefixo
    var prefixo2 string = "10.0.0.0/8"
    var classID2 int = 1
    var score2 int = 70
    // ... mais variáveis soltas
    
    // Problema: como passar todos esses dados para uma função?
    processar(prefixo, classID, score, degraded, lastSeen)  // Muitos parâmetros!
}
```

**Problemas:**
- Muitas variáveis soltas, difícil de rastrear
- Passar para funções é trabalhoso (muitos parâmetros)
- Fácil de errar a ordem dos parâmetros

**Solução: agrupar com struct**
```go
// Definir um tipo que agrupa dados relacionados
type PrefixState struct {
    Prefixo      string
    ClassID      int
    Score        int
    Degraded     bool
    LastSeen     time.Time
}

func main() {
    // Criar instância (todos os dados juntos)
    estado1 := PrefixState{
        Prefixo:  "192.168.1.0/24",
        ClassID:  0,
        Score:    85,
        Degraded: false,
        LastSeen: time.Now(),
    }
    
    estado2 := PrefixState{
        Prefixo: "10.0.0.0/8",
        ClassID: 1,
        Score:   70,
    }
    
    // Passar para função: UM parâmetro só!
    processar(estado1)
    processar(estado2)
}

// Função recebe um valor, não 5
func processar(estado PrefixState) {
    fmt.Println(estado.Prefixo, estado.Score)
}
```

**Ganhos:**
- **Organização**: Dados relacionados ficam juntos
- **Menos parâmetros**: Uma struct em vez de 10 variáveis
- **Clareza**: `estado.Score` é mais claro que `score2`
- **Tipagem**: O compilador verifica se você está acessando campos que existem

**Analogia do professor**: Uma struct é como uma **ficha de cadastro**:
- A ficha tem campos predefinidos (Nome, Idade, Endereço)
- Você preenche uma ficha para cada pessoa
- Pode ter milhares de fichas, cada uma com seus valores
- Passar a ficha inteira é mais fácil que passar cada campo separadamente

### 1.6 Por que Métodos? A diferença entre função e método

**Função**: Código independente, não pertence a nenhum tipo específico.

**Método**: Função associada a um tipo (geralmente struct).

**Por que usar métodos?**

```go
// SEM métodos: funções soltas
type PrefixState struct {
    Prefixo string
    Score   int
}

// Função que opera sobre PrefixState
func calcularNovoScore(estado *PrefixState, delta int) {
    estado.Score += delta
}

func main() {
    estado := PrefixState{Score: 50}
    calcularNovoScore(&estado, 10)  // Precisa passar a struct
}
```

**COM métodos:**
```go
type PrefixState struct {
    Prefixo string
    Score   int
}

// Método: função com "receiver"
// (e *PrefixState) = este método pertence ao tipo PrefixState
func (e *PrefixState) IncrementarScore(delta int) {
    e.Score += delta
}

func main() {
    estado := PrefixState{Score: 50}
    estado.IncrementarScore(10)  // Mais natural!
}
```

**Ganhos:**
- **Sintaxe natural**: `estado.IncrementarScore(10)` vs `calcularNovoScore(&estado, 10)`
- **Encapsulamento**: O método "sabe" sobre a struct, não precisa passar tudo
- **Organização**: Métodos ficam junto com a struct no mesmo arquivo
- **Interface**: Métodos permitem criar interfaces (veremos adiante)

**Por que `(e *PrefixState)` e não `(e PrefixState)`?**

```go
// SEM ponteiro: modifica uma CÓPIA
func (e PrefixState) IncrementarCopia(delta int) {
    e.Score += delta  // Modifica a cópia, original não muda!
}

// COM ponteiro: modifica o ORIGINAL
func (e *PrefixState) IncrementarOriginal(delta int) {
    e.Score += delta  // Modifica o original
}

func main() {
    estado := PrefixState{Score: 50}
    
    estado.IncrementarCopia(10)
    fmt.Println(estado.Score)  // 50 (não mudou!)
    
    estado.IncrementarOriginal(10)
    fmt.Println(estado.Score)  // 60 (mudou!)
}
```

**Regra do professor**: 
- Se o método **lê** dados: pode usar valor `(e PrefixState)`
- Se o método **modifica** dados: use ponteiro `(e *PrefixState)`

### 1.7 Por que Ponteiros? O problema da cópia

**Em Go, quando você passa um valor para uma função, ele é COPIADO.**

```go
func dobrar(n int) {
    n = n * 2  // Modifica a CÓPIA
}

func main() {
    x := 10
    dobrar(x)
    fmt.Println(x)  // 10 (não mudou!)
}
```

**Por que isso acontece?**
- Go copia o valor para a função
- A função modifica a cópia
- O original permanece inalterado

**Solução: passar o endereço (ponteiro)**
```go
func dobrar(n *int) {
    *n = *n * 2  // Modifica o valor NO ENDEREÇO
}

func main() {
    x := 10
    dobrar(&x)  // Passa o endereço de x
    fmt.Println(x)  // 20 (mudou!)
}
```

**Analogia do professor**:
- **Valor** = uma caixa com um número
- **Ponteiro** = um papel com o endereço da caixa
- `&x` = anota o endereço da caixa
- `*p` = vai ao endereço e vê/modifica o que tem na caixa

**Por que isso é importante no projeto?**

```go
// internal/decision/engine.go
type Engine struct {
    prefixState map[string]*PrefixState  // Mapa grande!
}

// Se passássemos por valor, COPIARIA o mapa inteiro a cada chamada!
// Com ponteiro, usamos o mesmo mapa.
func (e *Engine) Decide(prefix string) Decision {
    // Modifica o mapa original
    e.prefixState[prefix] = &PrefixState{...}
}
```

**Ganhos dos ponteiros:**
- **Performance**: Não copia dados grandes
- **Modificação**: Permite que funções modifiquem o original
- **Compartilhamento**: Múltiplas partes do código acessam os mesmos dados

### 1.8 Por que Interfaces? O problema do acoplamento

**Problema: código acoplado a um tipo específico**

```go
type FileLogger struct {
    path string
}

func (f *FileLogger) Log(msg string) {
    // Escreve em arquivo
}

// Engine está acoplado a FileLogger
type Engine struct {
    logger *FileLogger  // Só aceita FileLogger!
}

func (e *Engine) Decide() {
    e.logger.Log("decisão tomada")
}
```

**Problemas:**
- Não pode trocar para `ConsoleLogger` sem mudar o código
- Não pode usar um mock para testes
- Engine depende de implementação específica

**Solução: interface define COMPORTAMENTO, não implementação**

```go
// Interface: define o que fazer, não como fazer
type Logger interface {
    Log(msg string)
}

// Implementação 1: arquivo
type FileLogger struct {
    path string
}

func (f *FileLogger) Log(msg string) {
    // Escreve em arquivo
}

// Implementação 2: console
type ConsoleLogger struct {}

func (c *ConsoleLogger) Log(msg string) {
    fmt.Println(msg)
}

// Engine aceita QUALQUER Logger
type Engine struct {
    logger Logger  // Interface, não tipo específico!
}

func (e *Engine) Decide() {
    e.logger.Log("decisão tomada")
}

func main() {
    // Usa FileLogger
    e1 := Engine{logger: &FileLogger{path: "/var/log/steerd.log"}}
    
    // Ou ConsoleLogger
    e2 := Engine{logger: &ConsoleLogger{}}
    
    // Ou Mock para testes
    e3 := Engine{logger: &MockLogger{}}
}
```

**Ganhos:**
- **Flexibilidade**: Trocar implementação sem mudar Engine
- **Testabilidade**: Usar mock em testes
- **Desacoplamento**: Engine não sabe nem precisa saber como Log funciona

**No projeto:**

```go
// internal/decision/engine.go
type OverrideManager interface {
    SetOverride(family int, prefix string, classID uint32, ttlNs uint64) error
    RemoveOverride(family int, prefix string) error
    IncrementClassGeneration(classID uint32) error
}
```

O `Engine` não sabe se está usando:
- O loader eBPF real
- Um mock para testes
- Uma implementação alternativa

Ele só sabe que pode chamar `SetOverride()`, `RemoveOverride()`, etc.

### 1.9 Por que Goroutines? O problema da execução sequencial

**Problema: programa sequencial**
```go
func main() {
    // Tudo acontece um após o outro
    carregarConfig()      // Espera 1 segundo
    conectarBanco()       // Espera 2 segundos
    iniciarAPI()          // Espera 1 segundo
    // Total: 4 segundos para iniciar!
}
```

**Problemas:**
- Tempo de inicialização = soma de todos os tempos
- Se uma operação demora, tudo espera
- Não aproveita múltiplos núcleos do CPU

**Solução: executar em paralelo**
```go
func main() {
    // go = execute em paralelo (goroutine)
    go carregarConfig()   // Executa em paralelo
    go conectarBanco()    // Executa em paralelo
    go iniciarAPI()       // Executa em paralelo
    // Total: ~2 segundos (o mais demorado)
    
    // Esperar todos terminarem
    time.Sleep(2 * time.Second)
}
```

**Analogia do professor**:
- **Sequencial**: Uma pessoa fazendo todas as tarefas, uma por uma
- **Paralelo**: Várias pessoas trabalhando ao mesmo tempo

**No projeto:**

```go
// cmd/steerd/main.go
func main() {
    // Múltiplas goroutines trabalhando em paralelo
    go fibSyncer.Run(ctx)      // Sincroniza rotas do Bird
    go healthMonitor.Run(ctx)  // Monitora saúde
    go apiServer.Run(ctx)      // Serve gRPC
    go prober.Run(ctx)         // Faz probes ativos
    
    // Todas executam simultaneamente!
}
```

**Ganhos:**
- **Performance**: Aproveita todos os núcleos do CPU
- **Responsividade**: Uma tarefa lenta não bloqueia as outras
- **Simplicidade**: Go gerencia as threads automaticamente

### 1.10 Por que Channels? O problema de comunicação entre goroutines

**Problema: goroutines precisam se comunicar**

```go
var contador int

func incrementar() {
    for i := 0; i < 1000; i++ {
        contador++  // DUAS goroutines acessando a mesma variável!
    }
}

func main() {
    go incrementar()
    go incrementar()
    time.Sleep(time.Second)
    fmt.Println(contador)  // Pode ser < 2000! (race condition)
}
```

**Problema**: Duas goroutines acessando a mesma variável = **race condition** (corrida de dados).

**Solução 1: Mutex (veremos depois)**

**Solução 2: Channels (comunicação)**

```go
// Channel: canal de comunicação
func main() {
    ch := make(chan int)
    
    // Goroutine produtora
    go func() {
        for i := 0; i < 3; i++ {
            ch <- i  // Envia valor para o canal
        }
        close(ch)  // Fecha o canal
    }()
    
    // Goroutine consumidora (main)
    for valor := range ch {  // Recebe até fechar
        fmt.Println(valor)
    }
}
```

**Analogia do professor**:
- Channel = uma **esteira de fábrica**
- Uma goroutine coloca itens na esteira (`ch <- valor`)
- Outra goroutine retira itens da esteira (`valor := <-ch`)
- A esteira sincroniza automaticamente

**No projeto:**

```go
// internal/health/monitor.go
type Monitor struct {
    eventCh chan PrefixEvent  // Canal de eventos do eBPF
}

func (m *Monitor) Run(ctx context.Context) {
    for {
        select {
        case evt := <-m.eventCh:  // Recebe evento do eBPF
            m.RecordEvent(evt)     // Processa
        }
    }
}
```

**Ganhos:**
- **Sincronização automática**: Channel bloqueia até ter dados
- **Sem race conditions**: Dados fluem em uma direção
- **Código limpo**: Não precisa de locks manuais

### 1.11 Por que Select? O problema de múltiplos canais

**Problema: esperar em múltiplos canais**

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    // Como esperar em AMBOS os canais?
    // Sem select, precisaria de goroutines separadas
}
```

**Solução: select**

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    go func() { ch1 <- 1 }()
    go func() { ch2 <- 2 }()
    
    for i := 0; i < 2; i++ {
        select {
        case v := <-ch1:
            fmt.Println("Recebido de ch1:", v)
        case v := <-ch2:
            fmt.Println("Recebido de ch2:", v)
        }
    }
}
```

**No projeto:**

```go
// internal/health/monitor.go
func (m *Monitor) Run(ctx context.Context) error {
    ticker := time.NewTicker(5 * time.Second)
    
    for {
        select {
        case <-ctx.Done():        // Cancelamento
            return ctx.Err()
        case <-ticker.C:           // Poll periódico
            m.collect()
        case evt := <-m.eventCh:   // Evento do eBPF
            m.RecordEvent(evt)
        }
    }
}
```

**Ganhos:**
- **Multiplexação**: Espera em múltiplos canais simultaneamente
- **Cancelamento**: `ctx.Done()` permite sair gracefully
- **Timer**: `ticker.C` dispara em intervalos regulares

### 1.12 Por que Context? O problema do cancelamento

**Problema: como cancelar goroutines?**

```go
func main() {
    go func() {
        for {
            // Como parar esta goroutine?
            fazerTrabalho()
        }
    }()
    
    // Usuário aperta Ctrl+C
    // A goroutine continua rodando!
}
```

**Solução: Context**

```go
func main() {
    // Context com cancelamento
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        for {
            select {
            case <-ctx.Done():  // Cancelado!
                return
            default:
                fazerTrabalho()
            }
        }
    }()
    
    // Quando quiser parar
    cancel()  // Sinaliza cancelamento
}
```

**No projeto:**

```go
// cmd/steerd/main.go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Passa context para todos os componentes
    go fibSyncer.Run(ctx)
    go healthMonitor.Run(ctx)
    
    // SIGINT (Ctrl+C) chama cancel()
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh
    cancel()  // Cancela todos os contextos
}
```

**Ganhos:**
- **Cancelamento coordenado**: Um sinal para todos
- **Timeout**: `context.WithTimeout` para operações com prazo
- **Graceful shutdown**: Termina trabalho em andamento

### 1.13 Por que Error Handling assim? O problema de exceções

**Em linguagens como Python/Java:**
```python
# Python: exceções
try:
    resultado = dividir(10, 0)
except Exception as e:
    print("Erro:", e)
```

**Em Go:**
```go
// Go: erros são valores
resultado, err := dividir(10, 0)
if err != nil {
    fmt.Println("Erro:", err)
    return  // Retorna cedo
}
```

**Por que Go escolheu assim?**

1. **Explícito**: Você vê que uma função pode falhar
2. **Sem surpresas**: Exceções não "pulam" de lugar nenhum
3. **Controlado**: Você decide o que fazer com o erro

**Padrão comum:**
```go
func processar() error {
    // Cada operação pode falhar
    cfg, err := config.Load("config.yaml")
    if err != nil {
        return fmt.Errorf("carregar config: %w", err)  // Wrap
    }
    
    engine, err := decision.NewEngine(cfg)
    if err != nil {
        return fmt.Errorf("criar engine: %w", err)
    }
    
    return nil  // Sucesso
}
```

**Ganhos:**
- **Clareza**: Fica óbvio onde pode falhar
- **Rastreamento**: `%w` preserva o erro original
- **Controle**: Você decide se propaga ou trata

### 1.14 Por que sync.RWMutex? O problema de race conditions

**Problema: múltiplas goroutines acessando os mesmos dados**

```go
type Contador struct {
    valor int
}

func (c *Contador) Incrementar() {
    c.valor++  // NÃO é atômico!
    // Na verdade: lê valor, soma 1, escreve valor
    // Se duas goroutines fazem ao mesmo tempo = race condition
}
```

**Solução: Mutex (exclusão mútua)**

```go
type Contador struct {
    mu    sync.Mutex  // Tranca
    valor int
}

func (c *Contador) Incrementar() {
    c.mu.Lock()         // Tranca (outras goroutines esperam)
    defer c.mu.Unlock() // Destranca ao sair
    c.valor++
}
```

**RWMutex: leitura vs escrita**

```go
type Engine struct {
    mu          sync.RWMutex  // Mutex de leitura/escrita
    prefixState map[string]*PrefixState
}

// Escrita: exclusiva (só uma goroutine)
func (e *Engine) Decide(prefix string) Decision {
    e.mu.Lock()
    defer e.mu.Unlock()
    // ... modifica prefixState
}

// Leitura: múltiplas goroutines podem ler
func (e *Engine) GetState(prefix string) *PrefixState {
    e.mu.RLock()
    defer e.mu.RUnlock()
    return e.prefixState[prefix]
}
```

**Analogia do professor**:
- `Lock()` = tranca a sala, só você entra
- `RLock()` = permite várias pessoas lerem, mas ninguém escreve

**Ganhos:**
- **Thread-safe**: Sem race conditions
- **Performance**: RLock permite leitura concorrente
- **Simplicidade**: Go gerencia o lock automaticamente

---

## 2. C eBPF - Explicado Detalhadamente

### 2.1 Por que eBPF? O problema da extensibilidade do kernel

**O kernel Linux é o coração do sistema operacional:**
- Gerencia memória, CPU, rede, disco
- Precisa de permissões especiais (root)
- Um bug pode crashar a máquina inteira

**Problema tradicional:**
- Você quer uma funcionalidade nova no kernel
- Opção 1: Modificar o código-fonte do kernel → recompilar → reiniciar
- Opção 2: Carregar um módulo de kernel → pode crashar o sistema

**Por que isso é ruim?**
- **Lento**: Compilar kernel demora horas
- **Arriscado**: Módulos podem crashar o kernel
- **Inflexível**: Precisa reiniciar para mudar

**Solução: eBPF**

eBPF permite **carregar programas pequenos e seguros** no kernel, em tempo de execução, sem reiniciar.

```
┌─────────────────────────────────────────────────────────────┐
│                    ANTES (Módulo de Kernel)                  │
│                                                              │
│  1. Escrever código C do módulo                              │
│  2. Compilar módulo                                          │
│  3. Carregar módulo (insmod)                                 │
│  4. Se tiver bug → KERNEL PANIC → reiniciar máquina         │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    DEPOIS (eBPF)                             │
│                                                              │
│  1. Escrever programa eBPF em C                              │
│  2. Compilar com clang                                       │
│  3. Carregar programa                                        │
│  4. Verifier analisa → se inseguro, rejeita                 │
│  5. Se tiver bug → programa falha, kernel continua          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Ganhos:**
- **Segurança**: Verifier garante que o programa não crasha o kernel
- **Sem reinício**: Carrega e descarrega em tempo de execução
- **Performance**: Executa no kernel, sem transições para userspace

### 2.2 Por que Mapas eBPF? O problema de comunicação

**Programas eBPF rodam no kernel.** Mas precisamos:
- Passar configuração do userspace para o kernel
- Receber métricas do kernel para o userspace

**Solução: Mapas eBPF**

Mapas são **estruturas de dados compartilhadas** entre kernel e userspace.

```
┌─────────────────────────────────────────────────────────────┐
│                      USERSPACE (Go)                          │
│                                                              │
│   steerd lê/escreve mapas via syscall bpf()                  │
│                           │                                  │
│                           ▼                                  │
├─────────────────────────────────────────────────────────────┤
│                      KERNEL SPACE                            │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              eBPF Maps (memória compartilhada)        │  │
│   │                                                       │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐              │  │
│   │   │tcp_stats│  │ override │  │ fib_v4  │              │  │
│   │   └─────────┘  └─────────┘  └─────────┘              │  │
│   │        ▲              ▲             ▲                 │  │
│   │        │              │             │                 │  │
│   │   ┌────┴────┐    ┌────┴────┐   ┌────┴────┐            │  │
│   │   │tcp_mon  │    │steering │   │ fib     │            │  │
│   │   │         │    │_connect │   │ syncer  │            │  │
│   │   └─────────┘    └─────────┘   └─────────┘            │  │
│   │   (programas eBPF)                                   │  │
│   └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Analogia do professor**:
- Mapas são como **quadros de mensagens** na parede
- O programa eBPF escreve métricas no quadro
- O programa Go lê as métricas do quadro
- O programa Go escreve configurações no quadro
- O programa eBPF lê as configurações do quadro

### 2.3 Por que cada tipo de mapa?

#### BPF_MAP_TYPE_HASH

**O que é**: Dicionário chave-valor, como um map do Go.

**Por que usar**: Quando você precisa buscar dados por uma chave arbitrária.

**Exemplo**: `tcp_watch_v4` - lista de IPs sendo monitorados.

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 20000);
    __type(key, __u32);              // Chave: IP de destino
    __type(value, struct watch_entry); // Valor: configuração
} tcp_watch_v4 SEC(".maps");
```

**Ganhos:**
- Lookup O(1) - muito rápido
- Chave pode ser qualquer tipo
- Adicionar/remover dinamicamente

#### BPF_MAP_TYPE_LRU_HASH

**O que é**: Hash map que remove entradas antigas automaticamente.

**Por que usar**: Quando você tem mais dados que memória disponível.

**Problema que resolve**:
- Tabela de roteamento BGP tem ~900.000 rotas
- Não podemos guardar estatísticas para todas
- Precisamos manter apenas as "quentes" (acessadas recentemente)

**Exemplo**: `tcp_stats_v4` - estatísticas TCP por IP.

```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, 65536);
    __type(key, __u32);
    __type(value, struct tcp_dest_stats);
} tcp_stats_v4 SEC(".maps");
```

**Como funciona LRU (Least Recently Used)**:
1. Mapa tem 65536 entradas
2. Quando enche, remove a entrada mais antiga não acessada
3. IPs acessados recentemente ficam
4. IPs não acessados são removidos

**Ganhos:**
- Memória limitada (não cresce infinitamente)
- Mantém dados relevantes automaticamente
- Sem necessidade de limpeza manual

#### BPF_MAP_TYPE_LPM_TRIE

**O que é**: Mapa especializado para **Longest Prefix Match**.

**Por que usar**: Para roteamento IP!

**O problema do roteamento**:
- Você tem múltiplas rotas: `/0`, `/8`, `/16`, `/24`, `/32`
- Quando busca um IP, quer a rota **mais específica**
- Exemplo: `10.1.1.5` → `/24` é mais específico que `/8`

**Como funciona LPM**:

```
Mapa override_v4:
  10.0.0.0/8     → class 1
  10.1.0.0/16    → class 2
  10.1.1.0/24    → class 3

Busca por 10.1.1.5:
  1. Tenta /32 (10.1.1.5) - não existe
  2. Tenta /31 - não existe
  3. Tenta /30 - não existe
  ...
  7. Tenta /24 (10.1.1.0/24) - EXISTE! Retorna class 3

Busca por 10.2.0.5:
  1. Tenta /32 - não existe
  ...
  8. Tenta /16 - não existe
  9. Tenta /8 (10.0.0.0/8) - EXISTE! Retorna class 1
```

**Estrutura da chave LPM**:

```c
struct lpm_key_v4 {
    __u32 prefixlen;  // Comprimento do prefixo (ex: 24 para /24)
    __u32 addr;       // Endereço IP em network byte order
};
```

**Ganhos:**
- Busca automática do mais específico
- Perfeito para roteamento
- Eficiente (O(prefixlen))

#### BPF_MAP_TYPE_ARRAY

**O que é**: Array indexado por inteiro (0, 1, 2, ...).

**Por que usar**: Quando você tem número fixo de elementos e índices conhecidos.

**Exemplo**: `bind_classes_v4` - pools de IP por classe.

```c
#define MAX_CLASSES 8

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, MAX_CLASSES);  // 8 classes
    __type(key, __u32);                // Índice: 0-7
    __type(value, struct bind_class_v4); // Pool de IPs
} bind_classes_v4 SEC(".maps");
```

**Por que Array e não Hash?**
- Índices são sempre 0-7 (conhecidos)
- Lookup é O(1) e mais rápido que Hash
- Memória pré-alocada

**Ganhos:**
- Mais rápido que Hash
- Índices contíguos
- Memória pré-alocada

#### BPF_MAP_TYPE_RINGBUF

**O que é**: Buffer circular para enviar eventos do kernel para userspace.

**Por que usar**: Comunicação assíncrona eficiente.

**O problema**:
- Programa eBPF detecta uma retransmissão
- Precisa notificar o userspace
- Não pode bloquear (está no kernel!)

**Solução: Ring Buffer**

```
┌────────────────────────────────────────────────────────────┐
│                    Ring Buffer (256KB)                      │
│                                                             │
│   [evt1][evt2][evt3][evt4][    ][    ][    ][evt5][evt6]   │
│                    ▲                              ▲         │
│                 consumer                       producer    │
│                 (Go lê)                     (eBPF escreve) │
└────────────────────────────────────────────────────────────┘

1. eBPF reserva espaço no ring buffer
2. eBPF escreve evento
3. eBPF "submit" (evento disponível)
4. Go lê evento (sem syscall por evento!)
5. Go processa evento
```

**Exemplo**:

```c
// eBPF: enviar evento
struct steer_event *evt;
evt = bpf_ringbuf_reserve(&tcp_events, sizeof(*evt), 0);
if (evt) {
    evt->type = EVENT_RETRANSMIT;
    evt->dest_ip = dest;
    bpf_ringbuf_submit(evt, 0);  // Envia
}
```

```go
// Go: receber evento
for {
    record, _ := ringBuf.Read()
    var evt steer_event
    binary.Read(bytes.NewReader(record.RawSample), binary.LittleEndian, &evt)
    // Processa evento
}
```

**Ganhos:**
- **Zero-copy**: Go lê diretamente da memória do kernel
- **Sem syscalls por evento**: Muito eficiente
- **Multi-producer**: Múltiplos programas eBPF podem escrever
- **Sem bloqueio**: eBPF não espera

### 2.4 Por que CO-RE? O problema da portabilidade

**Problema**: Estruturas do kernel mudam entre versões.

```c
// Kernel 5.10
struct sock {
    int field1;
    int field2;
    int interesting_field;  // Offset 8
};

// Kernel 5.15
struct sock {
    int field1;
    int new_field;          // Novo campo!
    int field2;
    int interesting_field;  // Agora offset 12!
};
```

**Sem CO-RE**:
```c
// Acesso direto - quebra em kernels diferentes!
int value = sk->__sk_common.skc_daddr;
```

**Com CO-RE**:
```c
// Acesso via BPF_CORE_READ - funciona em qualquer kernel!
int value = BPF_CORE_READ(sk, __sk_common.skc_daddr);
```

**Como funciona CO-RE (Compile Once, Run Everywhere)**:

1. **Compilação**: clang gera código com "relocations" (marcadores)
2. **BTF**: Kernel expõe seus tipos via BTF (BPF Type Format)
3. **Carga**: libbpf resolve relocations usando BTF do kernel atual
4. **Execução**: Programa usa offsets corretos

**BTF (BPF Type Format)**: É como um "header file" do kernel em execução.

```bash
# Ver BTF do seu kernel
bpftool btf dump file /sys/kernel/btf/vmlinux format c | head -50
```

**Ganhos:**
- **Portabilidade**: Um binário funciona em múltiplos kernels
- **Sem headers**: Não precisa de headers do kernel específico
- **Manutenção**: Não recompilar para cada kernel

### 2.5 Por que cgroups? O problema do escopo

**Problema**: Como aplicar eBPF apenas a certos processos?

**Solução**: cgroups (Control Groups)

cgroups permitem agrupar processos e aplicar limites/isolamento.

```
┌─────────────────────────────────────────────────────────────┐
│                        cgroup v2                             │
│                                                              │
│   /sys/fs/cgroup/                                            │
│       ├── init.scope/                                        │
│       ├── user.slice/                                        │
│       └── steering.slice/        ← NOSSO CGROUP             │
│           ├── cgroup.procs       ← PIDs dos processos       │
│           └── (eBPF anexado aqui)                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Como funciona:**

1. Criamos cgroup: `mkdir /sys/fs/cgroup/steering.slice`
2. Movemos processos: `echo $PID > .../cgroup.procs`
3. Anexamos eBPF: `bpftool cgroup attach ... connect4`

**Resultado**: Apenas processos no cgroup passam pelo eBPF!

**Ganhos:**
- **Escopo controlado**: Só processos específicos
- **Isolamento**: Não afeta outros processos
- **Flexibilidade**: Pode mover processos dinamicamente

### 2.6 O que é o Verifier? Por que é importante?

**Verifier** é o "guardião" que analisa o programa eBPF antes de carregá-lo.

**Por que precisamos dele?**
- Programas eBPF rodam no kernel
- Um bug pode crashar a máquina inteira
- Precisamos garantir que o programa é seguro

**O que o verifier verifica:**

1. **Acesso à memória**: Não pode acessar memória aleatória
```c
// ERRO: acesso não verificado
int *ptr = ...;
int val = *ptr;  // ptr pode ser NULL!

// CORRETO: verificar antes
if (ptr) {
    int val = *ptr;  // OK
}
```

2. **Bounds de array**: Não pode acessar fora dos limites
```c
// ERRO: índice não verificado
int arr[10];
int idx = ...;
int val = arr[idx];  // idx pode ser > 9!

// CORRETO: verificar limites
if (idx < 10) {
    int val = arr[idx];  // OK
}
```

3. **Loops**: Devem ter limite conhecido
```c
// ERRO: loop potencialmente infinito
while (condition) { ... }

// CORRETO: loop com limite
#pragma unroll
for (int i = 0; i < 100; i++) { ... }
```

4. **Stack**: Máximo 512 bytes

**Ganhos:**
- **Segurança**: Programas verificados não crasham o kernel
- **Previsibilidade**: Tempo de execução limitado
- **Confiabilidade**: Kernel continua funcionando

### 2.7 Por que Struct Layout é crítico?

**Problema**: Go e C devem concordar sobre o layout da struct.

**Por que isso é um problema?**
- Compiladores podem adicionar "padding" (espaços vazios) para alinhamento
- Go e C podem adicionar padding diferente
- Se discordarem, dados são lidos incorretamente!

**Exemplo do problema:**

```c
// C struct
struct example {
    __u8  a;     // 1 byte
    // 3 bytes de padding (C adiciona para alinhar!)
    __u32 b;     // 4 bytes
};
// Total C: 8 bytes
```

```go
// Go struct (sem padding explícito)
type Example struct {
    A uint8   // 1 byte
    B uint32 // 4 bytes
}
// Go pode adicionar padding diferente!
```

**Solução: Padding explícito**

```c
// C struct com padding explícito
struct example {
    __u8  a;        // 1 byte
    __u8  _pad[3];  // 3 bytes explícitos
    __u32 b;        // 4 bytes
};
// Total: 8 bytes (previsível)
```

```go
// Go struct correspondente
type Example struct {
    A    uint8
    _pad [3]byte  // Mesmo padding!
    B    uint32
}
// Total: 8 bytes (igual ao C)
```

**Ganhos:**
- **Correção**: Dados lidos corretamente
- **Portabilidade**: Funciona em todas as arquiteturas
- **Debugabilidade**: Erros de padding são difíceis de achar

---

## 3. Protobuf e gRPC

### 3.1 Por que Protobuf? O problema de JSON

**JSON é legível, mas ineficiente:**

```json
// JSON (58 bytes)
{"name": "João", "age": 30, "active": true}
```

**Protobuf é binário e compacto:**

```protobuf
// Protobuf (9 bytes)
// name: "João" (4 bytes + length)
// age: 30 (1 byte varint)
// active: true (1 byte)
```

**Comparação:**

| Aspecto | JSON | Protobuf |
|---------|------|----------|
| Tamanho | Grande (texto) | Pequeno (binário) |
| Parsing | Lento | Rápido |
| Schema | Opcional | Obrigatório |
| Tipos | Dinâmicos | Estáticos |

**Ganhos:**
- **Menor tamanho**: Menos banda de rede
- **Parsing rápido**: CPU economizada
- **Tipos garantidos**: Schema define estrutura

### 3.2 Por que gRPC? O problema de APIs REST

**REST é simples, mas tem limitações:**
- Múltiplas chamadas HTTP = overhead
- JSON parsing em cada chamada
- Sem streaming bidirecional

**gRPC resolve:**
- **HTTP/2**: Múltiplas chamadas em uma conexão
- **Protobuf**: Serialização eficiente
- **Streaming**: Comunicação bidirecional

**Ganhos:**
- **Performance**: Menos overhead
- **Tipagem**: Schema define API
- **Streaming**: Comunicação em tempo real

---

## 4. YAML

### 4.1 Por que YAML? O problema de configuração

**Opções para configuração:**
- JSON: Verboso, sem comentários
- TOML: Menos conhecido
- YAML: Legível, suporta comentários

**YAML é escolhido porque:**
- **Legível**: Fácil de entender
- **Comentários**: Pode documentar
- **Estruturado**: Suporta listas, maps, aninhamento

---

## 5. Makefile

### 5.1 Por que Makefile? O problema de comandos manuais

**Sem Makefile:**
```bash
# Lembrar todos os comandos...
go build -o bin/steerd ./cmd/steerd
go build -o bin/steerctl ./cmd/steerctl
go test ./...
protoc --go_out=. --go-grpc_out=. internal/api/steering.proto
# ... e mais 20 comandos
```

**Com Makefile:**
```bash
make build    # Compila tudo
make test     # Roda testes
make proto    # Gera protobuf
```

**Ganhos:**
- **Simplicidade**: Um comando vs muitos
- **Documentação**: Makefile documenta como buildar
- **Automação**: Dependências automáticas

---

## 6. Conceitos de Sistema

### 6.1 Netlink

**Por que**: Comunicação com o kernel para rotas.

**Ganho**: Receber eventos de rota do Bird em tempo real.

### 6.2 LPM Trie

**Por que**: Roteamento IP precisa de longest prefix match.

**Ganho**: Busca automática do prefixo mais específico.

### 6.3 Ring Buffer

**Por que**: Eventos assíncronos do kernel para userspace.

**Ganho**: Zero-copy, sem syscalls por evento.

---

## 7. Integração no Projeto

### 7.1 Fluxo de Dados Completo

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USERSPACE                                   │
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │  Bird    │───▶│ FIB      │───▶│ Decision │───▶│ Override │       │
│  │ (BGP)    │    │ Syncer   │    │ Engine   │    │ Manager │       │
│  └──────────┘    └──────────┘    └──────────┘    └────┬─────┘       │
│                                                        │             │
│  ┌──────────┐    ┌──────────┐                          │             │
│  │ Health   │◀───│ eBPF     │◀─────────────────────────┘             │
│  │ Monitor  │    │ Loader   │                                        │
│  └────┬─────┘    └────┬─────┘                                        │
│       │               │                                               │
└───────┼───────────────┼───────────────────────────────────────────────┘
        │               │
        ▼               ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         KERNEL SPACE                                   │
│                                                                        │
│  steering_connect.c → override_v4 → bind_classes → bpf_bind           │
│         ↑                                                              │
│  tcp_monitor.c → tcp_stats → ringbuf → Health Monitor                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Por que essa arquitetura?

1. **Bird → FIB Syncer**: Bird conhece rotas, FIB Syncer traduz para eBPF
2. **FIB Syncer → Decision Engine**: Rotas viram decisões de failover
3. **Decision Engine → Override Manager**: Decisões viram overrides no eBPF
4. **eBPF → Health Monitor**: Métricas do kernel alimentam decisões
5. **Health Monitor → Decision Engine**: Métricas viram novas decisões

**Ganho**: Ciclo fechado, automático, sem intervenção manual.

---

## 8. Resumo: Por que cada escolha?

| Tecnologia | Problema que resolve | Ganho principal |
|------------|----------------------|-----------------|
| **Go** | Performance + simplicidade | Código rápido e mantível |
| **func/struct** | Organização de código | Reutilização e clareza |
| **Goroutines** | Execução paralela | Usa todos os núcleos |
| **Channels** | Comunicação entre goroutines | Sem race conditions |
| **Context** | Cancelamento coordenado | Graceful shutdown |
| **Interfaces** | Acoplamento | Flexibilidade e testes |
| **eBPF** | Extensibilidade do kernel | Sem reiniciar, seguro |
| **Mapas eBPF** | Comunicação kernel-userspace | Dados compartilhados |
| **LPM Trie** | Roteamento IP | Prefixo mais específico |
| **Ring Buffer** | Eventos assíncronos | Zero-copy |
| **CO-RE** | Portabilidade | Um binário, múltiplos kernels |
| **Protobuf** | Serialização eficiente | Menor tamanho, mais rápido |
| **gRPC** | API eficiente | HTTP/2, streaming |
| **YAML** | Configuração legível | Fácil de entender |
| **Makefile** | Automação de build | Um comando para tudo |
