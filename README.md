# Projeto Recicla++
## Repositórios
[Página do projeto](https://www.recicle.app.br/)
- [Doador](https://github.com/recicleUSP/Donor) - Repositório do aplicativo do doador.
- [Coletor](https://github.com/leonardo8787/Coletor) -  Repositório do aplicativo do coletor.
- [Admistrativo](https://github.com/recicleUSP/siteAdmRecicle) -  Repositório do site de administração.
- [Servidor de rotas](https://github.com/louzeiro/osrm_recicleUSP/edit/main/README.md) -  Repositório das instruções para a configuração do servidor de rotas.


## Configuração do servidor de rotas

O Open Source Rounting Machine - OSRM, disponível em http://project-osrm.org/, é uma ferramenta de otimização de rotas. Diferente da maioria dos servidores de roteamento, o OSRM não usa uma variante A* para calcular o caminho mais curto, mas usa hierarquias de contração ou Dijkstra multinível [[1]](https://wiki.openstreetmap.org/wiki/Open_Source_Routing_Machine). Resultando em respostas rápidas e tornando o OSRM um bom candidato para aplicações que precisam de roteamento. Suas principais caracteríscas são:


- Roteamento muito rápido
- Altamente portátil
- O formato de dados simples facilita a importação de conjuntos de dados personalizados em vez de dados do OpenStreetMap ou importação de dados de tráfego
- Perfis de roteamento flexíveis (por exemplo, para diferentes modos de transporte)
- Respeita as restrições de curva, incluindo restrições condicionais baseadas em tempo
- Respeita as faixas de conversão
- Instruções passo a passo localizadas fornecidas por instruções de texto OSRM

É disponibilizado atráves de imagem Docker, facilitando a configuração do servidor de roteamento. A seguir serão apresentados os passos utilizados para configurar o servidor utilizado no projeto.

### Passo a passo
#### Instalação do Docker

    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    
Testando o docker
    
    sudo docker run hello-world

No terminal ocorrerá a saída abaixo. Caso contrário, favor consultar o matérial https://docs.docker.com/engine/install/ubuntu/ 

![image](https://github.com/louzeiro/osrm_recicleUSP/assets/7797969/dc0c4cc2-525a-41f0-9c3f-06d786b429d1)

#### OSRM
Inicialmente, instala-se as dependências do projeto
    
    sudo apt install build-essential git cmake pkg-config \
    libbz2-dev libxml2-dev libzip-dev libboost-all-dev \
    lua5.2 liblua5.2-dev libtbb-dev

Em seguida, cria-se uma pasta para receber o repositório

    sudo mkdir -p /srv/osrm
    cd /srv/osrm

Clona-se o repositório do backend do projeto OSRM

    sudo git clone https://github.com/Project-OSRM/osrm-backend.git

Cria-se o diretório para a compilação do projeto

    mkdir /srv/osrm/osrm-backend/build
    
Compila-se o projeto
    
    cd /srv/osrm/osrm-backend/build
    sudo make ..
    sudo cmake --build .
    sudo cmake --build . --target install
    

O próximo passo é baixar o mapa da região, nesse caso, utilizamos os dados dos estados da região sudeste. Para outras regiões do país, os dados estão disponíveis em http://download.geofabrik.de/south-america/brazil.html
    
    cd /srv/osrm/osrm-backend/
    sudo wget http://download.geofabrik.de/south-america/brazil/sudeste-latest.osm.pbf
    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/sudeste-latest.osm.pbf
    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/sudeste-latest.osrm
    sudo docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/sudeste-latest.osrm
    
Finalmente, os dados do mapa são adicionados ao container docker.
    
    sudo docker run -t -i -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --algorithm mld /data/sudeste-latest.osrm

Para testar a aplicação, realiza-se uma consulta de rota.

    curl "http://127.0.0.1:5000/route/v1/driving/-22.01353404635547,-47.88069891161758;-22.01841397120339,-47.88327352469903?steps=true"

## Vídeo tutorial
   
