# Steering Host Network - Código de Configuração e Implementação


> **Data**: 2026-03-26
> **Versão**: 1.1

---

Este documento contém todos os exemplos de código, configurações e implementações de referência para o projeto steering-host-network. Para discussões arquiteturais, motivadores e diagramas, consulte o documento [`TUNNING.md`](TUNNING.md).

---

## 1. Regras iptables para Steering por Conexão

### 1.1 Regras Principais (connmark)

```bash
# Tabela mangle - chain PREROUTING
# Regra 1: Se conexão já tem mark, mantém (não consulta ipset)
iptables -t mangle -A PREROUTING -m connmark ! --mark 0 -j CONNMARK --restore-mark

# Regra 2: Só consulta ipset para conexões novas (sem mark)
iptables -t mangle -A PREROUTING -m mark --mark 0 -m set --match-set steer_path_0_v4 dst -j MARK --set-mark 100
iptables -t mangle -A PREROUTING -m mark --mark 0 -m set --match-set steer_path_1_v4 dst -j MARK --set-mark 101

# Regra 3: Salva mark na conexão para pacotes futuros
iptables -t mangle -A PREROUTING -j CONNMARK --save-mark
```

### 1.2 Verificação de Estado conntrack

```bash
# conntrack -L | grep 203.0.113.10
tcp 6 431998 ESTABLISHED src=App_IP dst=203.0.113.10 sport=12345 dport=80
src=203.0.113.10 dst=200.20.1.3 sport=80 dport=12345 [ASSURED]
mark=0 use=1
[NAT] src=App_IP dst=203.0.113.10 → src=200.20.1.3 dst=203.0.113.10
```

---

## 2. Configuração de Edge A (Originador)

### 2.1 Configuração Completa do steerd

```yaml
# /etc/steerd/config.yaml
paths:
- id: "eth0-direct"
type: direct
interface: eth0
priority: 1
snat_pool:
- "200.10.1.1"
- "200.10.1.2"

- id: "wg0-via-edge-b"
type: tunnel
interface: wg0
peer_endpoint: "10.200.0.2"
priority: 2

hysteresis:
degrade_threshold: 50
recover_threshold: 80
min_time_in_path: 60s
```

---

## 3. Configuração de Edge B (Forwarder)

### 3.1 Configuração de Forwarding

```bash
# /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1

# iptables SNAT para tráfego do túnel
iptables -t nat -A POSTROUTING -s 10.200.0.0/30 -o eth0 -j SNAT \
--to-source 200.20.1.1-200.20.1.6 --random-fully

# Rota de retorno para o túnel
ip route add 10.200.0.0/30 dev wg0
```

### 3.2 Como o Retorno Funciona

O destino **NUNCA vê o IP de Edge A**. O fluxo é:

1. App em Edge A envia pacote com `src=App_IP, dst=203.0.113.10`
2. Pacote passa pelo túnel wg0 (encapsulado)
3. Edge B faz **SNAT**: `src=App_IP → src=200.20.1.3` (IP do Edge B)
4. Destino recebe e responde para `200.20.1.3` (IP do Edge B)
5. Edge B recebe, **conntrack** identifica a conexão e faz DNAT automático
6. Edge B envia resposta pelo túnel de volta para Edge A

**Por que funciona**: O SNAT no Edge B garante que o destino responda para o IP do Edge B, não para o IP de Edge A. O conntrack do Edge B mantém o estado da conexão e sabe que a resposta deve voltar pelo túnel.

---

## 4. Configuração de Histerese

### 4.1 Configuração YAML

```yaml
# configs/config.yaml.example - Histerese recomendada
hysteresis:
# Thresholds assimétricos previnem oscilação
degrade_threshold: 40 # Score < 40 para considerar migração
recover_threshold: 85 # Score > 85 para retornar ao primário

# Histerese baseada em tempo
min_time_in_path: 60s # Tempo mínimo antes de permitir migração
evaluation_window: 30s # Janela para cálculo do score
required_samples: 6 # Mínimo de amostras na janela

# Backoff exponencial para migrações repetidas
migration_backoff_base: 1m
migration_backoff_max: 30m
```

---

## 5. Dimensionamento de Mapas eBPF

### 5.1 Configuração de Mapas

```yaml
# configs/config.yaml.example
ebpf:
dest_stats_map_max_entries: 100000 # ~10MB com LRU
use_lru: true # Previne OOM

# Watch list para monitoramento sob demanda
watch_list:
max_entries: 10000 # Destinos ativos
healthy_removal_time: 300s # Remove saudáveis após 5min
```

---

