# Projeto Recicla++



## Repositórios
[Página do projeto](https://www.recicle.app.br/)
- [Doador](https://github.com/recicleUSP/Donor) - Repositório do aplicativo do doador.
- [Coletor](https://github.com/leonardo8787/Coletor) -  Repositório do aplicativo do coletor.
- [Admistrativo](https://github.com/recicleUSP/siteAdmRecicle) -  Repositório do site de administração.
- [Servidor de rotas](https://github.com/louzeiro/osrm_recicleUSP/edit/main/README.md) -  Repositório do site de administração.


## Configuração do servidor de rotas

O Open Source Rounting Machine - OSRM, é um projeto open-source, disponível em http://project-osrm.org/. Diferente da maioria dos servidores de roteamento, o OSRM não usa uma variante A* para calcular o caminho mais curto, mas usa hierarquias de contração ou Dijkstra multinível [[1]](https://wiki.openstreetmap.org/wiki/Open_Source_Routing_Machine). Resultando em respostas rápidas e tornando o OSRM um bom candidato para aplicações que precisam de roteamento. Suas principais caracteríscas são:

- Roteamento muito rápido
- Altamente portátil
- O formato de dados simples facilita a importação de conjuntos de dados personalizados em vez de dados do OpenStreetMap ou importação de dados de tráfego
- Perfis de roteamento flexíveis (por exemplo, para diferentes modos de transporte)
- Respeita as restrições de curva, incluindo restrições condicionais baseadas em tempo
- Respeita as faixas de conversão
- Instruções passo a passo localizadas fornecidas por instruções de texto OSRM

É disponibilizado atráves de imagem Docker, facilitando a configuração do servidor de roteamento. A seguir serão apresentados os passos utilizados para configurar o servidor utilizado no projeto.

### Passo a passo
Inicialmente, cria-se uma pasta 

    mkdir -p /srv/osrm
    cd /srv/osrm

Em seguida é clonado o repositório do backend do projeto OSRM e são criadas as pastas auxiliares

    git clone https://github.com/Project-OSRM/osrm-backend.git
    mkdir build

O próximo passo é baixar o mapa da região, nesse caso, utilizamos os dados dos estados da região sudeste. Para outras regiões do país, os dados estão disponíveis em http://download.geofabrik.de/south-america/brazil.html
    
    mkdir -p /srv/osrm/osrm-run
    wget http://download.geofabrik.de/south-america/brazil/sudeste-latest.osm.pbf

Pre-process the extract with the car profile and start a routing engine HTTP server on port 5000

    docker run -t -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-extract -p /opt/car.lua /data/berlin-latest.osm.pbf || echo "osrm-extract failed"

The flag `-v "${PWD}:/data"` creates the directory `/data` inside the docker container and makes the current working directory `"${PWD}"` available there. The file `/data/berlin-latest.osm.pbf` inside the container is referring to `"${PWD}/berlin-latest.osm.pbf"` on the host.

    docker run -t -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-partition /data/berlin-latest.osrm || echo "osrm-partition failed"
    docker run -t -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-customize /data/berlin-latest.osrm || echo "osrm-customize failed"

Note there is no `berlin-latest.osrm` file, but multiple `berlin-latest.osrm.*` files, i.e. `berlin-latest.osrm` is not file path, but "base" path referring to set of files and there is an option to omit this `.osrm` suffix completely(e.g. `osrm-partition /data/berlin-latest`).

    docker run -t -i -p 5000:5000 -v "${PWD}:/data" ghcr.io/project-osrm/osrm-backend osrm-routed --algorithm mld /data/berlin-latest.osrm

Make requests against the HTTP server

    curl "http://127.0.0.1:5000/route/v1/driving/13.388860,52.517037;13.385983,52.496891?steps=true"

Optionally start a user-friendly frontend on port 9966, and open it up in your browser

    docker run -p 9966:9966 osrm/osrm-frontend
    xdg-open 'http://127.0.0.1:9966'

In case Docker complains about not being able to connect to the Docker daemon make sure you are in the `docker` group.

    sudo usermod -aG docker $USER

After adding yourself to the `docker` group make sure to log out and back in again with your terminal.

We support the following images in the Container Registry:

Name | Description
-----|------
`latest` | `master` compiled with release flag
`latest-assertions` | `master` compiled with with release flag, assertions enabled and debug symbols
`latest-debug` | `master` compiled with debug flag
`<tag>` | specific tag compiled with release flag
`<tag>-debug` | specific tag compiled with debug flag

### Building from Source

The following targets Ubuntu 22.04.
For instructions how to build on different distributions, macOS or Windows see our [Wiki](https://github.com/Project-OSRM/osrm-backend/wiki).

Install dependencies

```bash
sudo apt install build-essential git cmake pkg-config \
libbz2-dev libxml2-dev libzip-dev libboost-all-dev \
lua5.2 liblua5.2-dev libtbb-dev
```

Compile and install OSRM binaries

```bash
mkdir -p build
cd build
cmake ..
cmake --build .
sudo cmake --build . --target install
```

### Request Against the Demo Server

Read the [API usage policy](https://github.com/Project-OSRM/osrm-backend/wiki/Demo-server).

Simple query with instructions and alternatives on Berlin:

```
curl "https://router.project-osrm.org/route/v1/driving/13.388860,52.517037;13.385983,52.496891?steps=true&alternatives=true"
```

### Using the Node.js Bindings

The Node.js bindings provide read-only access to the routing engine.
We provide API documentation and examples [here](docs/nodejs/api.md).

You will need a modern `libstdc++` toolchain (`>= GLIBCXX_3.4.26`) for binary compatibility if you want to use the pre-built binaries.
For older Ubuntu systems you can upgrade your standard library for example with:

```
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update -y
sudo apt-get install -y libstdc++-9-dev
```

You can install the Node.js bindings via `npm install @project-osrm/osrm` or from this repository either via

    npm install

which will check and use pre-built binaries if they're available for this release and your Node version, or via

    npm install --build-from-source

to always force building the Node.js bindings from source.

#### Unscoped packages

Prior to v5.27.0, the `osrm` Node package was unscoped. If you are upgrading from an old package, you will need to do the following:

```
npm uninstall osrm --save
npm install @project-osrm/osrm --save
```

#### Package docs

For usage details have a look [these API docs](docs/nodejs/api.md).

An exemplary implementation by a 3rd party with Docker and Node.js can be found [here](https://github.com/door2door-io/osrm-express-server-demo).

