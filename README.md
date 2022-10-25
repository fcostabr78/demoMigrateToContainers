# Demo Migrate To Containers To AskAnAssociate Stores

> A maioria desses servidores é superutilizada e o cliente já faz uma avaliação usando "Migrate for Anthos and GKE" para ter uma ideia das opções de migração. A AskAnAssociate Stores deseja revisar com você os resultados do [relatório de avaliação](https://htmlpreview.github.io/?https://github.com/fcostabr78/demoMigrateToContainers/blob/main/mfit-report-private-cloud-full3.html) e uma demonstração de uma migração de VM para Containers para cargas de trabalho.

### Resumo da Demo

- **Passo 1**: Fazer Deploy da Solução utilizando Máquina Virtual
- **Passo 2**: Executar a Coleta do Ambiente
- **Passo 3**: Preparar Plano de Migração
- **Passo 4**: Executar a Migração da Máquina Virtual

## Fazer Deploy da Solução utilizando Máquina Virtual
### Cópia do repositório
```
$ git clone https://github.com/GoogleCloudPlatform/bank-of-anthos.git
$ cd bank-of-anthos/
```

### Configuração das variáveis de ambiente
```
$ export PROJECT_ID=<ID do seu projecto>
$ export ZONE=us-central1-c
```

### Deploy da VM Monolítica no GCP
```
$ make monolith-deploy
```

### Liberação de firewall
```
$ gcloud compute --project=${PROJECT_ID} firewall-rules create default-allow-http  \
--direction=INGRESS --priority=1000 --network=default --action=ALLOW  \
--rules=tcp:8080 --source-ranges=0.0.0.0/0 --target-tags=monolith
```

### Criação do Cluster BOA (Bank Of Anthos) no Google Kubernetes Engine - Modo Standard
```
$ gcloud container clusters create boa-cluster \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 \
--disk-size "10" \
--num-nodes "4" \
--monitoring=SYSTEM \
--logging=SYSTEM,WORKLOAD \
--subnetwork=default \
--enable-shielded-nodes \
--shielded-secure-boot \
--shielded-integrity-monitoring
```

#### Obter as credenciais de acesso
```
$ gcloud container clusters get-credentials boa-cluster --zone=${ZONE} --project=${PROJECT_ID}
```

#### Deploy dos recursos no cluster GKE
```
$ alias k=kubectl
$ sed -i 's/\[PROJECT_ID\]/'${PROJECT_ID}'/g' ${HOME}/bank-of-anthos/src/ledgermonolith/config.yaml
$ k apply -f src/ledgermonolith/config.yaml
$ k apply -f extras/jwt/jwt-secret.yaml
$ k apply -f kubernetes-manifests/accounts-db.yaml
$ k apply -f kubernetes-manifests/userservice.yaml
$ k apply -f kubernetes-manifests/contacts.yaml
$ sed -i 's/: frontend/: frontendgke/g' kubernetes-manifests/frontend.yaml
$ k apply -f kubernetes-manifests/frontend.yaml
$ sed -i 's/frontend:80/frontendgke:80/g' kubernetes-manifests/loadgenerator.yaml
$ k apply -f kubernetes-manifests/loadgenerator.yaml
$ k get po
```

#### Obter o IP de acesso ao produto
```
$ k get svc frontendgke | awk '{print $4}'
```
<br></br>
Acessar https://**IP_obtido_no_comando_acima** para acessar o Bank of Anthos
<br></br>
Faça o login com as credenciais padrão e veja as transações no painel. 

As transações listadas na tela vêm da VM que fizemos o Deploy.

<br></br>
:checkered_flag: **Passo 1 concluído com sucesso.** Vamos ao passo 2


