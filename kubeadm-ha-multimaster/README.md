# Configurando um cluster em Alta Disponibilidade usando kubeadm

Siga esta documentação para configurar um cluster Kubernetes altamente disponível usando __Ubuntu 20.04 LTS__.

Esta documentação orienta você na configuração de um cluster com 3 nodes master, 3 nodes worker e 2 nodes de balanceador de carga usando Haproxy juntamente com o keepalived.

## Pré-requisitos

|Função|Quantidade|SO|RAM|CPU|
|----|----|----|----|----|
|Load Balancer|1|Ubuntu 20.04|1G|1|
|Master|3|Ubuntu 20.04|2G|2|
|Worker|3|Ubuntu 20.04|2G|2|

### Portas liberadas em sua rede

|MASTER|PORTAS|
|----|----|
|kube-apiserver|6443 TCP|
|etcd server API|2379-2380 TCP|
|Kubelet API|10250 TCP|
|kube-scheduler|10251 TCP|
|kube-controller-manager|10252 TCP|
|Kubelet API Read-only|10255 TCP|

|WORKER|PORTAS|
|----|----|
|Kubelet API|10250 TCP|
|Kubelet API Read-only|10255 TCP|
|NodePort Services|30000-32767 TCP|

## Configurando o Load Balancer

Na máquina de Load Balancer siga os passos:

### Instalação Haproxy

``` bash
apt update && apt install -y haproxy
```

### Configuração Haproxy

Adicione o conteudo abaixo em **/etc/haproxy/haproxy.cfg**
Ajuste os ips de acordo com o seu ambiente.

```bash
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
  log /dev/log local0
  log /dev/log local1 notice
  daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
  mode                    http
  log                     global
  option                  httplog
  option                  dontlognull
  option                  http-server-close
  option                  forwardfor       except 127.0.0.0/8
  option                  redispatch
  retries                 1
  timeout http-request    10s
  timeout queue           20s
  timeout connect         5s
  timeout client          20s
  timeout server          20s
  timeout http-keep-alive 10s
  timeout check           10s

frontend kubernetes-api
  bind *:6443
  mode tcp
  option tcplog
  default_backend apiserver

backend apiserver
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server k8s-master001 IPMASTER01:6443 check
    server k8s-master002 IPMASTER02:6443 check
    server k8s-master003 IPMASTER03:6443 check
```

### Reinicie o serviço do Haproxy e marque como ativado no próximo boot

```bash
systemctl restart haproxy
systemctl enable haproxy
```

## Execute os proximos passos em todos os nodes do cluster

### Desative o Firewall

```bash
ufw disable
systemctl disable ufw
```

### Desative o swap

```bash
swapoff -a; sed -i '/swap/d' /etc/fstab
```

### Carregue os módulos necessários

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF
for a in `cat /etc/modules-load.d/k8s.conf `; do modprobe $a; done
```

### Carregue os parametros de kernel relacionados a rede do cluster

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## Configuração Kubernetes

### Adicione o repositorio APT

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

### Instale os componentes do Kubernetes

```bash
apt update && apt install -y kubeadm kubelet kubectl
```

### Instalando o containerd

``` bash
apt update && apt install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml
systemctl enable containerd
systemctl start containerd
```

## Em qualquer node master (Ex: k8s-master001)

### Inicializando o Cluster Kubernetes

```bash
kubeadm init --control-plane-endpoint="ENDEREÇOLOADBALANCER:6443" --upload-certs --apiserver-advertise-address=$(hostname -I | awk '{print $1}')
```

Copie os comandos para juntar outros nodes e salve num bloco de notas, iremos utilizar na sequencia.

### Deploy weavenet CNI

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## Junte os demais nodes ao cluster

> Use os respectivos comandos kubeadm join que você copiou da saída do comando kubeadm init no primeiro master.
>
> IMPORTANTE: Caso sua máquina tenha mais de uma interface de rede, você também precisa passar --apiserver-advertise-address ao comando join ao ingressar os demais nodes master, caso não seja definido irá utilizar a interface padrão.

## Habilitando o auto-complete do kubectl

### Descomente as linhas conforme abaixo em ~/.bashrc

```bash
#if [ -f /etc/bash_completion ] && ! shopt -oq posix; then
#    . /etc/bash_completion
#fi
```

### Agora precisamos gerar o arquivo de auto complete do kubectl

```bash
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
bash
```

### Existe também a opção de criar um alias

```bash
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
bash
```

## Verificando o cluster

```bash
kubectl cluster-info
kubectl get nodes
kubectl get all --all-namespaces
crictl ps
crictl --help
```

Divirta-se!!! :)
