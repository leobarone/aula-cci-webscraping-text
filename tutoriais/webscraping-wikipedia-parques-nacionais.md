---
title: "Tutorial — Raspagem de dados na Wikipédia"
subtitle: "Parques nacionais do Brasil"
output: html_document
---

# Apresentação do problema

Vamos começar visitando a página da [Wikipédia que apresenta a lista de parques nacionais do Brasil](https://pt.wikipedia.org/wiki/Lista_de_parques_nacionais_do_Brasil). Abra o link no navegador e observe o conteúdo da página antes de continuar a atividade.

Há uma tabela com informações sobre os parques nacionais brasileiros. Nela encontramos os nomes dos parques e outras características, como localização, área e data de criação. Observe também que os nomes dos parques são *links*: se clicarmos sobre um deles, chegaremos a outra página, com um texto sobre aquele parque.

Nosso objetivo nesta atividade será aproveitar essa organização da Wikipédia para fazer duas coisas:

1. Capturar a tabela com as informações sobre os parques, tal como aparece na página da lista.
2. Capturar os links dos parques, visitar suas páginas automaticamente e guardar os textos em um *data frame*.

Para isso, vamos recuperar algumas funções que já conhecemos e aprender outras. Veremos como capturar páginas com `read_html()`, extrair tabelas com `html_table()`, utilizar a estrutura do HTML para encontrar links e parágrafos e, finalmente, construir um *for loop* para repetir a operação diversas vezes.

Vamos manter o código bastante simples. A ideia não é construir um programa sofisticado de coleta de dados, mas compreender, passo a passo, **como uma raspagem de dados funciona**.

# Pacotes no R

Algumas das funções que vamos utilizar nesta atividade não fazem parte das funções básicas do R. Precisamos, portanto, instalar dois pacotes: *rvest* e *tidyverse*.

O *rvest* contém as funções que permitem capturar páginas de internet e extrair informações de documentos HTML. O *tidyverse* é um conjunto de pacotes muito úteis para trabalhar com dados. Nesta atividade utilizaremos, entre outras, funções dos pacotes *dplyr*, *stringr* e *tibble*, que fazem parte do *tidyverse*.

A instalação de um pacote precisa ser feita apenas uma vez em cada computador. Caso ainda não estejam instalados, execute os comandos abaixo:

```{r instalar-pacotes, eval=FALSE}
install.packages("rvest")
install.packages("tidyverse")
```

Uma vez instalados, os pacotes não ficam automaticamente disponíveis para uso. Precisamos **chamá-los** a cada nova sessão de trabalho. Vamos fazer isso com a função `library()`:

```{r carregar-pacotes}
library(rvest)
library(tidyverse)
```

Excelente! Já temos as funções necessárias. **Obs.:** embora trabalhemos com os dois pacotes na mesma atividade, chamar o *tidyverse* não carrega automaticamente o *rvest*. Por isso usamos os dois comandos.

# Capturando uma página com `read_html()`

Nossa primeira tarefa consiste em armazenar o endereço da página que queremos capturar. Vamos criar um objeto chamado `url_lista_parques`:

```{r url-da-lista}
url_lista_parques <- 'https://pt.wikipedia.org/wiki/Lista_de_parques_nacionais_do_Brasil'
```

Note que, até agora, criamos apenas um objeto de texto. A URL é um endereço, mas guardar esse endereço em um objeto não significa que acessamos o conteúdo da página. É como anotar o endereço de uma biblioteca: sabemos onde ela fica, mas ainda não entramos para consultar os livros.

Para acessar a página e guardar o seu código HTML no R, utilizaremos a função `read_html()`, do pacote *rvest*:

```{r ler-pagina}
pagina_lista_parques <- read_html(url_lista_parques)
```

Vamos ler a instrução: utilizamos o texto guardado em `url_lista_parques` como argumento da função `read_html()` e armazenamos o resultado no objeto `pagina_lista_parques`.

O que há nesse objeto? Podemos utilizar a função `class()`, que nos informa a classe de um objeto no R:

```{r classe-da-pagina}
class(pagina_lista_parques)
```

O resultado identifica um documento interpretado pelo R como uma estrutura do tipo XML/HTML. Isso significa que não temos apenas o texto que enxergamos no navegador. Temos também a **estrutura da página**, com seus títulos, tabelas, parágrafos, links e outros elementos.

Essa diferença será importante logo mais. Primeiro, porém, vamos começar pelo caso mais fácil: a captura de uma tabela pronta.

# Extraindo a tabela com `html_table()`

A Wikipedia já organiza as informações dos parques em uma tabela. Em vez de copiar e colar os dados manualmente, podemos utilizar uma função do *rvest* que procura tabelas no documento HTML e tenta transformá-las objetos (data frames) em R: `html_table()`.

```{r extrair-tabelas}
tabelas_pagina <- pagina_lista_parques %>%
  html_table()
```

Note o símbolo `%>%`. Ele é conhecido como *pipe* e permite encadear funções. Podemos ler o código acima mais ou menos assim: **pegue o objeto `pagina_lista_parques` e passe-o para a função `html_table()`**. O resultado será guardado no objeto `tabelas_pagina`.

Vamos examinar o que obtivemos:

```{r examinar-lista-tabelas}
class(tabelas_pagina)
length(tabelas_pagina)
```

O resultado de `html_table()` é uma **lista**. Por quê? Porque uma página pode conter mais de uma tabela e cada tabela ocupará uma posição diferente dessa lista.

Vale lembrar uma particularidade das listas em R. Se usamos apenas um par de colchetes, como em `tabelas_pagina[1]`, obtemos uma **sublista** contendo o primeiro elemento. Se usamos dois pares de colchetes, como em `tabelas_pagina[[1]]`, extraímos o **próprio objeto** que está na primeira posição.

No nosso exemplo, queremos trabalhar com a tabela dos parques. Vamos extrair o primeiro elemento da lista:

```{r escolher-tabela}
tabelas_parques <- tabelas_pagina[[1]]
```

Agora podemos observar o resultado com as funções que já conhecemos:

```{r examinar-tabela}
class(tabelas_parques)
head(tabelas_parques)
```

Se estiver usando o RStudio, você também pode examinar a tabela em uma janela semelhante a uma planilha. Execute o comando abaixo no console ou simplesmente clique no nome do objeto na janela *Environment*:

```{r visualizar-tabela, eval=FALSE}
View(tabelas_parques)
```

Simples, não? Com poucas linhas de código capturamos a tabela apresentada na Wikipedia e a transformamos em um *data frame*. **Obs.:** a posição de uma tabela dentro de uma página não é uma propriedade permanente. Quando trabalhamos com outras páginas — ou quando o site muda sua estrutura — precisamos examinar a lista antes de escolher um elemento com `[[1]]`.

Mas ainda não terminamos. O desafio agora será usar os **links** que aparecem na tabela para visitar as páginas de cada parque e extrair seus textos.

# HTML: tags, conteúdos e atributos

Na primeira parte da atividade, não precisamos conhecer a estrutura HTML da página: `html_table()` fez quase todo o trabalho. O que fazer, entretanto, quando a informação que queremos não está organizada em uma tabela? Ou quando queremos extrair algo que não aparece nas células do *data frame*, como o endereço dos links?

Para responder a essas perguntas, precisamos conhecer alguns elementos da linguagem HTML.

HTML é uma linguagem de marcação utilizada para organizar o conteúdo de páginas na internet. Assim como acontece em documentos XML, seu conteúdo é identificado por **tags**, isto é, marcações que indicam que determinado trecho é um parágrafo, uma tabela, um link, uma imagem etc.

Abra novamente a página da Wikipédia. Posicione o mouse sobre o nome de um parque na tabela, clique com o botão direito e escolha **Inspecionar**. O navegador mostrará o trecho do código HTML correspondente àquele elemento. Você verá uma estrutura parecida com esta (o exemplo foi simplificado):

```html
<table>
  <tr>
    <th>
      <a href="https://pt.wikipedia.org/wiki/Parque_Nacional_de_Itatiaia" title="Parque Nacional de Itatiaia">Itatiaia</a>
    </th>
  </tr>
</table>
```

Vamos entender o que aparece nesse pequeno pedaço de HTML:

- `table` indica uma tabela.
- `tr` indica uma linha da tabela.
- `th` indica uma célula de cabeçalho da tabela.
- `a` indica um link, ou seja, um elemento clicável da página.

Observe que algumas tags estão **dentro** de outras tags. Uma `table` contém linhas (`tr`); uma linha contém células (`th`); e uma célula pode conter um link (`a`). Essa organização forma uma hierarquia.

Agora observe a última tag, `a`. Ela contém duas informações diferentes. A palavra `Itatiaia`, entre as tags de abertura e fechamento, é o **conteúdo** da tag: aquilo que enxergamos no navegador. Já `/wiki/Parque_Nacional_do_Itatiaia` é o valor de um **atributo**, chamado `href`, que indica para onde o link aponta.

Guarde bem essa diferença. Na raspagem de dados, às vezes queremos obter o **texto** de uma tag e, em outras situações, o valor de um **atributo** da tag. Utilizaremos as duas possibilidades nesta atividade.

# Caminhos no HTML: XPath

Como o documento HTML tem uma estrutura hierárquica, podemos construir caminhos para chegar exatamente aos elementos que nos interessam. Pense nos caminhos de pastas no seu computador: para encontrar um arquivo, podemos indicar em qual pasta ele está, dentro de qual outra pasta, e assim por diante.

No nosso pequeno exemplo, o caminho que leva aos links dentro das células da tabela passa por quatro tags:

```text
//table//tr/th//a
```

Esse caminho é escrito em uma linguagem chamada **XPath**. Vamos traduzi-lo:

- `//table`: procure uma tag `table` em qualquer parte do documento.
- `//tr`: dentro dela, procure linhas `tr`, mesmo que existam outras tags entre a tabela e as linhas.
- `/th`: escolha as células `th` que sejam filhas diretas dessas linhas.
- `//a`: dentro dessas células, procure as tags `a`, isto é, os links.

A diferença entre uma barra (`/`) e duas (`//`) é importante. Uma barra indica uma relação direta entre os elementos. Duas barras permitem procurar em qualquer nível inferior da hierarquia, sem que precisemos escrever todas as tags intermediárias.

Não precisamos decorar o nome de todas as tags HTML para fazer raspagem de dados. Precisamos, sobretudo, aprender a **inspecionar a página** e construir um caminho que identifique as informações que nos interessam. É por isso que a raspagem tem sempre um caráter um pouco "artesanal": páginas diferentes podem exigir caminhos diferentes.

Vamos agora usar esse caminho no R.

# Extraindo os links com `html_nodes()` e `html_attr()`

A função `html_nodes()`, do *rvest*, permite selecionar pedaços do documento HTML, também chamados de *nodes*. Vamos utilizá-la para procurar os links que aparecem dentro das células de cabeçalho das tabelas da página.

Depois, usaremos `html_attr()`, que extrai o valor de um atributo dos elementos selecionados. Como queremos os **endereços** dos links, e não os nomes dos parques escritos na página, o atributo que nos interessa é `href`.

```{r extrair-links}
url_parques <- pagina_lista_parques %>%
  html_nodes(xpath = '//table//tr/th//a') %>%
  html_attr('href')
```

Avance devagar neste ponto. Vamos ler as três linhas:

1. Começamos pelo objeto `pagina_lista_parques`, que contém o documento HTML completo da página.
2. Passamos esse documento para `html_nodes()`, com o caminho XPath que indica os links das tabelas.
3. Passamos os elementos encontrados para `html_attr('href')`, que devolve apenas o endereço guardado no atributo `href` de cada um deles.

O resultado **não é uma tabela**. É um *vetor de textos*, no qual cada elemento contém um endereço.

Vamos observar as primeiras informações capturadas:

```{r examinar-links}
head(url_parques)
length(url_parques)
```

Note que nem todo link de uma página necessariamente leva ao conteúdo que queremos. Por exemplo, uma tabela pode conter links para páginas de outros tipos. No nosso exemplo, as notas de rodapé, que estão ao final da página, tem um XPath semelhante ao dos links que queremos e acabam por vir juntos em nossa extração.

Neste caso, em vez de refinarmos nossa busca, vamos tratar o resultado. Uma maneira simples de selecionar apenas os endereços que seguem determinado padrão é usar `str_subset()`, função do pacote *stringr*.

As páginas de artigos da Wikipédia utilizam endereços que contêm `https://pt.wikipedia.org/wiki/`. Podemos manter apenas os links que contêm esse trecho:

```{r filtrar-links}
url_parques <- url_parques %>% 
  str_subset('https://pt.wikipedia.org/wiki/')

url_parques
```

Há, porém, mais links do queríamos. Isso ocorre pois links de uma outra tabela, que também têm o mesmo XPath, são juntados à nossa extração. Neste caso, a solução é simples: vamos limitar aos primeiros 76 elementos da lista. Para escolher posições de um vetor, usamos colchetes, como já fizemos com outros objetos no R:


```{r filtrar-links-2}
url_parques <- url_parques[1:76]

url_parques
```

Isso significa: **fique apenas com os elementos das posições 1, 2 3, ..., 76**. Pronto. Nossa lista de links está preparada:

```{r conferir-links}
head(url_parques)
length(url_parques)
```

# Capturando o texto de uma página

Já sabemos capturar uma tabela e extrair um vetor de links. Agora precisamos resolver outro problema: como extrair o **texto de uma página individual de um parque**?

Vamos começar com apenas uma página. Essa é uma boa estratégia em raspagem de dados: primeiro verificamos se sabemos capturar corretamente uma observação; depois repetimos o procedimento com quantas forem necessárias.

O primeiro endereço está na posição 1 do vetor `url_parques`. Podemos guardá-lo em um objeto chamado `url_p`:

```{r primeira-pagina-de-parque}
url_p <- url_parques[1]

pagina_parque <- read_html(url_p)
```

A função `read_html()` continua fazendo exatamente a mesma coisa que antes. A diferença é que agora ela recebe o endereço de um parque, em vez do endereço da página que contém a lista dos parques.

Abra essa página no navegador e observe como o texto está organizado. Em HTML, os parágrafos de texto são normalmente colocados dentro de tags chamadas `p`, abreviação de *paragraph*. Podemos, portanto, construir um caminho bastante simples para selecionar essas tags:

```text
//p
```

Leia esse caminho como: **encontre todas as tags `p` do documento, independentemente do ponto da hierarquia em que elas estejam**.

Usaremos `html_nodes()` para selecionar os parágrafos e `html_text()` para extrair seu conteúdo em texto:

```{r capturar-paragrafos}
texto <- pagina_parque %>%
  html_nodes(xpath = '//p') %>%
  html_text()
```

Compare com o código usado para os links. Nos dois casos começamos pelo documento HTML e utilizamos `html_nodes()` para selecionar elementos. Mas agora, em vez de extrair um atributo com `html_attr('href')`, extraímos o **conteúdo textual** das tags com `html_text()`.

Vamos examinar o resultado:

```{r examinar-paragrafos}
class(texto)
length(texto)
head(texto)
```

O resultado é um vetor. Cada elemento corresponde ao texto encontrado em um parágrafo. Se quisermos guardar o texto da página toda em uma única observação de um *data frame*, seria mais conveniente unir esses elementos em um só texto.

Podemos fazer isso com a função `paste0()` e o argumento `collapse`:

```{r juntar-paragrafos}
texto <- paste0(texto, collapse = ' ')
```

O argumento `collapse = ' '` informa que queremos juntar todos os elementos do vetor **sem acrescentar nenhum caractere entre eles**. Portanto, depois dessa instrução, `texto` passa a ser um vetor com apenas um elemento, que contém os parágrafos concatenados.

Excelente! Já sabemos como obter o texto de uma página. Agora só falta repetir esse processo para os demais parques.

# For loop: visitando várias páginas

*Loops* são processos iterativos, isto é, processos nos quais pedimos ao computador que repita determinadas instruções. Eles são especialmente úteis quando precisamos realizar a mesma tarefa muitas vezes.

Por exemplo, podemos imprimir os números de 1 a 5 com o seguinte código:

```{r exemplo-for}
for (i in 1:5) {
  print(i)
}
```

Podemos ler essa instrução assim: **para cada número `i` no conjunto que vai de 1 a 5, imprima o número `i`**. O que aparece entre as chaves é a instrução que será executada a cada passagem pelo loop.

Tente agora mudar o código para imprimir cada número multiplicado por 7. O resultado é a tabuada do 7! Não precisamos aprender novas funções; basta modificar a expressão dentro de `print()`.

Mas loops não servem apenas para repetir operações com números. Podemos também percorrer um **vetor de textos**, como o nosso vetor de URLs. Veja:

```{r exemplo-for-links}
for (url_p in url_parques) {
  print(url_p)
}
```

O que aconteceu? O R pegou o primeiro endereço de `url_parques`, chamou esse endereço de `url_p` e imprimiu seu conteúdo. Depois fez a mesma coisa com o segundo, terceiro, etc.

Agora imagine que, no lugar de apenas imprimir `url_p`, incluímos as instruções que acabamos de aprender: capturar a página com `read_html()`, selecionar as tags `p`, extrair o texto e juntar os parágrafos. É exatamente o que faremos a seguir.

# Armazenando os resultados em um *data frame*

Há um problema na nossa solução até aqui. Se colocarmos apenas as instruções de captura dentro do loop, o R conseguirá visitar as páginas, mas estaremos sobrescrevendo o objeto `texto` a cada nova passagem. Ao final, teremos apenas o texto da última página capturada.

Precisamos, portanto, de um lugar para **guardar o resultado de cada iteração**.

Para isso, vamos utilizar um *data frame*. Nosso objetivo é bastante simples: cada linha representará um parque visitado, com duas variáveis:

- `url_pagina_parque`: o endereço da página de onde retiramos o texto.
- `texto_parque`: o conteúdo dos parágrafos daquela página, reunido em um único texto.

Utilizaremos a função `tibble()`, do *tidyverse*, para criar cada nova linha. E utilizaremos `bind_rows()`, do pacote *dplyr*, para colocar uma linha debaixo da outra.

Antes de começar, vamos criar o objeto que guardará nossos dados. Por enquanto, ele estará vazio:

```{r iniciar-tabela}
tabela_textos <- NULL
```

Agora estamos prontos. Leia o código abaixo com calma. Ele combina praticamente tudo que aprendemos nesta atividade:

```{r capturar-todos-os-textos}
for (url_p in url_parques[1:76]) {

  print(url_p)

  pagina_parque <- read_html(url_p)

  texto <- pagina_parque %>%
    html_nodes(xpath = '//p') %>%
    html_text()

  texto <- paste0(texto, collapse = '')

  tabela_textos <- tabela_textos %>%
    bind_rows(
      tibble(
        url_pagina_parque = url_p,
        texto_parque = texto
      )
    )

}
```

Vamos traduzir o que fizemos:

1. **`for (url_p in url_parques)`**: para cada endereço do vetor `url_parques`, execute as instruções entre as chaves.
2. **`print(url_p)`**: mostre na tela o endereço que está sendo capturado. É uma maneira simples de acompanhar o andamento da atividade.
3. **`read_html(url_p)`**: acesse o endereço e guarde o documento HTML em `pagina_parque`.
4. **`html_nodes(xpath = '//p')`**: encontre os parágrafos do documento.
5. **`html_text()`**: extraia o conteúdo textual desses parágrafos.
6. **`paste0(texto, collapse = '')`**: reúna os textos encontrados em um único elemento.
7. **`tibble(...)`**: crie uma pequena tabela com duas colunas, uma contendo a URL e outra o texto do parque.
8. **`bind_rows(...)`**: acrescente essa tabela ao objeto `tabela_textos`, preservando o que já foi capturado nas passagens anteriores.

Ufa! Leia de novo. Parece bastante coisa, mas todas as partes do código já apareceram separadamente na atividade. A única novidade é que agora estamos usando as instruções **juntas**.

Note especialmente a última parte. Na primeira passagem, `tabela_textos` ainda está vazio (`NULL`), e `bind_rows()` coloca ali a primeira observação. Na segunda passagem, acrescenta uma nova linha. E assim por diante, até que todos os links do vetor tenham sido percorridos.

# Examinando os dados capturados

Pronto! Se tudo correu bem, teremos um objeto com 76 linhas, uma para cada página visitada, e duas colunas.

Podemos verificar a estrutura com `str()` e observar as primeiras e últimas linhas com `head()` e `tail()`, funções que já utilizamos em outras atividades:

```{r verificar-resultado}
str(tabela_textos)
nrow(tabela_textos)
head(tabela_textos)
tail(tabela_textos)
```

Como a segunda coluna contém textos longos, a melhor maneira de examinar o resultado, no RStudio, talvez seja abrir o objeto como uma planilha. Execute no console:

```{r visualizar-resultado, eval=FALSE}
View(tabela_textos)
```

Observe que **cada linha corresponde a uma página**, e não a um parágrafo. Isso ocorre porque usamos `paste0()` antes de criar a linha do *data frame*. Se não tivéssemos unido os parágrafos, seria necessário pensar em outra forma de organizar as observações.

Também é importante lembrar que `//p` captura **todas as tags de parágrafo encontradas**. Dependendo da estrutura de cada artigo, podemos obter textos além dos parágrafos principais. Para uma primeira atividade, isso é suficiente; em trabalhos mais complexos, poderíamos refinar o caminho XPath para selecionar apenas a região da página que nos interessa.

# Código completo

Vamos agora juntar o código da atividade, sem as explicações, para observar como ele é curto. O código é praticamente o mesmo com que começamos.

```{r codigo-completo, eval=FALSE}
library(rvest)
library(tidyverse)

url_lista_parques <- 'https://pt.wikipedia.org/wiki/Lista_de_parques_nacionais_do_Brasil'

pagina_lista_parques <- read_html(url_lista_parques)

tabelas_pagina <- pagina_lista_parques %>%
  html_table()

tabelas_parques <- tabelas_pagina[[1]]

url_parques <- pagina_lista_parques %>%
  html_nodes(xpath = '//table//tr/th//a') %>%
  html_attr('href')

url_parques <- url_parques %>% 
  str_subset('https://pt.wikipedia.org/wiki/')

url_parques <- url_parques[1:76]

tabela_textos <- NULL

for (url_p in url_parques) {

  print(url_p)

  pagina_parque <- read_html(url_p)

  texto <- pagina_parque %>%
    html_nodes(xpath = '//p') %>%
    html_text()

  texto <- paste0(texto, collapse = '')

  tabela_textos <- tabela_textos %>%
    bind_rows(
      tibble(
        url_pagina_parque = url_p,
        texto_parque = texto
      )
    )

}
```
