---
title: Relatorio de Desempenho Escolar no ENEM
author: Wesley Teixeira
date: '2021-01-10'
slug: relatorio-enem
categories: []
tags: ["Educação"]
subtitle: ''
summary: 'Uma ferramenta de apoio às escolas do ensino médio'
authors: []
lastmod: '2021-01-10T14:40:05-03:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Quando trabalhei na área da educação, uma das coisas que mais me chamou a atenção foi a quantidade imensa de dados disponíveis para análises, mas que eram pouco utilizados. Um claro exemplo disso ocorre com o ENEM. Temos milhões de pessoas que o fazem anualmente e que fornecem variadas informações que podem ser muito úteis para um gestor escolar.

Dentre todas essas informações, creio que uma das mais importantes é o desempenho dos seus alunos. Parte desses dados já são analisados. Em determinada época do ano é comum ver vários meios de comunicação divulgando informações do desempenho das escolas e quais foram suas notas médias em determinada área. Isso geralmente acontece quando o Inep divulga o microdados do ENEM e permite que qualquer um baixe os dados e faça análises sobre ele.

Enfim, ultimamente estava buscando alguma forma de possibilitar que as escolas tenham um acesso mais detalhado aos seus resultados escolares. É muito interessante que elas saibam suas notas médias em cada grande área, mas imagine se elas soubessem qual o desempenho em determinados assuntos como geometria plana ou história do Brasil? Ou melhor, e se elas pudessem saber qual o percentual de acerto que a escola obteve em cada questão da prova do ENEM?

Nesse intuito, desenvolvi um painel que possibilita cada escola fazer justamente isso. Nele você pode selecionar a escola (esquerda do painel) e baixar um relatório agregado de desempenho (desempenho por assunto) e um relatório desagregado de desempenho (desempenho por questão) [canto superior direito do painel]. Também é possível fazer o download das provas caso queiram saber quais são as questões de cada prova e a classificação delas.

O painel foi elaborado com a linguagem R e utilizando o pacote Shiny. Ele está hospedado em um servidor gratuito, mas com uma cota limitada de horas. Isso quer dizer que dependendo da quantidade de acessos, pode ser que o painel fique indisponível por um tempo determinado. Caso isso aconteça e você queira ver o relatório de alguma escola, pode entrar em contato comigo que eu mando o relatório no privado.

[Link do Painel](https://bit.ly/enemescolas)

Algumas imagens do painel:

![Painel Principal](painel.jpg)
![Primeira págia do relatório](relatorio1.jpg)

>Observações:
>1. Os dados são do ENEM aplicado em 2019.
>2. Para o cálculo do desempenho da escola, selecionei apenas os resultados dos alunos que iriam completar o ensino médio no ano corrente da aplicação.
>3. Talvez alguns alunos não marquem a escola que estudam no momento da inscrição. Esses alunos não são contabilizados na análise.
>4. Na elaboração dos ranques de desempenho, só são contabilizadas as escolas que possuíam 10 ou mais alunos inscritos no ENEM.
#
#
## Versão do PowerBI

Em 2019 eu também desenvolvi uma outra ferramenta de análise do desempenho das escolas no ENEM. Ela foi elaborada através do PowerBI e você pode verificar o painel através do link:

[Painel ENEM 2017](https://app.powerbi.com/view?r=eyJrIjoiMGI2ZGQ4MDktODEyNC00MmU5LWE3ZTYtYWZmZDZhMDBjODE4IiwidCI6ImQ0M2VkZjRjLTEzYjUtNDYyNy05YTI1LWNmZjE3MjI0N2YzZCJ9)

![Painel PowerBI](powerbi.png)
