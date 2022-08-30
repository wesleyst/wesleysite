---
title: Paulo foi o autor de Hebreus?
author: Wesley Teixeira
date: '2022-08-29'
slug: paulo-foi-o-autor-de-hebreus
categories: []
tags: []
subtitle: ''
summary: 'Uma análise com redes neurais'
authors: []
lastmod: '2022-08-29T22:48:04-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


A carta aos Hebreus na Bíblia é sigular. Basta uma leitura superficial e se percebe que ela é diferente de todas as outras cartas do novo testamento. Muitos então se perguntam quem foi o autor de Hebreus? Um dos principais candidatos à autoria é o apóstolo Paulo. Apesar da excentricidade da carta, é possível ver algumas similaridades com outras cartas, principalmente quando lemos a sua conclusão.

Para além de toda discussão teológica, tentarei verificar nesse post se a Inteligência Artificial pode nos ajudar de alguma maneira. Sei que a discussão sobre o assunto é muito mais complexa do que rodar um algoritmo de computador. Existem diversas outras formas mais confiáveis de se verificar a autoria de um texto. Então não encare com muita seriedade a análise que estarei fazendo.

Basicamente o que farei é treinar o computador para identificar quais textos da Bíblia pertecem a Paulo ou não. Então o computados vai "ler" os textos que são do Novo Testamento e verificar esses padrões. Depois disso vou testar o modelo em uma amostra aleatória de textos e ver qual o percentual de acerto que o computador teve. Por fim, vamos verificar qual o resultado final que o computador fornece para o livro de Hebreus. 

Bem, o processo que vou seguir é bastante simples:
1. Vou importar a bíblia para a memória do computador
2. Tratar a base de dados de forma a deixá-la pronta para rodar o modelo
3. Criar o modelo de predição e testá-lo
4. Aplicar o modelo no livro de Hebreus e verificar qual a decisão final do computador sobre a autoria paulina

### Obtenção dos dados

A primeira coisa a ser feita é a importação dos dados. Como queremos que o computador tenha os melhores dados para ser treinado, vamos utilizar como insumo o Novo Testamento escrito em grego: 


```r
library(tidyverse)
biblia_nt <- read_csv("00-new-testament.csv", locale = locale(encoding = "UTF-8"))
#biblia_nt <- biblia_nt %>% filter(!(book %in% c("matthew", "mark", "luke", "acts-of-the-apostles", "revelation", "john")))
```
![grego](livro.png)

Agora seguiremos uma série de modificações nos dados para colocá-lo no melhor formato para a aplicação do modelo. Como pode ser visto, cada versículo possui uma quantidade diferente de palavras. Isso não é bom para o modelo que vamos aplicar aqui. Precisamos fazer com que cada trecho da Bíblia precise ter a mesma quantidade de palavras. Então serão essas transformações que faremos agora:


```r
# Os versículos são transformados em uma lista de palavras
dados_palavra <- biblia_nt %>% mutate(text = str_split(text, " ")) %>%
  # Cria-se uma linha para cada palavra acompanhada pela identificação
  # do livro, capítulo e verso em que se encontra
  unnest(cols = c("text"))

# Defini-se a quantidade de palavras para cada trecho
len <- 10

dados <- dados_palavra %>%
  # Agrupa os dados por livro
  group_by(book) %>% 
  # Classifica as palavras por grupo de 25 palavras
  mutate(linha = (row_number() - 1)%/%len) %>%
  # Agrupa os dados por livro e grupo de 25 palavras
  group_by(book, linha) %>%
  # Cria-se uma coluna com a junção das palavras de um um mesmo grupo
  mutate(string = paste0(text, collapse = " "),
         # Cria-se um contador de palavras por grupo
         n = n(),
         # Identifica a origem das palavras: Qual o versículo da primeira
         # palavra do grupo e o versículo da última palavra do grupo
         localizacao = str_c("Partida_", first(chapter), "_", first(verse),"_Chegada_", last(chapter), "_", last(verse))) %>%
  # Exclui-se colunas desnecessárias
  select(-chapter, -verse, -text) %>%
  # Exclui-se linhas repetidas
  unique() %>% 
  # Seleciona-se apenas grupos com 25 palavras
  filter(n==len) %>%
  # Desagrupa a base de dados
  ungroup()
```
Pronto. Nossos dados estão agora assim:



Não mudou muita coisa, mas agora temos cada livro dividido por grupos de 25 palavras. Vamos então identificar cada grupo por seu autor. Assim, se o trecho foi escrito por Paulo, vamos dar a ele a idetificação do número 1. Se ele não foi escrito por ele, receberá a identificação do número 0.

