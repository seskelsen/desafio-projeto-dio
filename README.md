# desafio-projeto-dio
Repositório para o desafio de projeto AZ900

## Passo a passo (resumo)

Este guia propõe uma arquitetura simples para praticar os fundamentos do AZ-900 (DIO), criando:
- 1 Resource Group
- 1 Virtual Network com 2 sub-redes (web e app)
- 1 Network Security Group para a sub-rede web
- 1 VM Linux (web)
- 1 Storage Account
- 1 App Service (Web App)
- Monitoramento básico

Você pode realizar pelo Portal Azure (cliques) ou, opcionalmente, pela Azure CLI (comandos). Escolha uma região próxima (ex.: Brazil South) e nomes únicos quando indicado.

---

### 1) Criar Resource Group
- Portal: Resource groups > Create > Nome: `rg-az900-lab` > Region: `Brazil South` > Review + Create.
- CLI (opcional):
```bash
az group create -n rg-az900-lab -l brazilsouth
```

### 2) Criar Virtual Network e Sub-redes
- Portal: Virtual networks > Create > Nome: `vnet-az900` > RG: `rg-az900-lab` > Address space: `10.0.0.0/16`.
  - Subnet1 (web): `snet-web` com `10.0.1.0/24`
  - Subnet2 (app): `snet-app` com `10.0.2.0/24` > Create.
- CLI (opcional):
```bash
az network vnet create -g rg-az900-lab -n vnet-az900 \
  --address-prefix 10.0.0.0/16 --subnet-name snet-web --subnet-prefix 10.0.1.0/24
az network vnet subnet create -g rg-az900-lab --vnet-name vnet-az900 \
  -n snet-app --address-prefixes 10.0.2.0/24
```

### 3) Criar Network Security Group (NSG) para a sub-rede web
- Portal: Network security groups > Create > Nome: `nsg-web` > RG: `rg-az900-lab` > Create.
  - Em `Inbound security rules`, adicione:
    - Allow HTTP (porta 80) de `Any`
    - Allow HTTPS (porta 443) de `Any`
    - Allow SSH (porta 22) apenas do seu IP (boa prática)
  - Associe o NSG à subnet `snet-web`.
- CLI (opcional):
```bash
az network nsg create -g rg-az900-lab -n nsg-web
az network nsg rule create -g rg-az900-lab --nsg-name nsg-web \
  -n allow-http --priority 1000 --access Allow --protocol Tcp --direction Inbound --destination-port-ranges 80
az network nsg rule create -g rg-az900-lab --nsg-name nsg-web \
  -n allow-https --priority 1010 --access Allow --protocol Tcp --direction Inbound --destination-port-ranges 443
# Substitua X.X.X.X/32 pelo seu IP público para restringir o SSH
az network nsg rule create -g rg-az900-lab --nsg-name nsg-web \
  -n allow-ssh --priority 1020 --access Allow --protocol Tcp --direction Inbound --source-address-prefixes X.X.X.X/32 --destination-port-ranges 22
az network vnet subnet update -g rg-az900-lab --vnet-name vnet-az900 -n snet-web --network-security-group nsg-web
```

### 4) Provisionar uma VM Linux (Ubuntu) na subnet web
- Portal: Virtual machines > Create > Azure virtual machine.
  - RG: `rg-az900-lab`, Nome: `vm-web-01`, Região: `Brazil South`, Image: `Ubuntu 22.04 LTS`, Size: `B1s`.
  - Autenticação por chave SSH (recomendado). Rede: VNet `vnet-az900`, Subnet `snet-web`, NSG: `nsg-web`.
- CLI (opcional):
```bash
az vm create -g rg-az900-lab -n vm-web-01 \
  --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-az900 --subnet snet-web \
  --nsg nsg-web --public-ip-sku Standard \
  --generate-ssh-keys
```

### 5) Criar uma Storage Account
- Portal: Storage accounts > Create > RG: `rg-az900-lab` > Nome único: `staz900<seu-sufixo>` > Region: `Brazil South` > Performance: `Standard` > Redundância: `LRS` > Create.
- CLI (opcional):
```bash
az storage account create -g rg-az900-lab -n staz900seusufixo -l brazilsouth --sku Standard_LRS
```

### 6) Criar um App Service (Web App)
- Portal: App services > Create > Publish: `Code` > Runtime Stack: escolha (ex.: `Python 3.12` ou `.NET 8`) > OS: Linux.
  - Region: `Brazil South` > App Service plan: criar `asp-az900` (SKU gratuito F1) > Nome único: `webaz900-<seu-sufixo>` > Review + Create.
- CLI (opcional):
```bash
az appservice plan create -g rg-az900-lab -n asp-az900 --sku F1 --is-linux
az webapp create -g rg-az900-lab -p asp-az900 -n webaz900-seusufixo --runtime "PYTHON:3.12"
```

### 7) Habilitar monitoramento básico
- Portal: Em cada recurso, abra `Metrics` para visualizar CPU/RAM/Network.
- Crie um Alert Rule (ex.: VM CPU > 80% por 5 minutos) em Monitor > Alerts > Create > Alert rule.
- Habilite Boot diagnostics na VM (caso não esteja ativo) em `Diagnostics settings`.

### 8) Validação
- Acesse o Web App: `https://webaz900-<seu-sufixo>.azurewebsites.net`.
- Acesse a VM por SSH: `ssh azureuser@<public-ip>` e rode `sudo apt update` para testar conectividade.
- Verifique métricas e alertas em Azure Monitor.

### 9) Limpeza (para evitar custos)
- Portal: Delete o Resource Group `rg-az900-lab`.
- CLI (opcional):
```bash
az group delete -n rg-az900-lab --yes --no-wait
```

---

Dicas:
- Use nomes consistentes e regiões próximas para menor latência.
- Trate chaves/segredos com segurança (evite commitá-los).
- Recursos gratuitos/baixo custo: App Service F1, VM B1s (pago), Storage Standard LRS.