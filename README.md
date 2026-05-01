# CheckMK Community — Deploy Kubernetes (Homelab)

Deploy do **CheckMK Community Edition 2.5.0** em Kubernetes com armazenamento NFS.

O YAML original foi criado pelo meu colega **Norman Sa Silva** para um cluster RKE1 corporativo e adaptado por mim para o meu homelab (k8s v1.35 / Debian 13 / Calico).

---

## Por que PersistentVolume manual em vez da StorageClass do homelab?

O homelab usa a StorageClass `nfs-homelab` (provisionador `nfs.csi.k8s.io`) com **NFSv4.1**, que é o padrão recomendado para K8s moderno. No entanto, o **CheckMK/OMD tem um problema conhecido de file locking sobre NFSv4**: o processo interno usa `flock()` extensivamente para locking entre os serviços do site (Nagios, Apache, rrdcached, etc.), e o kernel Linux, ao operar sobre NFSv4 sem `local_lock`, envia essas chamadas ao servidor NFS — o que resulta em `OSError: [Errno 9] Bad file descriptor` e o container entra em crash loop.

A documentação oficial do CheckMK menciona esse comportamento para ambientes containerizados com volumes de rede:

> *"If you use an NFS volume, make sure that file locking works correctly. Some NFS configurations do not support POSIX file locking, which can cause issues with the OMD site."*
> — [CheckMK Docker documentation](https://docs.checkmk.com/latest/en/introduction_docker.html)

### Solução: PV manual com NFSv3 + nolock + local_lock=all

Usando um `PersistentVolume` manual é possível especificar as mount options diretamente no objeto PV — algo que o provisionador CSI dinâmico aplica a nível de StorageClass (afetando todos os workloads que a usam). Com um PV dedicado para o CheckMK, as opções ficam isoladas:

```yaml
mountOptions:
  - vers=3          # NFSv3: sem o problema de locking do NFSv4 com flock()
  - nolock          # Não envia lock requests ao servidor NFS
  - local_lock=all  # Todo o locking (flock + posix) é resolvido localmente no node
  - hard
  - noatime
  - rsize=1048576
  - wsize=1048576
```

**Por que não alterar a StorageClass do homelab?**  
A StorageClass `nfs-homelab` é usada por outros workloads (n8n, postgres, cotacoes, etc.) que funcionam corretamente com NFSv4.1. Mudar as mount options globalmente poderia impactar esses serviços — e `local_lock=all` não é recomendado para workloads multi-pod que dependem de locking distribuído real. O PV manual mantém o CheckMK isolado com a configuração que ele precisa.

**Por que NFSv3 funciona?**  
No NFSv3, a semântica de `flock()` com `nolock` faz o kernel resolver o locking inteiramente no lado do cliente (node K8s), sem consultar o servidor. Como o CheckMK roda em um único pod (`strategy: Recreate`), não há risco de conflito entre instâncias — o locking local é suficiente e correto.

---

## Estrutura do deploy

```
checkmk.deploy.yaml
├── Namespace
├── PersistentVolume       # PV manual NFSv3 dedicado
├── PersistentVolumeClaim  # Bind direto ao PV (storageClassName: "")
├── ConfigMap              # siteid + timezone
├── Secret                 # CMK_PASSWORD
├── Deployment             # 1 réplica, strategy Recreate
│   ├── emptyDir (tmpfs)   # /opt/omd/sites/cmk/tmp — volátil, em memória
│   └── NFS PVC            # /omd/sites — persistente
├── Service                # ClusterIP porta 5000
└── Ingress                # nginx + cert-manager (Let's Encrypt)
```

### Por que emptyDir para o tmp?

O diretório `/opt/omd/sites/cmk/tmp` contém arquivos de estado voláteis (sockets, PIDs, checkresults, status.dat). A documentação oficial recomenda usar tmpfs para esse diretório em ambientes containerizados:

> *"Mount a tmpfs to /opt/omd/sites/\<site\>/tmp to improve performance and avoid unnecessary writes to the persistent volume."*
> — [CheckMK Docker documentation](https://docs.checkmk.com/latest/en/introduction_docker.html)

Sem o emptyDir, esses arquivos iriam para o NFS, aumentando a latência de I/O e gerando escrita desnecessária a cada check.

---

## Pré-requisitos

- Kubernetes 1.28+
- Ingress NGINX
- cert-manager com ClusterIssuer `prod-letsencrypt-cloudflare`
- Servidor NFS acessível (ajuste `server` e `path` no PV)
- Diretório NFS criado previamente no servidor:
  ```bash
  mkdir -p /mnt/nfs-data/k8s/checkmk
  chmod 777 /mnt/nfs-data/k8s/checkmk
  ```

---

## Como aplicar

```bash
# Ajuste o host no Ingress e o path/server do NFS no PV antes de aplicar
kubectl apply -f checkmk.deploy.yaml

# Acompanhe o primeiro start (pode levar 2-3 min para o omd create)
kubectl logs -n checkmk -f deployment/checkmk

# Aguarde o pod ficar Ready (startupProbe tem 10 min de tolerância)
kubectl get pod -n checkmk -w
```

Acesso: `https://checkmk-k8s.staypuff.info/cmk/` — usuário `cmkadmin`, senha conforme o Secret.

---

## Variáveis de ambiente importantes

| Variável | Valor | Observação |
|---|---|---|
| `CMK_SITE_ID` | `cmk` | Nome do site OMD |
| `CMK_PASSWORD` | base64 | Senha do `cmkadmin` |
| `TZ` | `Brazil/East` | **Não usar** `America/Sao_Paulo` — bug no entrypoint.sh que usa `_` como separador no `sed`, quebrando o restart |

---

## Troubleshooting: agentes legados não coletam (pull mode)

Esta seção documenta um problema real encontrado durante o deploy e a investigação completa que levou à solução.

### Sintoma

O pod subia normalmente, a UI do CheckMK estava acessível, mas hosts configurados apareciam como `DOWN` e sem dados. O comando de diagnóstico retornava:

```
ERROR [agent]: Communication failed: timed out
```

### Contexto: como o pull mode funciona

O CheckMK tem dois modos de comunicação com agentes:

| Modo | Direção | Porta |
|---|---|---|
| **Pull (legado)** | CheckMK server → host monitorado | 6556 (no host) |
| **Push (Agent Controller)** | host monitorado → CheckMK server | 8000 (no server) |

No pull mode, **o servidor inicia a conexão**. O agente escuta na porta 6556 do host monitorado e responde com os dados. Nenhuma porta precisa ser exposta no pod para pull mode.

---

### Investigação passo a passo

#### Etapa 1 — Verificar se o pod consegue pingar os hosts

```bash
sudo kubectl exec -it -n checkmk <pod> -- bash
ping -c2 192.168.3.31
```

**Resultado:** ping funcionando. Isso confirmou que a rede básica estava ok, mas era necessário `NET_RAW` capability para o CheckMK enviar ICMP — adicionado no container:

```yaml
securityContext:
  capabilities:
    add: ["NET_RAW"]
```

#### Etapa 2 — Verificar conectividade TCP na porta do agente

```bash
# De dentro do pod
nc -zw3 192.168.3.11 6556; echo EXIT:$?
```

**Resultado:** `EXIT:0` — a conexão TCP era estabelecida. O problema não era firewall nem rota básica.

#### Etapa 3 — Verificar se o agente manda dados

```bash
# De dentro do pod
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556' | head -3
```

**Resultado:** nenhum dado recebido. O TCP conecta mas o agente não responde.

#### Etapa 4 — Verificar o estado do agente no host monitorado

```bash
# No PVE1 (192.168.3.11)
cmk-agent-ctl status --json
```

**Resultado:**
```json
{
  "version": "2.4.0p25",
  "agent_socket_operational": true,
  "ip_allowlist": [],
  "allow_legacy_pull": true,
  "connections": []
}
```

O agente estava com `allow_legacy_pull: true` e `ip_allowlist: []` (sem restrição de IP). Não era `only_from` bloqueando.

#### Etapa 5 — Confirmar que o agente funciona de outros hosts

```bash
# Do k8s-master (192.168.3.30)
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556' | head -3
```

**Resultado:**
```
<<<check_mk>>>
Version: 2.4.0p25
AgentOS: linux
```

Do master funcionava. Do pod, não. Mesma porta, mesmo host — comportamento diferente.

#### Etapa 6 — Captura de tráfego (tcpdump) — a prova definitiva

Iniciamos um `tcpdump` no PVE1 enquanto o pod tentava conectar:

```bash
# No PVE1
tcpdump -n -i nic0 port 6556

# Simultaneamente, no pod
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556'
```

**Evidência capturada:**

```
# Conexão do SERVIDOR ANTIGO (192.168.3.5) — funciona normalmente:
09:02:24 IP 192.168.3.5.42496  > 192.168.3.31.6556:  Flags [S]   # SYN
09:02:24 IP 192.168.3.31.6556  > 192.168.3.5.42496:  Flags [S.]  # SYN-ACK
09:02:24 IP 192.168.3.5.42496  > 192.168.3.31.6556:  Flags [.]   # ACK
09:02:26 IP 192.168.3.31.6556  > 192.168.3.5.42496:  Flags [.]   # dados fluindo (183KB)

# Conexão do POD (192.168.194.91) — PROBLEMA:
09:02:25 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (1ª tentativa)
09:02:26 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmissão)
09:02:26 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmissão)
09:02:27 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmissão)
09:02:28 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmissão)
09:02:30 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmissão)
```

**Diagnóstico:** o PVE1 nunca recebeu o ACK do pod. O handshake TCP nunca se completou — o SYN-ACK era enviado mas o pod não o recebia (retransmissões infinitas).

---

### Causa raiz: sobreposição de CIDR + Calico sem MASQUERADE

O cluster usa Calico com pod CIDR `192.168.0.0/16`.

A rede física do homelab também usa `192.168.3.0/24` — que está **dentro** do range `192.168.0.0/16`.

Quando o pod (IP `192.168.194.91`) enviava um pacote para o PVE1 (`192.168.3.11`):

```
Pod 192.168.194.91 ──SYN──▶ PVE1 192.168.3.11
```

O Calico interpretava o destino `192.168.3.x` como **pertencente ao pod CIDR** e **não aplicava MASQUERADE (SNAT)**. O pacote chegava ao PVE1 com source IP real do pod (`192.168.194.91`).

O PVE1 tentava responder:

```
PVE1 192.168.3.11 ──SYN-ACK──▶ 192.168.194.91 ???
```

Mas o PVE1 não tinha rota para `192.168.194.0/24` — essa sub-rede só existe dentro do cluster Kubernetes. O SYN-ACK ia para o gateway padrão e se perdia. O handshake nunca se completava.

Isso explicava por que `nc -z` às vezes retornava 0 (o SYN chegava ao PVE1), mas nenhum dado era recebido (o handshake não se completava).

> **Por que do master/worker funcionava?**
> Conexões de `192.168.3.30` ou `192.168.3.31` são IPs físicos normais. O PVE1 sabe rotear a resposta de volta para `192.168.3.x` pela rede local — o handshake completa normalmente.

---

### Solução: `hostNetwork: true`

Com `hostNetwork: true`, o pod não tem um IP de pod (`192.168.194.x`). Ele usa diretamente o IP do nó em que está rodando (`192.168.3.31`).

```
Pod usando 192.168.3.31 ──SYN──▶ PVE1 192.168.3.11
PVE1 192.168.3.11 ──SYN-ACK──▶ 192.168.3.31  ✓ (rota conhecida)
```

O campo `dnsPolicy: ClusterFirstWithHostNet` é necessário para que o pod continue resolvendo nomes de serviços internos do cluster (ex: `checkmk.svc.cluster.local`) mesmo usando a rede do host.

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

#### Verificação após o fix

```bash
# IP do pod agora é o IP do nó
sudo kubectl get pods -n checkmk -o wide
# NAME                       READY   STATUS    IP             NODE
# checkmk-547c56b7bd-m27xd   1/1     Running   192.168.3.31   k8s-worker1

# Coleta funcionando
su - cmk -c "cmk -vd PVE1"
# <<<check_mk>>>
# Version: 2.4.0p25
# AgentOS: linux
# Hostname: pve1
# ...
```

#### Alternativas consideradas e descartadas

| Alternativa | Por que não |
|---|---|
| Rotas estáticas em cada host monitorado | Frágil: o pod pode mover de nó entre restarts |
| Mudar o pod CIDR do cluster | Requer rebuild completo do cluster |
| BGP entre Calico e roteador físico | Complexidade desnecessária para homelab |

---

## Referências

- [CheckMK Docker — documentação oficial](https://docs.checkmk.com/latest/en/introduction_docker.html)
- [CheckMK Docker Hub](https://hub.docker.com/r/checkmk/check-mk-community)
- [OMD — Open Monitoring Distribution](https://omdistro.org/)
- [Calico — pod CIDR e MASQUERADE](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Kubernetes hostNetwork](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces)