É consenso que os livros escritos por Paulo foram: Romanos, 1 e 2 Coríntios, Gálatas, Efésios, Filipenses, Colossenses, 1 e 2 Tessalonissenses, 1 e 2 Timóteo, Tito e Filemom. Então vamos criar uma lista deles e realizar a identificação dos grupos:


```r
paul <- c("romans", "1-corinthians", "2-corinthians", "philemon", "galatians", 
          "philippians", "1-thessalonians", "2-thessalonians", "ephesians",
          "colossians", "1-timothy", "2-timothy", "titus")
```

Deve-se lembrar também que antes de realizarmos a identificação, é importante também separar o livro de Hebreus da base de dados. Afinal, é ele que estamos tentando prever a autoria:



```r
# Armazenando o livro de Hebreus
hebr <- dados %>% filter(book == "hebrews")

dados <- dados %>%
  # Excluindo o livro de Hebreus da base
  filter(book != "hebrews") %>%
  # Identificando trechos escritos por Paulo
  mutate(paul = if_else(book %in% paul, 1, 0), id = row_number())
```

Identificados os trechos, devemos agora separar uma amostra para o treinamento e a testagem do modelo. Esse é um passo importante em toda costrução de um modelo de previsão e/ou classificação. Quando separamos uma amostra de treinamento e outra de teste, asseguramos que o nosso modelo treinado tem, de fato, uma boa taxa de acerto.

Geralmente é separado 30% da amostra para servir como dados de teste e 70% da amostra para servir de treinamento:


```r
# Criação dos dados de teste
n_hebr_test <- dados %>% sample_frac(0.30)
# Criação dos dados de treinamento
n_hebr <- anti_join(dados, n_hebr_test, by = "id")
```

### Criação do modelo de previsão

Partimos então para a aplicação do modelo de redes neurais. Essa etapa é constituída de uma série de detalhes dos quais não entrarei muito. A princípio, teremos que tokenizar as palavras. Cada palavra será identificada por um número e esse número passará a substituir as palavras em grego.


```r
library(keras)

# Criando estrutura de tokenização
tokenizer <- text_tokenizer() %>%
  # Identificando dataframe para tokenizar
  fit_text_tokenizer(n_hebr$string)

# Tokenizando data frame
sequences <- texts_to_sequences(tokenizer, n_hebr$string)

# Dicionário de tokenização (palavra, token)
word_index = tokenizer$word_index

# Criando dataframe com o número máximo de palavras aceitas (maxlen)
data <- pad_sequences(sequences)

# Criando um vetor de identificação da autoria paulina
labels <- as.array(n_hebr$paul)
```

Os dados resultantes dessas tranformações são uma matriz onde cada linha representa um trecho de 25 palavras e cujas colunas representa uma palavra codificada.


```r
data[1:5,]
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10]
## [1,] 5663 2509   52   65  326  227  326  210  210   327
## [2,]    8  751  751    4  327    8  557  557    4   327
## [3,]    8 1393    1   26  352   10  580    4  327     8
## [4,]    8 2510 2510    4  327    8 3485   29   12  5664
## [5,] 3485    4  327    8 3486   29   12 5665 3486     4
```

Como se pode ver, algumas linhas da matriz acima são muito parecidas. Palavras cujo código são 4, 9 e 375 parecem se repetir constantemente. Um dos motivos para isso é que estamos olhando para o início do livro de Mateus. Se dermos uma olhada, esse livro começa contando a genealogia de Jesus e vemos que, de fato, existem muitas palavras repetidas nela. 

O próximo passo é a seleção da amostra para o treinamento do modelo. Para isso, é necessario ainda dividir os dados que temos entre um grupo de treinamento e outro de validação. Assim, utilizamos o grupo de treinamento para que a máquina aprenda os padrões de escrita de Paulo. Já o grupo de validação servirá para vermos o desempenho apresentado pelo computador e realizarmos algumas modificações nos parâmetros da rede neural. Essas etapas seguem o seguinte roteiro:


```r
# Criação de um vetor de índices "embaralhados". Queremos que os trechos de paulo
# estejam misturados com os de outros autores
indices <- sample(1:nrow(data))

# Selecionando os índices aleatórios de treinamento
training_indices <- indices[1:as.integer(length(indices)*0.7)]

# Selecionando os índices de validação (excluindo-se os índices de treino)
validation_indices <- indices[(as.integer(length(indices)*0.7) + 1):length(indices)]

# Criando variáveis para o modelo

# Criação da matriz de grupos de palavras para treino
x_train <- data[training_indices,]
# Respectivo vetor de classificação das linhas da matriz (escrito paulino ou não)
y_train <- labels[training_indices]

# Criação da matriz de grupos de palavras para validação
x_val <- data[validation_indices,]
# Respectivo vetor de classificação das linhas da matriz (escrito paulino ou não)
y_val <- labels[validation_indices]
```



### Treinamento do modelo

Finalmente chegamos na parte de treinarmos o modelo e a identificação dos melhores parâmetros para o modelo. Aqui não entrarei em muitos detalhes no que estarei fazendo (não é meu intuito ensinar redes neurais, até porque ainda estou aprendendo a mexer com elas), mas deixo um [link](https://www.manning.com/books/deep-learning-with-r-second-edition) do livro que tenho utilizado para aprender um pouco sobre esses parâmetros.


```r
embedding_dim <- 100

model <- keras_model_sequential() %>%
  layer_embedding(output_dim = embedding_dim,
                  input_dim = length(word_index) + 1,
                  input_length = len) %>%
  layer_flatten() %>%
  layer_dense(units = 32, activation = "relu") %>%
  layer_dense(units = 1, activation = "sigmoid")
summary(model)
```

```
## Model: "sequential"
## ________________________________________________________________________________
## Layer (type)                        Output Shape                    Param #     
## ================================================================================
## embedding (Embedding)               (None, 10, 100)                 1351700     
## ________________________________________________________________________________
## flatten (Flatten)                   (None, 1000)                    0           
## ________________________________________________________________________________
## dense_1 (Dense)                     (None, 32)                      32032       
## ________________________________________________________________________________
## dense (Dense)                       (None, 1)                       33          
## ================================================================================
## Total params: 1,383,765
## Trainable params: 1,383,765
## Non-trainable params: 0
## ________________________________________________________________________________
```

Aqui definimos os parâmetros que vamos utilizar para valorar o desenpenho do nosso modelo:


```r
model %>% compile(
  optimizer = "rmsprop",
  loss = "binary_crossentropy",
  metrics = c("acc"))
```

Agora verificamos qual foi o desempenho do modelo de acordo com os parâmetros escolhidos


```r
history <- model %>% fit(
  x_train, y_train,
  epochs = 20,
  batch_size = 32,
  validation_data = list(x_val, y_val))
plot(history)
```

```
## `geom_smooth()` using formula 'y ~ x'
```

<img src="{{< blogdown/postref >}}index.en_files/figure-html/unnamed-chunk-11-1.png" width="672" />

Como podemos ver, a taxa de acerto do computador começa a se estabilizar na quarta época. Sendo assim, nós vamos limitar o treino do nosso modelo para três épocas para que não tenhamos um overfitting, ou seja, para que o modelo não seja apenas adaptado para os dados de treinamento.


```r
embedding_dim <- 100

model <- keras_model_sequential() %>%
  layer_embedding(output_dim = embedding_dim,
                  input_dim = length(word_index) + 1,
                  input_length = len) %>%
  layer_flatten() %>%
  layer_dense(units = 32, activation = "relu") %>%
  layer_dense(units = 1, activation = "sigmoid")

model %>% compile(
  optimizer = "rmsprop",
  loss = "binary_crossentropy",
  metrics = c("acc"))

model %>% fit(
  x_train, y_train,
  epochs = 2,
  batch_size = 32,
  validation_data = list(x_val, y_val))
```


Antes de aplicar o modelo treinado na variável teste, vou tokenizar os meus dados de teste e criar um vetor indicando se a passagem foi escrita por Paulo ou não.


```r
sequences <- texts_to_sequences(tokenizer, n_hebr_test$string)
x_test <- pad_sequences(sequences)
y_test <- as.array(n_hebr_test$paul)
```

Agora vejo o desempenho apresentado pelo modelo na variável teste


```r
resultado <- model %>% evaluate(x_test, y_test)
```

Como pode ser visto, o modelo conseguiu acertar a autoria de 86.2% dos trechos paulinos. Creio que seja um valor consideravelmente satisfatória visto que a proporção de trechos escritos por Paulo na nossa variável de teste é de 0.228838582677165%


```r
table(y_test)
```

```
## y_test
##    0    1 
## 3134  930
```
Observando de uma forma mais detalhada, podemos ver quantos trechos de cada livro foram classificados de forma correta ou não:


```r
final <- tibble(livro = n_hebr_test$book, real = y_test, pred = model %>% predict_classes(x_test), paul = n_hebr_test$paul) %>%
  mutate(resul = if_else(real == pred[,1], "1", "0"))
final_comp <- final %>% group_by(livro, resul) %>% summarise(quant = n()) %>% 
  pivot_wider(names_from = resul, values_from = quant) %>% 
  rename(erro = `0`, acerto = `1`) %>% 
  mutate(percentual_acerto = round(acerto*100/(erro+acerto), 1))
```

```
## `summarise()` has grouped output by 'livro'. You can override using the `.groups` argument.
```

```r
final_comp %>% print(n= 26)
```

```
## # A tibble: 26 x 4
## # Groups:   livro [26]
##    livro                 erro acerto percentual_acerto
##    <chr>                <int>  <int>             <dbl>
##  1 1-corinthians           72    124              63.3
##  2 1-john                   9     47              83.9
##  3 1-peter                 32     14              30.4
##  4 1-thessalonians         10     38              79.2
##  5 1-timothy               13     24              64.9
##  6 2-corinthians           33    100              75.2
##  7 2-john                   6      2              25  
##  8 2-peter                  9     23              71.9
##  9 2-thessalonians         11     13              54.2
## 10 2-timothy               22     15              40.5
## 11 3-john                   1      3              75  
## 12 acts-of-the-apostles    53    534              91  
## 13 colossians              17     34              66.7
## 14 ephesians               22     43              66.2
## 15 galatians               20     36              64.3
## 16 james                   17     30              63.8
## 17 john                    27    448              94.3
## 18 jude                     9     10              52.6
## 19 luke                    32    607              95  
## 20 mark                     7    357              98.1
## 21 matthew                 25    549              95.6
## 22 philemon                 4      4              50  
## 23 philippians             19     30              61.2
## 24 revelation              11    272              96.1
## 25 romans                  71    137              65.9
## 26 titus                    7     11              61.1
```

Percebe-se que o modelo apresentou uma boa previsão para livros como João e Apocalipse. Isso acontece porque esses livros possuem uma maior quantidade de versos e, consequentemente, mais trechos para o treinamento. Outro motivo é porque sabemos que o estilo de escrita desses livros se diverge do estilo utilizado nas cartas do novo testamento.


### Autoria da carta aos Hebreus

Relizados todos os ajustes e testes, finalmente aplicaremos a previsão do modelo para a carta aos Hebreus:


```r
sequences <- texts_to_sequences(tokenizer, hebr$string)
x_hebreus <- pad_sequences(sequences)


final <- tibble(livro = hebr$book, real = 1, pred = model %>% predict_classes(x_hebreus), prob = model %>% predict(x_hebreus), paul = 1, localizacao = hebr$localizacao) %>%
  mutate(resul = if_else(real == pred[,1], "1", "0"))
final_comp <- final %>% group_by(livro, resul) %>% summarise(quant = n())
final_comp
```

```
## # A tibble: 2 x 3
## # Groups:   livro [1]
##   livro   resul quant
##   <chr>   <chr> <int>
## 1 hebrews 0       316
## 2 hebrews 1       184
```
Como pode ser visto, o modelo previu que grande parte do livro não foi escrita por Paulo. Assim, somente 35,8% das passagens teriam sido escritas pelo apóstolo.

Talvez seja interessante identificar quais passagens o modelo apontou como criadas por Paulo. Aqui mostro algumas delas.

É interessante pensar que o final de Hebreus foi classificado como uma escrita paulina. Isso realmente faz sentido pois a conclusão da carta realmente se assemelha às saudações paulinas. Reforça esse ponto o fato do final da carta citar também o nome de Timóteo (um de seus maiores ajudantes e citado algumas vezes em outras cartas).

O trecho que recebeu a maior probabilidade de pertencer a Paulo se encontra no início de Hebreus 2:10 que diz:

> Porque convinha que aquele, por cuja causa e por quem todas as coisas existem

De fato, esses trecho lembra muito o estilo de escrita de Paulo. Pode-se citar, por exemplo, o texto de Colossensses 1:16-17:

> Pois, nele, foram criadas todas as coisas, nos céus e sobre a terra, as visíveis e as invisíveis, sejam tronos, sejam soberanias, quer principados, quer potestades. Tudo foi criado por meio dele e para ele. Ele é antes de todas as coisas. Nele, tudo subsiste.

Ou o trecho de 1 Coríntios 8:6

> todavia, para nós há um só Deus, o Pai, de quem são todas as coisas e para quem existimos; e um só Senhor, Jesus Cristo, pelo qual são todas as coisas, e nós também, por ele.

### Conclusão

O modelo previu que apenas 35,8% da carta foi escrita por Paulo. Confesso que esperava um percentual maior por ser particularmente um defensor da autoria paulina da carta. Contudo, compreendo que muitas nuances do escrito não são facilmente percebidas pelo modelo. De toda forma, achei bem curioso que alguns trechos apontados como de autoria paulina, de fato, se assemelham com outros textos escritos pelo apóstolo.
