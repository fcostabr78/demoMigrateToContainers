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
<br></br>

## Executar a Coleta do Ambiente

### Executar conexão SSH com a VM instalada
```
$ gcloud compute ssh --zone "${ZONE}" "ledgermonolith-service"  --project "${PROJECT_ID}"
```

### Download da ferramenta de coleta
```
$ mkdir m4a && cd m4a
$ curl -O "https://mfit-release.storage.googleapis.com/1.12.1/mfit-linux-collect.sh"
$ chmod +x mfit-linux-collect.sh
```

### Download da ferramentas de analise
```
$ curl -O "https://mfit-release.storage.googleapis.com/1.12.1/mfit"
$ chmod +x mfit
```

### Execute a coleta
```
$ sudo ./mfit-linux-collect.sh
```

O script de coleta gera um arquivo TAR chamado m4a-collect-ledgermonolith-service-TIMESTAMP.tar e o salva no diretório atual. O carimbo de data/hora está no formato YYYY-MM-DD-hh-mm.

### Salve o retorno 
```
$ ./mfit assess sample m4a-collect-ledgermonolith-service-2022-10-24-17-43.tar --format json > ledgermonolith-mfit-report.json
$ exit
```

### Copie o arquivo gerado ao ambiente do Cloud Shell
```
$ gcloud compute scp --tunnel-through-iap --zone "${ZONE}" --project "${PROJECT_ID}" \
ledgermonolith-service:~/m4a/ledgermonolith-mfit-report.json ${HOME}/
```

### Desde a console no navegador, solicite o download a máquina local
$ cloudshell download ${HOME}/ledgermonolith-mfit-report.json

### Valide o fit de migração apontado pela ferramenta

> Em "Migrate To Containers", selecione "Open Fit Assessment" e abra o aquivo ".json"
> será apresentado o detalhe da migração sugerida a VM conforme a imagem abaixo

![This is an image](https://github.com/fcostabr78/demoMigrateToContainers/blob/main/report_json.png?raw=true)

> para ter maiores informações sobre msgs apontadas na avaliação de migracao https://cloud.google.com/migrate/containers/docs/fit-assessment-rules?hl=pt-br

> :checkered_flag: **Passo 2 concluído com sucesso.**  Na etapa a seguir, você criará o cluster do GKE usado como um cluster de processamento. É aqui que você instala o Migrate to Containers e executa a
> migração. Não use o mesmo cluster que o Bank of Anthos em execução para não interromper os serviços. Isso é intencional. Depois que a migração for 
> concluída, exclua esse cluster de processamento. Vamos ao passo 3
--
<br></br>

## Preparar Plano de Migração

> Antes de iniciar essa etapa habilite a criacao de service account 
> em IAM/Organization, Policies, atribuir o enforcement como off as variáveis 
> e disableServiceAccountKeyCreation e disableServiceAccountCreation.
> Também habilitar actions API e Cloud Resource Manager API

### Criar o cluster GKE de processamento migration-processing e instalar o migrate to containers

```
$ alias k=kubectl
$ gcloud container clusters create migration-processing \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type e2-standard-4 \
--num-nodes 1 \
--subnetwork default --scopes "https://www.googleapis.com/auth/cloud-platform" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing \
--enable-shielded-nodes \
--shielded-secure-boot \
--shielded-integrity-monitoring
```

### Configurar os componentes do "migration to containers" no cluster de processamento
```
$ gcloud iam service-accounts create m4a-install --project=${PROJECT_ID}
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \ 
--member="serviceAccount:m4a-install@${PROJECT_ID}.iam.gserviceaccount.com" --role="roles/storage.admin"
$ gcloud iam service-accounts keys create m4a-install.json \
--iam-account=m4a-install@vibrant-sound-352319.iam.gserviceaccount.com --project=${PROJECT_ID} 
$ gcloud container clusters get-credentials migration-processing --zone=${ZONE} --project=${PROJECT_ID} 
$ migctl setup install --json-key=m4a-install.json
$ migctl doctor
```

### Criar a origem da migração que representa a plataforma de origem (Compute Engine, VMWare, AWS ou Azure).
```
$ gcloud iam service-accounts create m4a-ce-src --project=${PROJECT_ID} 
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member="serviceAccount:m4a-ce-src@${PROJECT_ID}.iam.gserviceaccount.com" --role="roles/compute.viewer" #2
$ gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member="serviceAccount:m4a-ce-src@${PROJECT_ID}.iam.gserviceaccount.com" --role="roles/compute.storageAdmin" #2
$ gcloud iam service-accounts keys create m4a-ce-src.json \
--iam-account=m4a-ce-src@${PROJECT_ID}.iam.gserviceaccount.com --project=${PROJECT_ID}
$ migctl source create ce quickstart-source --project=${PROJECT_ID} --json-key=m4a-ce-src.json
```

### Determinar origem e candidatos

1. Na console do GCP, em "Migrate to Containers", clique em "Source & Candidates"<br>
2. No campo Source, selecione quickstart-source<br>
3. Clique no botão azul "Assess Now"<br>
4. Será apresentado a mensagem "Assessment is in progress"

### Crie a migração

5. Ao finalizar será apresentado "ledgermonolith-service", de tipo VM, numa lista
6. Clique no botão de três pontos e selecione "Create Migration" (conforme apresenta a imagem abaixo)

![This is an image](https://github.com/fcostabr78/demoMigrateToContainers/blob/main/create_migration.png?raw=true)

7. Defina o nome da migracao "ledgermonolith-migration"
8. Atribua o "WorkLoad Type" como Linux system container
9. Confirme a criação da migração

> Ao ir a lista de Migrações, quando o plano estiver finalizado o status será "Migration plan generated"

![This is an image](https://github.com/fcostabr78/demoMigrateToContainers/blob/main/mig_plan_gen.png?raw=true)

:checkered_flag: **Passo 3 concluído com sucesso.**
<br><br/>

## Executar a Migração da Máquina Virtual

1. Clique no nome do plano, e em detalhes, na aba "Data Configuration" adicione o yaml abaixo no editor

```
volumes:
 - deploymentPvcName: ledgermonolith-db
   folders:
  # Folders to include in the data volume, e.g. "/var/lib/postgresql"
  # Included folders contain data and state, and therefore are automatically excluded from a generated container image
   - /var/lib/postgresql
   newPvc:
     spec:
       accessModes:
       - ReadWriteOnce
       resources:
         requests:
           storage: 10G
```

> Isso garante o banco de dados é mantido durante a migração. Clique em Salvar.

2. Clique em "Salvar" (ATENCAO)

3. Na aba "Migration Plan", em deployment, verifique se o serviço tem o nome ledgermonolith-service, a porta 8080 e o protocolo TCP. O objeto será parecido com:

```
endpoints:
  - name: ledgermonolith-service
    port: 8080
    protocol: TCP
```

4. Clique em "Save and Generate Artefacts"

![This is an image](https://github.com/fcostabr78/demoMigrateToContainers/blob/main/gen_artefact.png?raw=true)
