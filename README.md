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

## Referências

- [CheckMK Docker — documentação oficial](https://docs.checkmk.com/latest/en/introduction_docker.html)
- [CheckMK Docker Hub](https://hub.docker.com/r/checkmk/check-mk-community)
- [OMD — Open Monitoring Distribution](https://omdistro.org/)