## 6. Implementação: Monitoramento WireGuard

### 6.1 Código Go - WireGuard Health Monitor

```go
// internal/tunnel/wireguard.go
package tunnel

import (
"log/slog"
"net"
"os"
"strings"
"time"
)

// WireGuardMonitor monitora a saúde de interfaces WireGuard
type WireGuardMonitor struct {
log *slog.Logger
interfaces map[string]*WGInterface
handshakeTimeout time.Duration
checkInterval time.Duration
onUpdate func(iface string, healthy bool)
}

// WGInterface representa uma interface WireGuard
type WGInterface struct {
Name string
PublicKey string
ListenPort int
Peers map[string]*WGPeer
LastHandshake time.Time
RxBytes uint64
TxBytes uint64
}

// WGPeer representa um peer WireGuard
type WGPeer struct {
PublicKey string
Endpoint string
AllowedIPs []string
LastHandshake time.Time
TransferRx uint64
TransferTx uint64
PersistentKeepalive int
}

// PeerHealth representa a saúde de um peer
type PeerHealth struct {
LastHandshake time.Time
RxBytes uint64
TxBytes uint64
IsHealthy bool
Age time.Duration
}

// NewWireGuardMonitor cria um novo monitor
func NewWireGuardMonitor(log *slog.Logger) *WireGuardMonitor {
return &WireGuardMonitor{
log: log,
interfaces: make(map[string]*WGInterface),
handshakeTimeout: 3 * time.Minute,
checkInterval: 10 * time.Second,
}
}

// IsHealthy verifica se o túnel está funcional
func (w *WireGuardMonitor) IsHealthy(iface string) bool {
wg, ok := w.interfaces[iface]
if !ok {
return false
}

// Se último handshake foi há mais de 3 minutos, provavelmente down
if time.Since(wg.LastHandshake) > w.handshakeTimeout {
return false
}

return true
}

// GetPeerHealth retorna saúde de um peer específico
func (w *WireGuardMonitor) GetPeerHealth(iface, peerPubKey string) *PeerHealth {
wg, ok := w.interfaces[iface]
if !ok {
return nil
}

p, ok := wg.Peers[peerPubKey]
if !ok {
return nil
}

age := time.Since(p.LastHandshake)
return &PeerHealth{
LastHandshake: p.LastHandshake,
RxBytes: p.TransferRx,
TxBytes: p.TransferTx,
IsHealthy: age < w.handshakeTimeout,
Age: age,
}
}

// parseWGOutput faz parse da saída do comando `wg show`
func (w *WireGuardMonitor) parseWGOutput(output string) map[string]*WGInterface {
interfaces := make(map[string]*WGInterface)

var currentIface *WGInterface
var currentPeer *WGPeer

lines := strings.Split(output, "\n")
for _, line := range lines {
line = strings.TrimSpace(line)
if line == "" {
continue
}

// Nova interface
if !strings.HasPrefix(line, " ") && !strings.HasPrefix(line, "\t") {
parts := strings.Fields(line)
if len(parts) >= 1 {
currentIface = &WGInterface{
Name: parts[0],
Peers: make(map[string]*WGPeer),
}
interfaces[currentIface.Name] = currentIface
}
continue
}

if currentIface == nil {
continue
}

// Parse de campos
parts := strings.SplitN(line, ":", 2)
if len(parts) != 2 {
continue
}

key := strings.TrimSpace(parts[0])
value := strings.TrimSpace(parts[1])

switch key {
case "public key":
currentIface.PublicKey = value
case "listening port":
fmt.Sscanf(value, "%d", &currentIface.ListenPort)
case "peer":
currentPeer = &WGPeer{
PublicKey: value,
AllowedIPs: []string{},
}
currentIface.Peers[value] = currentPeer
case "endpoint":
if currentPeer != nil {
currentPeer.Endpoint = value
}
case "allowed ips":
if currentPeer != nil {
ips := strings.Split(value, ",")
for _, ip := range ips {
currentPeer.AllowedIPs = append(currentPeer.AllowedIPs, strings.TrimSpace(ip))
}
}
case "latest handshake":
if currentPeer != nil {
var seconds int64
fmt.Sscanf(value, "%d", &seconds)
currentPeer.LastHandshake = time.Unix(seconds, 0)
if currentPeer.LastHandshake.After(currentIface.LastHandshake) {
currentIface.LastHandshake = currentPeer.LastHandshake
}
}
case "transfer":
if currentPeer != nil {
// Parse "1.2 GiB received, 500 MiB sent"
fields := strings.Fields(value)
for i, f := range fields {
if f == "received" && i > 0 {
currentPeer.TransferRx = parseBytes(fields[i-1])
}
if f == "sent" && i > 0 {
currentPeer.TransferTx = parseBytes(fields[i-1])
}
}
}
}
}

return interfaces
}

// parseBytes converte string de bytes para uint64
func parseBytes(s string) uint64 {
var value float64
var unit string
fmt.Sscanf(s, "%f%s", &value, &unit)

switch unit {
case "KiB":
return uint64(value * 1024)
case "MiB":
return uint64(value * 1024 * 1024)
case "GiB":
return uint64(value * 1024 * 1024 * 1024)
default:
return uint64(value)
}
}
```

