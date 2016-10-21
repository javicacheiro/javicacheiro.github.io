Cando se lanzou o servizo Big Data en novembro de 2013, as tecnoloxías Big Data eran unha área emerxente e o obxectivo que perseguimos naquel momento co servizo era que os usuarios do CESGA puideran tomar contacto coas tecnoloxías Big Data e co modelo de programación MapReduce.  Este servizo inicial permitía aos usuarios despregar de modo moi sinxelo clusters Hadoop sobre a plataforma cloud do centro.

Sen embargo co transcorrer do tempo, foron aparecendo problemas máis grandes e o servizo era insuficiente para a execución dalgúns problemas de elevado tamaño que foron aparecendo posteriormente. Para estes casos especiais despregábanse clusters Hadoop adicados no clúster SVG. Estes clusters permitían un rendemento maior que o obtido coa plataforma cloud pero tiñan a limitación de que a arquitectura dos nodos do SVG non é a máis axeitada para traballos Big Data.

Por iso durante o 2015 levouse a cabo unha análise dos requisitos hardware1 específicos que serían necesarios para unha solución Big Data baseada en Apache Hadoop, Apache Spark e Apache HBase. Baseándose nesta análise realizouse a adquisición dunha infraestructura específica para Big Data que conta coas seguinte características:






A lo largo de estos años hemos desplegado Hadoop en distintos entornos.

Empezamos con cloud.

Cuando no daba rendimiento suficiente el cloud pasamos al svg.

El el caso del svg el problema era principalmente el tiempo para conseguir los nodos y
redesplegar los servicios. Así como la falta de adecuacion del hardware a calculos
bigdata (nodos con solo 1 o 2 discos y 24 cores).

Posteriormente realizamos pruebas de una plataforma alternativa con un piloto de Hadoop
sobre Dockers en hardware adecuado para bigdata.

Se realizó un informe técnico de análisis de hardware para bigdata y se utilizó para
la elaboración del concurso de la plataforma bigdata.

Ahora tenemos 2 racks con 48 nodos.

