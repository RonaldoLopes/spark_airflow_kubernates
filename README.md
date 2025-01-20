# Sobre o Projeto
Executando Jobs Spark no Kubernetes com Airflow
O projeto tem como objetivo demonstrar como orquestrar e automatizar processos de Big Data utilizando Apache Airflow para executar jobs Spark. Este repositório detalha a criação de um pipeline de dados que integra essas duas tecnologias, fornecendo uma base sólida para automatizar e gerenciar fluxos de trabalho complexos de dados.

# Feito Com

Abaixo segue o que foi utilizado na criação deste projeto:

- [Minikube](https://minikube.sigs.k8s.io/docs/start/) - Ferramenta de código aberto que permite criar um ambiente de teste do Kubernetes em sua máquina local. Com o Minikube, é possível criar e implantar aplicativos em um cluster Kubernetes em sua máquina local.
- [Helm](https://helm.sh/) - Ferramenta de gerenciamento de pacotes de código aberto para o Kubernetes. O Helm permite empacotar aplicativos Kubernetes em um formato padrão chamado de gráfico, que inclui todos os recursos necessários para implantar o aplicativo, incluindo configurações e dependências.
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - Ferramenta declarativa que usa a abordagem GitOps para implantar aplicações no Kubernetes. O Argo CD é gratuito, tem código aberto, é um projeto incubado pela CNCF, e possui uma interface web de visualização e gerenciamento dos recursos, mas também pode ser configurado via linha de comando.
- [Spark](https://spark.apache.org/) - O Spark é um framework de processamento de dados distribuído e de código aberto, que permite executar processamento de dados em larga escala, incluindo processamento em batch, streaming, SQL, machine learning e processamento de gráficos. Ele foi projetado para ser executado em clusters de computadores e fornece uma interface de programação fácil de usar para desenvolvedores;
- [Airflow](https://airflow.apache.org/) - O Airflow é uma plataforma de orquestração de fluxo de trabalho de dados de código aberto que permite criar, agendar e monitorar fluxos de trabalho complexos de processamento de dados. Ele usa uma linguagem de definição de fluxo de trabalho baseada em Python e possui uma ampla gama de conectores pré-construídos para trabalhar com diferentes sistemas de armazenamento de dados, bancos de dados e ferramentas de processamento de dados;
- [Reflector](https://github.com/emberstack/kubernetes-reflector) - O Reflector é uma ferramenta de sincronização de estado de código aberto que permite sincronizar recursos Kubernetes em diferentes clusters ou namespaces. Ele usa a abordagem de controlador de reconciliação para monitorar e atualizar automaticamente o estado dos recursos Kubernetes com base em um estado desejado especificado;
- [Minio](https://min.io/) - O Minio é um sistema de armazenamento de objetos de código aberto e de alta performance, compatível com a API Amazon S3. Ele é projetado para ser executado em clusters distribuídos e escaláveis e fornece recursos avançados de segurança e gerenciamento de dados;
- [Postgres](https://www.postgresql.org/) - O Postgres é um sistema de gerenciamento de banco de dados relacional de código aberto, conhecido por sua confiabilidade, escalabilidade e recursos avançados de segurança. Ele é compatível com SQL e é usado em uma ampla gama de aplicativos, desde pequenos sites até grandes empresas e organizações governamentais.

```
minikube start --driver=docker --memory=8000 --cpus=2
```

Para acessar alguns serviços via loadbalancer no Minikube, é necessário utilizar o [tunelamento do minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/#example-of-loadbalancer). Para isso, abra uma nova aba no seu terminal e execute o seguinte comando:
```sh
minikube tunnel
```

## Ferramentas
```sh
kubectl create namespace orchestrator
kubectl create namespace database
kubectl create namespace ingestion
kubectl create namespace processing
kubectl create namespace datastore
kubectl create namespace deepstorage
kubectl create namespace cicd
kubectl create namespace app
kubectl create namespace management
kubectl create namespace misc
```

Instale o argocd que será responsavel por manter nossas aplicações:
```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace cicd --version 5.27.1
```
```sh
# create a load balancer
kubectl patch svc argocd-server -n cicd -p '{"spec": {"type": "LoadBalancer"}}'
```
Em seguida instale o argo cli para fazer a configuração do repositorio:
```sh
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```
Em seguida armazene o ip atribiudo para acessar o argo e faça o login no argo, com os seguintes comandos:
```sh
ARGOCD_LB=$(kubectl get services -n cicd -l app.kubernetes.io/name=argocd-server,app.kubernetes.io/instance=argocd -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}")

# get password to log into argocd portal
# argocd login 192.168.0.200 --username admin --password UbV0FdJ2ZNCD8kxU --insecure
kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d | xargs -t -I {} argocd login $ARGOCD_LB --username admin --password {} --insecure
```
Uma vez feita a autenticação não é necessario adicionar um cluster, pois o argo esta configurado para usar o cluster em que ele esta instalado, ou seja, o cluster local ja esta adicionado como **`--in-cluster`**, bastando apenas adicionar o seu repositorio com o seguinte comando:

```sh
argocd repo add git@github.com:GersonRS/hands-on-running-spark-jobs-with-airflow.git --ssh-private-key-path ~/.ssh/id_ed25519 --insecure-skip-server-verification
```
> caso queira ver o password do argo para acessar a interface web execute este comando: `kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d`

> Lembrando que para este comando funcionar é necessario que você tenha uma `chave ssh` configurada para se conectar com o github no seu computador. Caso não tenha, use [este guia](https://docs.github.com/pt/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) para criar uma e [adiciona-la ao github](https://docs.github.com/pt/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).

Para acessar o argocd pelo IP gerado no Loadbalancer execute o comando:

```sh
echo http://$ARGOCD_LB
```

Uma vez que tenha acessado a pagina de autenticação do argocd use o `username` admin e o password gerado na instalação do argocd, executando o comando:

```sh
kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d
```

É hora de adicionar as ferramentas necessárias para o pipeline de dados. Para isso, é preciso criar secrets para armazenar senhas e informações confidenciais acessíveis pelas aplicações e processos do Spark. É necessário que eles estejam no namespace onde a aplicação está rodando e também no namespace processing, onde serão executados os processos do spark. Para realizar essa replicação dos secrets em diferentes namespaces, podemos usar o **[Reflactor](https://github.com/EmberStack/kubernetes-reflector)**, que evita a necessidade de recriar os secrets em namespaces diferentes. Para utilizá-lo, basta executar o comando a seguir:

Agora é hora de adicionar as outras ferramentas necesarias para o nosso pipeline de dados. E para isto precisamos criar `secrets` para armazenar senhas e informações censiveis, que  sejam acessiveis pelas aplicações e processos do **`Spark`**, por isso é necessario que eles estejam no namespace onde esta rodando a aplicação e também no namespace processing, onde será executado os processos do `spark`. Então para isto podemos usar o **[Reflactor](https://github.com/EmberStack/kubernetes-reflector)**, que ajudará a replicar os secrets nos namespaces necessários e a manter a segurança dos dados. Além disso, os comandos para instalar as configurações de acesso são importantes para garantir que apenas as pessoas autorizadas possam acessar os recursos do seu cluster, e para isto execute este comando:

```sh
kubectl apply -f manifests/management/reflector.yaml
```

> Antes de executar os comandos, você pode alterar os secrets dos arquivos localizados na pasta `secrets/` se quiser mudar as senhas de acesso aos bancos de dados e ao storage.

Após o Reflector estar funcionando, execute o comando que cria os secrets nos namespaces necessários:

```sh
# secrets
kubectl apply -f manifests/misc/secrets.yaml
```

> Caso não queira instalar o Reflactor para automatizar o processo de criar o secret em vários namespaces diferentes, você pode replicar manualmente o secret para outro namespace executando este comando:
`kubectl get secret minio-secrets -n deepstorage -o yaml | sed s/"namespace: deepstorage"/"namespace: processing"/| kubectl apply -n processing -f -`

Uma vez que os secrets estejam configurados, é possível instalar os bancos de dados e o storage do pipeline de dados com o seguinte comando:

```sh
# databases
kubectl apply -f manifests/database/postgres.yaml
# deep storage
kubectl apply -f manifests/deepstorage/minio.yaml
```

Por fim, instale o Spark e o Airflow, juntamente com suas permissões para executar os processos do Spark, executando os seguintes comandos:

```sh
# add & update helm list repos
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo add apache-airflow https://airflow.apache.org/
helm repo update
```

```sh
# processing
kubectl apply -f manifests/processing/spark.yaml
```
<!-- Para criar um imagem do airflow com algumas libs inclusas, para isto execute o seguinte comando:
```sh 
eval $(minikube docker-env)
docker build -f images/airflow/dockerfile images/airflow/ -t airflow:0.1
``` -->

Antes de instalar o Airflow, é preciso atender a um requisito: criar um secret contendo sua `chave ssh`, para que o Airflow possa baixar as `DAGs` necessárias por meio do `gitSync`. É possível criar esse secret com o seguinte comando:

> Lembrando que você deve ter a `chave ssh` configurada em sua máquina.

```sh
kubectl create secret generic airflow-ssh-secret --from-file=gitSshKey=$HOME/.ssh/id_ed25519 -n orchestrator
```

```sh
# orchestrator
kubectl apply -f manifests/orchestrator/airflow.yaml
```

Em seguida, instale as configurações de acesso:

```sh
kubectl apply -f manifests/misc/access-control.yaml
```

Para que seja possivel o Ariflow executar de maneira independente os processos spark é preciso que ele tenha uma conexão com o cluster, e para isto é necessario passar essa informação ao Airflow. Para adicionar a conexão com o cluster ao Airflow execute:
```sh
kubectl get pods --no-headers -o custom-columns=":metadata.name" -n orchestrator | grep scheduler | xargs -i sh -c 'kubectl cp images/airflow/connections.json orchestrator/{}:./ -c scheduler | kubectl -n orchestrator exec {} -- airflow connections import connections.json'
```
<!-- export SCHEDULER_POD_NAME="$(kubectl get pods --no-headers -o custom-columns=":metadata.name" -n orchestrator | grep scheduler)"
kubectl cp images/airflow/connections.json orchestrator/$SCHEDULER_POD_NAME:./ -c scheduler
kubectl -n orchestrator exec $SCHEDULER_POD_NAME -- airflow connections import connections.json -->