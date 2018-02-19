
# Mobile Security Framework

![MobSF](images/logo_MobSF.png)

Você já ouviu falar na Mobile Security Framework? É uma ferramenta do MobSF que analisa a APK e verifica a vulnerabilidade dela. Se vcoê quiser saber mais, neste post vou mostrar como funciona, a instalação e como usar, além de como subir uma imagem em Docker e usar REST API, e uma POC em Gitlab CI.

### O que é o MobSF?

Como já falei ali em cima, MobSF é uma ferramenta que analisa a APK e verifica a vulnerabilidae dela.  
Com ela podemos fazer tanto a análise estática quanto a dinâmica. Na estática, podemos realizar revisão de código,  detectar permissão e configurações, verificar ssl overriding, ssl bypass, criptografia fraca, códigos ofuscados, permissões impróprias, segredos codificados, uso indevido de API e armazenamento inseguro de arquivos.

O analisador dinâmico, por sua vez, executa o aplicativo em uma máquina virtual ou em um dispositivo configurado e detecta os problemas em tempo de execução. Uma análise mais aprofundada é feita nos pacotes de rede capturados, tráfego HTTPS desencrito, despejos de aplicativos, registros, relatórios de erros ou falhas, informações de depuração, rastreamento de pilha e os recursos do aplicativo, como definir arquivos, preferências e bancos de dados.

Essa estrutura é altamente escalável e você pode adicionar suas regras personalizadas com facilidade, e no final dos testes você consegue um relatório rápido e limpo. A ferramenta pode ser utilizada tanto para Android e IOS, e suporta o binário (APK e IPA), e códigos-fontes compactados. O MobSF também pode ser usado via API, e vamos mostrar isso daqui a pouquinho.

### Primeiros passos  

Para funcionar e rodar MobSF com sucesso é só seguir os próximos passos. Vamos lá?

