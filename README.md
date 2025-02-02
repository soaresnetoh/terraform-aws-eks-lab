Este repositório pode ser usado para criar um cluster Kubernetes utilizando o serviço EKS da AWS e possui alguns exemplos de deploys a serem feitos no cluster.

Será necessário a criação de um usuário do tipo "service" no IAM da AWS e em seguida logar com esse usuário no AWS CLI.
As permissões necessárias para esse usuário (ainda em validação para otimização) estão disponíveis na imagem "IAM_AWS.png"

# Criação do cluster:
Arquivos disponíveis dentro da pasta "terraform"

A criação do cluster e todas as suas dependências são feitas via terraform através dos comandos (comando deve ser executado dentro da pasta "terraform"):
- terraform init
- terraform plan
- terraform apply [yes quando for solicitado]

Ao final da execução do terraform (que deve durar de 20 a 30 minutos), será necessário execução do comando abaixo para configurar o kubectl para acesso ao cluster. Necessário que o kubectl esteja instalado na estação e alterar os parâmetros de região e nome do cluster no comando (essas informações são disponibilizadas no final da execução do terraform):
aws eks --region us-east-1 update-kubeconfig --name nome_cluster

Dos arquivos de configuração do terraform, apenas 'eks-cluster.tf' e 'vpc.tf' podem/precisam ser alterados antes da criação do cluster.

- eks-cluster.tf: Permite especificar a versão do cluster (versão do kubernetes), definir as tags que serão utilizadas no ambiente (muito importante para questões de faturamento) e a quantidade e tipo de nodes que serão utilizados.

- vpc.tf: Permite definir o nome do cluster, as tags utilizadas pelas VPCs e Subnets e os endereços IPs que serão utilizados no ambiente.

# Deploy no Kubernetes:
Arquivos disponíveis dentro da pasta "k8s"

Após criação do cluster via terraform, o acesso estará apto a ser feito via kubectl e poderá ser realizado alguns deploys no kubernetes.

Nessa pasta estão os arquivos que farão o deploy de duas aplicações simples (nginx apenas para exibir conteúdo HTML na tela), além dos manifestos que serão utilizados pelo Prometheus e Grafana (ambos via HELM) e para aplicação de um Ingress via Nginx. O passo a passo dessas aplicações estão definidos abaixo:

#1 - Instalar Metric Server para coleta de dados dos Pods:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

#2 - Criação dos namespaces a serem utilizados pelas aplicações e para monitoramento:
kubectl create namespace monitoring
kubectl create namespace api-services

#3 - Instalação do Prometheus via HELM:
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
helm repo update
helm inspect values prometheus-communitykubectl apply -f nginxingress.yaml -n api-services
helm upgrade --install prometheus prometheus-community/prometheus --values values-prometheus.yaml  --namespace monitoring
kubectl get all -n monitoring (para verificar se todos os objetos foram aplicados)

#4 - Instalação do Grafana via HELM:
helm repo add grafana https://grafana.github.io/helm-charts
helm repo upDeploy --install grafana grafana/grafana --values values-grafana.yaml  --namespace monitoring
kubectl get svc -n monitoring (copiar a URL que foi definida para o serviço do Grafana)

O acesso web no Grafana será via URL descrita no comando acima e a senha default a ser usada está disponível através do comando abaixo:
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

Via web no Grafana, adicionar o Prometheus como source via endereço "http://prometheus-server" e fazer o import de alguns dashboard já preparados para EKS e Kubernetes (no teste, utilizei os dashboard de IDs 11875, 7249, 315 e 6417. Alguns desses não trazem nenhuma informação)

#5 - Deploy de duas aplicações simples:
Os arquivos "deployment-blue" e "deployment-green" podem ser usados para deploy de duas aplicações simples que serão utilizadas apenas para verificar funcionamento do Ingress-Controller
kubectl apply -f deployment-blue.yaml -n api-services
kubectl apply -f deployment-green.yaml -n api-services

#6 - Instalar e aplicar serviço de Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/aws/deploy.yaml
kubectl get ingress -A (copiar a URL definida para o serviço de Ingress e inserir no arquivo a seguir)
kubectl apply -f nginxingress.yaml -n api-services (alterar nesse manifesto apenas a URL vista anteriormente)

Após a aplicação desse passo, as duas aplicações nginx estarão disponíveis via Ingress, bastanto alterar o "/name" no final da URL.

Obs.: Foi utlizado uma versão mais antiga do Ingress Controller devido alguns erros que ainda não descobri da nova versão.

# Informações Finais:

Com os passos acima, o laboratório estará pronto para ser utilizado em sua versão beta hehe.
Para evitar gastos desnecessários, destruir todo o cluster ao final do uso via comando abaixo, que deve ser executado dentro da pasta "kubernetes":
- terraform apply -auto-approve

Obs.: A saída do comando de destroy apresentará erro devido criação do LoadBalancer não ter sido feita via terraform (ainda), será necessário exclusão manual via console da AWS e em seguida, executar o destroy novamente.

Outras documentações que me ajudaram bastante a entender o terraform foram:

https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest

https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws/latest

https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest