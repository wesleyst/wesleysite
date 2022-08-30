---
title: Diagramas Voronoi para negocios
author: Wesley Teixeira
date: '2021-01-25'
slug: diagramas-voronoi-para-negocios
categories:
  - Negócios
tags: []
subtitle: ''
summary: ''
authors: []
lastmod: '2021-01-25T15:12:58-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---




Uma das maravilhas da análise de dados é a possibilidade abrangente de análises que podem ser feitas para a tomada de decisão em um negócio. Entretanto, para isso, é essencial que o analista tenha a sua matéria prima, ou seja, os dados.

Em relação aos negócios, uma das bases de dados mais interessantes de serem analisadas é a de CNPJ. Nela, você encontra informações de todas as pessoas jurídicas do Brasil. Sabe-se portanto, o tamanho da força de trabalho da empresa, qual foi a sua data de criação, sua localização, ramo de atuação e etc.

Nessa publicação iremos focar na utilização dos dados relacionados às grandes empresas varejistas da cidade de São Paulo. Assim, vamos verificar onde essas lojas se localizam na cidade e analisar qual a área de dominância dessas lojas. Para isso iremos aplicar o algoritmo de Voronoi. Esse algoritmo busca identificar qual a área de dominância de determinado ponto em um gráfico ou mapa. Por exemplo, olhando a imagem abaixo, percebemos vários pontos com uma determinada cor ao redor. Essa área pintada representa a área de predominância do ponto. Isso quer dizer que, em qualquer parte dessa área, o ponto preto mais próximo dela é aquele que está dentro da área de mesma cor. Consequentemente, a área de fronteira entre as cores mostra a região que está equidistante entre dois ou mais pontos.


![dados](Voronoi.png) 


Agora imagine se aplicarmos esse conceito nas lojas de varejo? A partir disso saberíamos onde, por exemplo, as Lojas Americanas possuem determinada predominância em relação às Casas Bahia. Informações como essas poderiam, por exemplo, instigar os concorrentes a abrirem novas filiais na região de forma a concorrerem por aquele mercado. Também seria útil para descobrir se existe determinada loja que atende uma grande parte da cidade. Assim, pode ser interessante instalar novas filiais nessas regiões que possuem uma menor densidade de lojas de varejo.  

### Montando o modelo

Primeiramente temos que ler o banco de dados com as informações do CNPJ de todas as empresas do país. Os dados foram baixados no [site do governo](https://www.gov.br/receitafederal/pt-br/assuntos/orientacao-tributaria/cadastros/consultas/dados-publicos-cnpj). Para a leitura do banco, irei utilizar a biblioteca DBI e o RSQLite para a leitura de dados SQL e a bibliteca dbplyr para a aplicação de funções tidyverse em tabelas de banco de dados.


```r
library(tidyverse)
library(dbplyr)
library(DBI)
library(RSQLite)
```



```r
db <- dbConnect(SQLite(), "bd_dados_qsa_cnpj.db")
src_dbi(db)
data <- tbl(db, "cnpj_dados_cadastrais_pj")
```

A partir daí, selecionarei as colunas de interesse (principalmente relacionadas à localização do estabelecimento), faço uma filtragem dos municípios de interesse para a análise (no caso, a região metropolitana de São Paulo) e escrevo esses resultados em um arquivo csv.


```r
cidades <- c("SAO PAULO", "SAO BERNARDO DO CAMPO","DIADEMA",
             "SAO CAETANO DO SUL", "SANTO ANDRE", "MAUA",
             "RIBEIRAO PIRES", "RIO GRANDE DA SERRA", "ARUJA",
             "ITAQUACETUBA", "FERRAZ DE VASCONCELOS", "POA",
             "TABOAO DA SERRA", "BARUERI", "CARAPICUIBA",
             "EMBU", "ITAPECERICA DA SERRA"))

dados <- data %>% 
  select(razao_social, nome_fantasia, situacao_cadastral,
         descricao_tipo_logradouro:municipio, codigo_natureza_juridica,
         cnpj) %>%  
  filter(municipio %in% cidades) %>% collect()

write.csv(dados, "lojas.csv")
dbDisconnect(db)
```

Tendo as informações de todos os CNPJs de São Paulo, realizo então uma filtragem dos estabeleciomentos de varejo que tenho interesse em averiguar. Para isso, busquei listar o nome das principais lojas conhecidas no país: LOJAS AMERICANA, CASA BAHIA, MAGAZINE LUIZA, CASAS BAHIA, PONTO FRIO UTILIDADES S A. Também realizo algumas padronizações nos dados. Nem todas as lojas Magazine Luiza possuem apenas esse nome em sua razão social. Ás vezes podemos encontrar nomes como Loja Magazine Luiza Barueri. De forma a padronizar a nomeclatura deles, criei uma coluna com uma identidade única para cada rede de varejo.


```r
dados <- read_csv("lojas.csv")

lojas <- dados %>%
  filter(str_detect(razao_social, c("LOJAS AMERICANA|CASA BAHIA|MAGAZINE LUIZA|CASAS BAHIA|PONTO FRIO UTILIDADES S A"))) %>%
  mutate(tipo = case_when(
    str_detect(razao_social, "CASA BAHIA") ~ "CASA BAHIA",
    str_detect(razao_social, "LOJAS AMERICANAS") ~ "LOJAS AMERICANAS",
    str_detect(razao_social, "MAGAZINE LUIZA") ~ "MAGAZINE LUIZA",
    str_detect(razao_social, "PONTO FRIO") ~ "PONTO FRIO"
  ))
```


Por fim, criei uma coluna localização que vai unificar as colunas de endereço do estabelecimento. Isso foi feito com o intuito de aumentar a taxa de sucesso na geolocalização do estabelecimento.


```r
lojas <- lojas %>%
  mutate(endereco = str_c(descricao_tipo_logradouro, logradouro, sep = " ")) %>%
  mutate(localizacao = str_c(lojas$uf,
                             lojas$municipio,
                             lojas$bairro,
                             lojas$endereco,
                             lojas$razao_social,
                             sep = ", "))
```

Em seguida, utilizo a função de geolocalização do Google Cloud para a obtenção das latitudes e longitudes do estabelecimento. Essas informações serão armazenadas nas colunas lat e lon da minha tabela de dados.


```r
register_google(key = chave)
x <- geocode(lojas$localizacao)
lojas$lat <- x$lat
lojas$lon <- x$lon

write.csv(lojas,"lugar_lat_lon.csv")
```

Tendo os dados de latitude e longitude, ainda resta uma alteração a ser feita nos dados. É sabido que muitas das lojas de varejo estão localizadas em shopping centers. Sendo assim, se formos aplicar os algoritmo de voronoi cruamante nesses dados, veremos vários pontos justapostos em uma mesma região, o que não é relevante para a aplicação do diagrama de voronoi em nosso caso. Sendo assim, criei uma nova classificação para as minhas lojas. Aquelas que estivessem localizadas muito próximas umas das outras, foram caracterizadas como pertencentes a shopping centers e o domínio daquela região não é concedido a nenhuma rede de varejo.

Essa identificação de lojas muito próximas foi feita através do arredondamento da latitude e longitude desses estabelecimentos. Para aqueles estabelecimentos que apresentassem números de latitude e longitude iguais após o arredondadmente, foram classificados como "SHOPPING".


```r
lojas <- read.csv("lugar_lat_lon.csv")

dados <- lojas %>% 
  mutate(lat_r =  round(lat, 3), lon_r =  round(lon, 3)) %>% 
  group_by(lat_r, lon_r) %>%
  mutate(igualdade = ifelse(n()>1 & n_distinct(tipo)>1, "SHOPPING", "não shop")) %>%
  ungroup() %>% 
  select(tipo, lat, lon, lat_r, lon_r, igualdade) %>% 
  distinct_at(vars(lat, lon, tipo), .keep_all = TRUE) %>% 
  mutate(tipo = if_else(igualdade == "SHOPPING", igualdade, tipo)) %>% 
  distinct_at(vars(lat_r, lon_r, tipo), .keep_all = TRUE)
dados
```

```
## # A tibble: 322 x 6
##    tipo         lat   lon lat_r lon_r igualdade
##    <chr>      <dbl> <dbl> <dbl> <dbl> <chr>    
##  1 CASA BAHIA -23.6 -46.6 -23.6 -46.6 não shop 
##  2 CASA BAHIA -23.6 -46.6 -23.6 -46.6 não shop 
##  3 CASA BAHIA -23.7 -46.6 -23.7 -46.6 não shop 
##  4 CASA BAHIA -23.7 -46.6 -23.7 -46.6 não shop 
##  5 CASA BAHIA -23.6 -46.6 -23.6 -46.6 não shop 
##  6 CASA BAHIA -23.7 -46.5 -23.7 -46.5 não shop 
##  7 CASA BAHIA -23.7 -46.5 -23.7 -46.5 não shop 
##  8 CASA BAHIA -23.7 -46.5 -23.7 -46.5 não shop 
##  9 CASA BAHIA -23.6 -46.5 -23.6 -46.5 não shop 
## 10 CASA BAHIA -23.7 -46.5 -23.7 -46.5 não shop 
## # ... with 312 more rows
```


Mudandças realizadas, vamos agora preparar o mapa para a visualização dos dados. Para isso é importante que você identifique quais os limites de latitude e longitude que você deseja que o seu mapa abranja. Uma método de se fazer isso é identificando a latitude e longitude máximas e mínimas dos seus dados e adicionando um pequeno valor numérico a esses resultados para que você tenha uma "borda".

Bem, as coordenadas que me pareceram visualmente agradáveis foram:


```r
sac_borders <- c(bottom  = -23.816784, 
                 top     = -23.340841,
                 left    = -46.991035,
                 right   = -46.240358)
```


Agora realizo a obtenção do mapa utilizando a função get_stamenmap(). O Stamen Design é uma ferramenta de uso gratuito e que utiliza o OpenStreetMap para a geração de mapas visualmente bonitos. Além disso, vou contruir uma paleta de cores para a visualização do resultado do algoritmo de voronoi.

Confesso que esse processo final é demorado e cheio de tentativas e erros. Entretanto, é imprescindível fazer com que as informações sejam visualmente esclarecedoras e agradáveis de serem vistas


```r
library(ggmap)
map <- get_stamenmap(sac_borders, zoom = 12, maptype = "toner-lite", color = "color")
cols <- c("MAGAZINE LUIZA" = "#211E6C",
          "CASA BAHIA" = "#E7A0A0",
          "LOJAS AMERICANAS" = "#FF9CFF",
          "PONTO FRIO" = "#FFE400",
          "SHOPPING" = "#000000")
```

Por fim, realizo a construção do mapa. Para a visualização do resultado do algoritmo de Voronoi, estou utilizando a função geom_voronoi da bibliteca ggvoronoi. Como ela possui interação com a biblioteca ggmap, é possível fazer várias customizações do gráfico:


```r
library(deldir)
library(ggvoronoi)
library(extrafont)
loadfonts(device = "win")

ggmap(map) +
  geom_voronoi(aes(x = lon, y = lat, fill = tipo), alpha = 0.3, data = dados)+
  scale_fill_manual(values = cols)+
  geom_point(data = dados, aes(x = lon, y = lat), size=2 )+
  stat_voronoi(aes(x = lon, y = lat), data = dados, geom = "path", color = "black")+
  labs(title = "Lojas de varejo em São Paulo\n")+
  theme(plot.title = element_text(hjust = 0.5, size = 20, color = "black"),
        plot.subtitle = element_text(hjust = 0.5, size = 16, color = "white"),
        axis.ticks = element_blank(),
        axis.text = element_blank(),
        axis.title = element_blank(),
        legend.position = "bottom")
```

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-11-1.png" width="672" />

Analisando o mapa gerado, é interessante perceber certa predominânica de algumas lojas em determinadas regiões de São Paulo. Pode-se perceber, por exemplo, certa dominânica das Casas Bahia na região sul e nordeste da cidade. Também é percebido que as Lojas Americanas se concentram mais na região oeste da cidade.

### Conclusão

É impressionante a quantidade de dados que estão disponíveis para qualquer um utilizar e tomar suas decisões a partir deles. Apesar da máxima de que os dados são o novo ouro do mercado, isso não quer dizer que o preço deles também é alto. Na verdade, existe muitas informações interessantes disponibilizadas gratuitamente pelo próprio governo e que podem ser de grande valia para tomadas de decisões empresariais.
