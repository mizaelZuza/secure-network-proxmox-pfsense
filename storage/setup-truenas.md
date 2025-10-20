Perfeito 🔥 — aqui está um **guia técnico completo em formato `.md`** com o passo a passo para configurar o **TrueNAS SCALE como storage NFS para um cluster Kubernetes**.

---

```markdown
# 🧩 Configuração do TrueNAS SCALE como Storage NFS para Kubernetes

## 📘 Visão Geral
Este guia descreve o processo completo para configurar o **TrueNAS SCALE** como **servidor NFS** para uso como **storage persistente** em um **cluster Kubernetes**.

---

## 🧱 Pré-requisitos

- ✅ TrueNAS SCALE instalado e acessível (Consulte guia `configuracoes-adicionais.md`)
- ✅ Cluster Kubernetes funcional
- ✅ Conectividade entre o TrueNAS e os nós do cluster
- ✅ Acesso ao pfSense (ou outro firewall) entre as redes

Exemplo de redes utilizadas:
```

Kubernetes: 192.168.3.0/24
TrueNAS:    192.168.1.0/24

````

---

## ⚙️ 1. Criar o Pool de Armazenamento no TrueNAS

1. Acesse **Storage → Pools → Add**
2. Escolha **Manual Configuration**
3. Selecione os discos (ex: 3 discos)
4. Defina o **Layout** como:
   - `RAIDZ1` → equilíbrio entre desempenho e tolerância a falhas
5. Clique em **Create**

> 💡 RAIDZ1 permite perder 1 disco sem perda de dados. Ideal para ambiente de testes e clusters pequenos.

---

## 📁 2. Criar Dataset para o Kubernetes

1. Vá em **Storage → Pools → (seu pool) → Add Dataset**
2. Nome: `k8s_storage`
3. Em “Share Type”: selecione **Generic**
4. Configure permissões:
   - **Owner (User):** `nobody`
   - **Owner (Group):** `nogroup`
   - Permissões: `Read/Write/Execute` para todos (modo estudo)
5. Clique em **Save**

---

## 🌐 3. Configurar o Serviço NFS

1. Vá em **System Settings → Services → NFS → Edit (⚙️)**
2. Configure conforme abaixo:

| Campo | Valor |
|--------|-------|
| **Enabled Protocols** | NFSv3, NFSv4 |
| **mountd(8) bind port** | `20048` |
| **rpc.statd(8) bind port** | `32765` |
| **rpc.lockd(8) bind port** | `32803` |
| **Allow non-root mount** | ✅ (habilitado) |

3. Clique em **Save**
4. Ative o serviço NFS:
   - **System Settings → Services → NFS → Start**
   - Marque **Start Automatically**

---

## 🧾 4. Criar o NFS Share

1. Vá em **Shares → Unix Shares (NFS) → Add**
2. **Path:** `/mnt/<pool>/k8s_storage`
3. **Mapall User:** `nobody`
4. **Mapall Group:** `nogroup`
5. **Authorized Networks:** `192.168.3.0/24`
6. Clique em **Save**  
7. Ative o compartilhamento NFS (toggle verde)

---

## 🧱 5. Ajustar Firewall (pfSense)

Caso o TrueNAS e o cluster estejam em redes diferentes, crie regras no **pfSense**:

### 🔓 Libere as seguintes portas (TCP/UDP):

| Porta | Serviço | Descrição |
|--------|----------|-----------|
| 111 | rpcbind | Serviço base do NFS |
| 2049 | nfs | Principal serviço NFS |
| 20048 | mountd | Montagem de volumes |
| 32765 | statd | Controle de bloqueios |
| 32803 | lockd | Gerenciador de locks |

### 🔒 Regra recomendada (em Firewall → Rules):
- **Interface:** LAN ou a VLAN correspondente  
- **Source:** `192.168.3.0/24`  
- **Destination:** `192.168.1.114` (IP do TrueNAS)  
- **Ports:** conforme tabela acima  
- **Action:** Pass  

---

## 🧪 6. Testar Conectividade

No **nó do Kubernetes**:

```bash
nc -zv 192.168.1.114 2049
nc -zv 192.168.1.114 111
nc -zv 192.168.1.114 20048
nc -zv 192.168.1.114 32765
nc -zv 192.168.1.114 32803
````

Se todos responderem com `succeeded`, o acesso está OK.

Valide os exports:

```bash
showmount -e 192.168.1.114
```

Saída esperada:

```
Export list for 192.168.1.114:
/mnt/k8s_storage 192.168.3.0/24
```

---

## 📦 7. Configurar o StorageClass no Kubernetes

Crie o arquivo `nfs-storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
```

Crie o PersistentVolume (PV):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/k8s_storage
    server: 192.168.1.114
  persistentVolumeReclaimPolicy: Retain
```

Crie o PersistentVolumeClaim (PVC):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ""
  volumeName: nfs-pv
```

Aplicar tudo:

```bash
kubectl apply -f nfs-storageclass.yaml
kubectl apply -f nfs-pv.yaml
kubectl apply -f nfs-pvc.yaml
```

---

## ✅ 8. Validar no Kubernetes

Verifique se o volume foi montado:

```bash
kubectl get pv
kubectl get pvc
```

Os status devem aparecer como:

```
STATUS: Bound
```

---

## 🚀 Conclusão

Com essas etapas, o **TrueNAS SCALE** estará servindo como **storage NFS estável e persistente** para o cluster Kubernetes.
Este método é ideal para **laboratórios, estudos e ambientes de desenvolvimento**.

---
