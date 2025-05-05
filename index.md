---
title: Automatización de Azure Load Testing con Acciones de GitHub
permalink: index.html
layout: home
---

Los siguientes ejercicios se han diseñado para brindarte una experiencia práctica de aprendizaje en la que implementarás flujos de trabajo y acciones de GitHub para automatizar la realización de una prueba de carga con Azure Load Testing. 

## Ejercicios
<hr/>


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
* [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }} {% endfor %}
