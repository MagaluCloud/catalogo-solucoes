 Rundeck Jobs for Magalu Cloud Backup                                                                                                                                  
                                                                                                                                                                        
Este repositório contém scripts de job do Rundeck para automatizar o backup (criação de snapshots) e a limpeza de snapshots de instâncias na Magalu Cloud.              
                                                                                                                                                                        
## Scripts                                                                                                                                                              
                                                                                                                                                                        
### 1. `mgc-instance-backup.yaml` - Backup de Instâncias MGC                                                                                                            
                                                                                                                                                                        
Este job é responsável por criar snapshots de máquinas virtuais (VMs) na Magalu Cloud.                                                                                  
                                                                                                                                                                        
**Funcionalidades:**                                                                                                                                                    
- Cria snapshots de VMs com base em um filtro de nome.                                                                                                                  
- Exclui automaticamente VMs de cluster Kubernetes (com prefixo `k8s-`).                                                                                                
- Realiza uma limpeza automática de snapshots antigos para a VM que acabou de receber o backup, mantendo os últimos 7 dias.                                             
                                                                                                                                                                        
**Parâmetros:**                                                                                                                                                         
- **`MgcApiKey` (obrigatório):** A chave de API para autenticar com a Magalu Cloud.                                                                                     
- **`Nome da VM` (opcional):** O nome ou parte do nome da VM para a qual o backup deve ser criado. Se este campo for deixado em branco, o job tentará criar um backup   
para todas as VMs elegíveis no ambiente.                                                                                                                                
                                                                                                                                                                        
**Como Funciona:**                                                                                                                                                      
1. O script lista todas as instâncias e filtra aquelas que não são parte de um cluster Kubernetes.                                                                      
2. Se um nome de VM for fornecido, ele filtra a lista para incluir apenas as VMs correspondentes.                                                                       
3. Para cada VM na lista final, um novo snapshot é criado com o nome no formato `snap-<NOME_DA_VM>-<DATA_HORA>`.                                                        
4. Após a criação do snapshot, ele remove os snapshots com mais de 7 dias de idade **para aquela VM específica**.                                                       
                                                                                                                                                                        
### 2. `mgc-backup-cleanup.yaml` - Limpeza de Snapshots MGC                                                                                                             
                                                                                                                                                                        
Este job é uma ferramenta de limpeza global que remove snapshots com base em um período de retenção.                                                                    
                                                                                                                                                                        
**Funcionalidades:**                                                                                                                                                    
- Remove snapshots de VMs que são mais antigos que um número de dias especificado.                                                                                      
- Possui um modo de "destruição" que remove **TODOS** os snapshots se nenhum período de retenção for definido.                                                          
                                                                                                                                                                        
**Parâmetros:**                                                                                                                                                         
- **`MgcApiKey` (obrigatório):** A chave de API para autenticar com a Magalu Cloud.                                                                                     
- **`Retenção (dias)` (opcional):** O número de dias que os snapshots devem ser mantidos. Snapshots mais antigos que este valor serão excluídos.                        
                                                                                                                                                                        
**⚠️ Atenção:**                                                                                                                                                         
Se o parâmetro **`Retenção (dias)`** for deixado em branco, o script interpretará que a intenção é remover **TODOS** os snapshots da conta. Use com extremo cuidado.    
                                                                                                                                                                        
**Como Funciona:**                                                                                                                                                      
1. O script lista todos os snapshots disponíveis na conta.                                                                                                              
2. Ele calcula a idade de cada snapshot.                                                                                                                                
3. Se o campo `Retenção (dias)` estiver vazio, ele deleta todos os snapshots.                                                                                           
4. Se um valor for fornecido, ele deleta apenas os snapshots cuja idade em dias for maior que o valor de retenção.                                                      
                                                                                                                                                                        
## Uso Recomendado                                                                                                                                                      
                                                                                                                                                                        
- **Backup Diário:** Agende o job `MGC - Instance Backup` para ser executado diariamente, deixando o campo `Nome da VM` vazio para garantir que todas as VMs sejam      
copiadas.                                                                                                                                                               
- **Limpeza Manual/Es esporádica:** Use o job `MGC - Snapshot Cleanup` para manutenções manuais ou para forçar uma política de limpeza mais ampla, se necessário.       
Devido ao seu potencial destrutivo, não é recomendado agendá-lo sem uma supervisão cuidadosa