---

## 7. Implementação: Rate Limiting de Migrações

### 7.1 Código Go - Migration Rate Limiter

```go
// internal/decision/rate_limiter.go
package decision

import (
"sync"
"time"
)

// MigrationLimiter controla a taxa de migrações
type MigrationLimiter struct {
maxPerSecond int
maxPerMinute int
maxConcurrent int
cooldownPeriod time.Duration

mu sync.Mutex
secondCount int
minuteCount int
activeMigrations int
lastSecondReset time.Time
lastMinuteReset time.Time

// Tracking de migrações por destino
destLastMigration map[string]time.Time
destMigrationCount map[string]int
}

// MigrationLimiterConfig configura o rate limiter
type MigrationLimiterConfig struct {
MaxPerSecond int `yaml:"max_per_second"`
MaxPerMinute int `yaml:"max_per_minute"`
MaxConcurrent int `yaml:"max_concurrent"`
CooldownPeriod time.Duration `yaml:"cooldown_period"`
}

// DefaultMigrationLimiterConfig retorna configuração padrão
func DefaultMigrationLimiterConfig() MigrationLimiterConfig {
return MigrationLimiterConfig{
MaxPerSecond: 100, // Máximo 100 migrações/segundo
MaxPerMinute: 1000, // Máximo 1000 migrações/minuto
MaxConcurrent: 5000, // Máximo 5000 destinos migrados simultaneamente
CooldownPeriod: 60 * time.Second, // 60s entre migrações do mesmo destino
}
}

// NewMigrationLimiter cria um novo rate limiter
func NewMigrationLimiter(cfg MigrationLimiterConfig) *MigrationLimiter {
return &MigrationLimiter{
maxPerSecond: cfg.MaxPerSecond,
maxPerMinute: cfg.MaxPerMinute,
maxConcurrent: cfg.MaxConcurrent,
cooldownPeriod: cfg.CooldownPeriod,
destLastMigration: make(map[string]time.Time),
destMigrationCount: make(map[string]int),
lastSecondReset: time.Now(),
lastMinuteReset: time.Now(),
}
}

// AllowMigration verifica se uma migração é permitida
func (l *MigrationLimiter) AllowMigration(dest string) bool {
l.mu.Lock()
defer l.mu.Unlock()

now := time.Now()

// Reset contadores de tempo
if now.Sub(l.lastSecondReset) >= time.Second {
l.secondCount = 0
l.lastSecondReset = now
}
if now.Sub(l.lastMinuteReset) >= time.Minute {
l.minuteCount = 0
l.lastMinuteReset = now
}

// Verifica limites globais
if l.secondCount >= l.maxPerSecond {
return false
}
if l.minuteCount >= l.maxPerMinute {
return false
}
if l.activeMigrations >= l.maxConcurrent {
return false
}

// Verifica cooldown por destino
if lastMigration, ok := l.destLastMigration[dest]; ok {
if now.Sub(lastMigration) < l.cooldownPeriod {
return false
}
}

// Permite migração
l.secondCount++
l.minuteCount++
l.activeMigrations++
l.destLastMigration[dest] = now
l.destMigrationCount[dest]++

return true
}

// CompleteMigration marca uma migração como completa
func (l *MigrationLimiter) CompleteMigration(dest string) {
l.mu.Lock()
defer l.mu.Unlock()
l.activeMigrations--
}

// GetStats retorna estatísticas do rate limiter
func (l *MigrationLimiter) GetStats() MigrationStats {
l.mu.Lock()
defer l.mu.Unlock()

return MigrationStats{
SecondCount: l.secondCount,
MinuteCount: l.minuteCount,
ActiveMigrations: l.activeMigrations,
TotalDestinations: len(l.destMigrationCount),
}
}

// MigrationStats contém estatísticas de migração
type MigrationStats struct {
SecondCount int
MinuteCount int
ActiveMigrations int
TotalDestinations int
}
```

---

## 8. Implementação: Histerese Forte