Faça o download do Oracle JDK 1.7 (ou superior) - **[Java JDK Download](http://www.oracle.com/technetwork/java/javase/downloads/index.html)**  
Faça o git clone do repositório do MobSF, no qual estarão alguns scripts que precisam ser executados. Depois disso, entre no repositório.

```
git clone https://github.com/MobSF/Mobile‐Security‐Framework‐MobSF.git
cd Mobile‐Security‐Framework‐MobSF
```

Faça a **[instalação do Python](https://www.python.org/downloads/)** (2.7 ou superior), pois os scripts são em Python, e rode o comando abaixo para que as dependências possam ser instaladas:

```
sudo apt install build‐essential libssl‐dev libffi‐dev python‐dev
pip install ‐r requirements.txt ‐‐user
```

O comando a seguir permite que você rode o MobSF:

```
python manage.py runserver
```

Ao executar esse comando ele vai gerar um Token, que sempre será o mesmo em qualquer máquina que ele estiver. Ele vai ser usado para podermos fazer a análise via API, portanto deixa guardado.

### MobSF com Docker

Faça a instalação do Docker  
**[Documetação](https://docs.docker.com/)**  
**[Instalação](https://docs.docker.com/engine/installation/linux/ubuntu/)**

E agora vamos subir a imagem da ferramenta. Faça o clone do repositório, caso não tenha feito ainda:

```
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
```

Assim que o repositório for clonado entre na pasta dele e execute estes comandos:

```
docker build -t mobsf .
docker run -it -p 8000:8000 mobsf
```
ou
```
docker pull opensecurity/mobile-security-framework-mobsf
docker run -it -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

### REST API

Agora vou mostrar como fazer a análise e gerar o report via API.
O comando a seguir faz o upload para o MobSF, que poderá ser visto no browser.

```
curl -F 'file=@/<path>/<apk>.apk' http://localhost:8000/api/v1/upload -H "Authorization: <Token>"
```

O "<Token>" é o mesmo que é gerado quando rodamos o MobSF.
Quando o comando for executado será gerado um hash (que será diferente para cada APK/IPA), que também tem que ser guardado para que possamos utilizar para fazer o scan e report.
Para fazer o scan do apk o comando é este:

```
curl -X POST --url http://localhost:8000/api/v1/scan --data "scan_type=apk&file_name=<apk>.apk&hash=<hash>" -H "Authorization: <Token>"

```

Por fim, mas não menos importante, para gerar o report:
```
curl -X POST --url http://localhost:8000/api/v1/delete_scan --data "hash=<hash>" -H "Authorization:<Token>" -o <name_report>.pdf

```

### POC do MobSF com GitlabCI

Antes de entrar neste tópico, acesse **[este post](https://concrete.com.br/2018/02/15/conheca-o-gitlab-ci/)** para saber como colocar o Gitlab CI pra rodar.

Foram criados três jobs, e para cada um deles foi feito um script para acessar a API.
O primeiro passo é configurar as variáveis que serão utilizadas, neste caso o Token e a url do MobSF. Para isso, siga os passos:

Projects >> Settings >> Pipelines >> Add new variable

Para iniciar o YAML:

```
# Job da pipeline
# Arquivo: .gitlab-ci.yml
stages:
  - upload
  - scan
  - report
```

##### Jobs  
###### Upload  
Aqui tem as configurações do job para que o upload do arquivo para o MobSF seja feito:
```
# Arquivo .gitlab-ci.yml
upload:
  stage: upload
  script:
   - find *.apk > apk # Encontra a APK e coloca o nome do mesmo em um arquivo "apk".
   - export APK=$(cat apk) # Como precisamos utilizar o mesmo em uma variavel, fazemos um export para isso.
   - bash upload.sh $URL_MOBSF $TOKEN_MOBSF # Rodamos o script para fazer o upload, o mesmo esta logo abaixo. Junto com as variáveis que configuramos.
  artifacts:
   paths:
    - hash
    - apk
```

E aqui está o script para fazer o upload da APK para o MobSF via API:

```
#!/bin/bash
# Upload a new APK to MobSF
# Arquivo upload.sh

URL=$1
Token=$2
curl -F "file=@/builds/lucas.mucheroni/MobSF/$APK" "$URL"/api/v1/upload -H "Authorization: $Token" | jq --raw-output .hash > hash

# Com o jq é possivel fazer um "filtro" e trazer apenas a hash do curl, e é colocado dentro de um arquivo "hash"

```

###### Scan  
O job para fazer o scan do APK:

```
# Arquivo .gitlab-ci.yml

scan:
  stage: scan
  script:
   - export hash=$(cat hash) # Export para passar o valor do hash para uma variavel.
   - export APK=$(cat apk) # Como é feito no job anterior o mesmo é feito com o job de scan
   - bash scan.sh $URL_MOBSF $TOKEN_MOBSF # Um script para fazer o scan via API, segue abaixo o script. Junto com as variáveis configuradas.
  artifacts:
   paths:
    - hash
    - apk
```

Este script faz o scan do APK que foi feito upload no job anterior:

```
#!/bin/bash
# Scan APK
# Arquivo scan.sh

URL=$1
Token=$2

curl -X POST --url $URL/api/v1/scan --data "scan_type=apk&file_name=$APK&hash=$hash" -H "Authorization:$Token"
```

###### Report
Por último, vamos gerar o report por meio do scan que foi feito no job anterior:

```
# Arquivo .gitlab-ci.yml

report:
  stage: report
  script:
   - export hash=$(cat hash) # Faz o export novamente para utilizar como variavel.
   - export APK=$(cat apk) # O mesmo aqui.
   - bash report.sh $URL_MOBSF $TOKEN_MOBSF # Executa um script para gerar o report, o mesmo segue abaixo. Junto com as variáveis configuradas.
   - bash slack.sh 'Link para download:' 'https://example.com/'$CI_PROJECT_NAMESPACE'/'$CI_PROJECT_NAME'/builds/artifacts/master/download?job=report' # Executa um script para enviar uma notificação no slack, com o link para fazer o download do report.
  artifacts:
    name: "report"
    paths:
    - /builds/lucas.mucheroni/MobSF/*.pdf # Arquiva o report e fica disponivel para download até uma semana.
    expire_in: 1 week
```

Script para gerar o report do scan, que também é feito via API:

```
#!/bin/bash
# Report
# Arquivo report.sh

URL=$1
Token=$2

curl -X POST --url $URL/api/v1/download_pdf --data "hash=$hash&scan_type=apk" -H "Authorization:$Token" -o /builds/lucas.mucheroni/MobSF/$APK.pdf

```

Para finalizar, os scripts e o yaml do seu repositório têm que ficar assim:

![Repo](images/repo.png)

Lembrando que a APK está no repositório do próprio MobSF (Mobile-Security-Framework-MobSF/StaticAnalyzer/test_files).

E se você quiser saber mais, aqui tem a documentação completa do MobSF:

**[Documentação](https://github.com/MobSF/Mobile-Security-Framework-MobSF/wiki/1.-Documentation#configuring-dynamic-analyzer-with-with-mobsf-android-412-arm-emulator)**  
**[REST API](https://github.com/MobSF/Mobile-Security-Framework-MobSF/wiki/3.-REST-API-Documentation)**  
**[Docker](https://github.com/MobSF/Mobile-Security-Framework-MobSF/wiki/7.-Docker-Container-for-MobSF-Static-Analysis)**  
**[Repositório MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF)**  

E é isso! Dúvidas e sugestões são muito bem-vindas, só usar os campos abaixo. :smile: Até a próxima! :wave:

--

Quer trabalhar com DevOps de verdade? [Acesse aqui](https://jobs.kenoby.com/concrete).