### 8.1 Código Go - Strong Hysteresis

```go
// internal/decision/hysteresis.go
package decision

import (
"net/netip"
"sync"
"time"
)

// StrongHysteresis implementa histerese forte para decisões
type StrongHysteresis struct {
// Thresholds assimétricos
degradeThreshold float64 // Score < 40 para migrar
recoverThreshold float64 // Score > 85 para voltar

// Tempo mínimo em cada estado
minTimeInPath time.Duration // 60 segundos

// Janela de avaliação
evaluationWindow time.Duration // 30 segundos
requiredSamples int // Mínimo 6 samples

// Backoff exponencial
backoffBase time.Duration
backoffMax time.Duration

// Histórico por destino
mu sync.RWMutex
history map[netip.Addr]*DestHistory
}

// DestHistory mantém histórico de um destino
type DestHistory struct {
CurrentPath string
PathSince time.Time
LastMigration time.Time
MigrationCount int
RecentScores []ScoreSample
ConsecutiveLow int
ConsecutiveHigh int
}

// ScoreSample é uma amostra de score
type ScoreSample struct {
Score int
Timestamp time.Time
}

// StrongHysteresisConfig configura a histerese
type StrongHysteresisConfig struct {
DegradeThreshold float64 `yaml:"degrade_threshold"`
RecoverThreshold float64 `yaml:"recover_threshold"`
MinTimeInPath time.Duration `yaml:"min_time_in_path"`
EvaluationWindow time.Duration `yaml:"evaluation_window"`
RequiredSamples int `yaml:"required_samples"`
BackoffBase time.Duration `yaml:"backoff_base"`
BackoffMax time.Duration `yaml:"backoff_max"`
}

// DefaultStrongHysteresisConfig retorna configuração padrão
func DefaultStrongHysteresisConfig() StrongHysteresisConfig {
return StrongHysteresisConfig{
DegradeThreshold: 40.0, // Score < 40 = degradado
RecoverThreshold: 85.0, // Score > 85 = saudável
MinTimeInPath: 60 * time.Second, // Mínimo 60s no path
EvaluationWindow: 30 * time.Second, // Janela de 30s
RequiredSamples: 6, // Mínimo 6 amostras
BackoffBase: 1 * time.Minute, // Backoff inicial: 1min
BackoffMax: 30 * time.Minute, // Backoff máximo: 30min
}
}

// NewStrongHysteresis cria nova instância
func NewStrongHysteresis(cfg StrongHysteresisConfig) *StrongHysteresis {
return &StrongHysteresis{
degradeThreshold: cfg.DegradeThreshold,
recoverThreshold: cfg.RecoverThreshold,
minTimeInPath: cfg.MinTimeInPath,
evaluationWindow: cfg.EvaluationWindow,
requiredSamples: cfg.RequiredSamples,
backoffBase: cfg.BackoffBase,
backoffMax: cfg.BackoffMax,
history: make(map[netip.Addr]*DestHistory),
}
}

// ShouldMigrate determina se deve migrar
func (h *StrongHysteresis) ShouldMigrate(dest netip.Addr, currentScore float64) bool {
h.mu.Lock()
defer h.mu.Unlock()

hist, exists := h.history[dest]
if !exists {
hist = &DestHistory{
PathSince: time.Now(),
RecentScores: []ScoreSample{},
}
h.history[dest] = hist
}

now := time.Now()

// 1. Tempo mínimo no path atual
if now.Sub(hist.PathSince) < h.minTimeInPath {
return false
}

// 2. Adiciona score à janela
h.addScore(hist, currentScore, now)

// 3. Verifica amostras suficientes
if len(hist.RecentScores) < h.requiredSamples {
return false
}

// 4. Calcula score médio na janela
avgScore := h.calculateAverageScore(hist)

// 5. Verifica se score está consistentemente baixo
if avgScore > h.degradeThreshold {
return false
}

// 6. Verifica backoff exponencial
backoff := h.calculateBackoff(hist.MigrationCount)
if now.Sub(hist.LastMigration) < backoff {
return false
}

return true
}

// ShouldRecover determina se deve retornar ao path primário
func (h *StrongHysteresis) ShouldRecover(dest netip.Addr, currentScore float64) bool {
h.mu.RLock()
defer h.mu.RUnlock()

hist, exists := h.history[dest]
if !exists {
return false
}

// Score deve estar acima do threshold de recuperação
if currentScore < h.recoverThreshold {
return false
}

// Incrementa contador de scores altos consecutivos
hist.ConsecutiveHigh++
hist.ConsecutiveLow = 0

// Requer múltiplos scores altos consecutivos
return hist.ConsecutiveHigh >= h.requiredSamples
}

// RecordMigration registra que uma migração ocorreu
func (h *StrongHysteresis) RecordMigration(dest netip.Addr, newPath string) {
h.mu.Lock()
defer h.mu.Unlock()

hist, exists := h.history[dest]
if !exists {
hist = &DestHistory{
RecentScores: []ScoreSample{},
}
h.history[dest] = hist
}

hist.CurrentPath = newPath
hist.PathSince = time.Now()
hist.LastMigration = time.Now()
hist.MigrationCount++
hist.ConsecutiveHigh = 0
hist.ConsecutiveLow = 0
hist.RecentScores = []ScoreSample{}
}

// addScore adiciona score à janela deslizante
func (h *StrongHysteresis) addScore(hist *DestHistory, score float64, now time.Time) {
// Remove scores antigos
cutoff := now.Add(-h.evaluationWindow)
validScores := []ScoreSample{}
for _, s := range hist.RecentScores {
if s.Timestamp.After(cutoff) {
validScores = append(validScores, s)
}
}

// Adiciona novo score
validScores = append(validScores, ScoreSample{
Score: int(score),
Timestamp: now,
})

hist.RecentScores = validScores
}

// calculateAverageScore calcula score médio
func (h *StrongHysteresis) calculateAverageScore(hist *DestHistory) float64 {
if len(hist.RecentScores) == 0 {
return 100.0
}

sum := 0
for _, s := range hist.RecentScores {
sum += s.Score
}

return float64(sum) / float64(len(hist.RecentScores))
}

// calculateBackoff calcula tempo de backoff
func (h *StrongHysteresis) calculateBackoff(migrationCount int) time.Duration {
// Backoff exponencial: 2^n * base
multiplier := 1 << uint(migrationCount)
backoff := time.Duration(multiplier) * h.backoffBase

if backoff > h.backoffMax {
return h.backoffMax
}

return backoff
}
```

---

## 9. Métricas Prometheus

### 9.1 Estrutura de Métricas

```yaml
# Prometheus metrics structure
steering_dest_health_score:
type: gauge
labels: [dest, path]
description: Health score per destination

steering_path_health_score:
type: gauge
labels: [path_id, path_type]
description: Aggregated health per path

steering_migrations_total:
type: counter
labels: [from_path, to_path, reason]
description: Total migrations performed

steering_tunnel_handshake_age_seconds:
type: gauge
labels: [tunnel, peer]
description: WireGuard handshake age

steering_ebpf_map_entries:
type: gauge
labels: [map_name]
description: Entries in eBPF maps

steering_rate_limiter_rejected_total:
type: counter
labels: [reason]
description: Migrations rejected by rate limiter
```

### 9.2 Alertas Prometheus

```yaml
# alerting_rules.yaml (Prometheus)
groups:
- name: steering.critical
rules:
- alert: SteeringPathDown
expr: steering_path_health_score < 20
for: 1m
labels:
severity: critical
annotations:
summary: "Path {{ $labels.path_id }} is down"

- alert: SteeringHighMigrationRate
expr: rate(steering_migrations_total[1m]) > 100
for: 2m
labels:
severity: warning
annotations:
summary: "High migration rate detected"

- alert: SteeringTunnelStale
expr: time() - steering_tunnel_last_handshake > 180
for: 1m
labels:
severity: warning
annotations:
summary: "Tunnel {{ $labels.tunnel }} handshake stale"

- alert: SteeringRateLimiterActive
expr: rate(steering_rate_limiter_rejected_total[5m]) > 10
for: 5m
labels:
severity: info
annotations:
summary: "Rate limiter is actively rejecting migrations"

- alert: SteeringEBPFMapFull
expr: steering_ebpf_map_entries / steering_ebpf_map_max_entries > 0.9
for: 5m
labels:
severity: warning
annotations:
summary: "eBPF map {{ $labels.map_name }} is > 90% full"
```

---

## 10. Resumo de Arquivos de Implementação

| Arquivo | Descrição |
|---------|-----------|
| `internal/tunnel/wireguard.go` | Monitoramento de saúde WireGuard |
| `internal/decision/rate_limiter.go` | Rate limiting de migrações |
| `internal/decision/hysteresis.go` | Histerese forte para decisões |
| `configs/config.yaml.example` | Configuração principal do steerd |
| `configs/alerting_rules.yaml` | Alertas Prometheus |

Para discussões arquiteturais, motivadores e diagramas, consulte o documento [`TUNNING.md`](TUNNING.md).