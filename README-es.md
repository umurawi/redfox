<h1 align="center">REDFOX</h1>


## Qué es Redfox?
Es un script en Bash para Linux que gestiona un cortafuegos basado en nftables. Se hace mediante una corta lista de acciones disponibles: activación, desactivación, consulta del estado... etc.

No es exactamente un frontend para nftables, ya que no pretende gestionarlo añadiendo o quitando reglas. En su lugar se apoya en otros scripts ajenos en quienes delega la tarea de hacerlo.

A Redfox no le importa qué hacen estos scripts, lo único que espera es que añadan las reglas cuando los mande ejecutar, y así considerar que el cortafuegos ha sido activado.

Estos son exactamente dos scripts: uno para IPv4 y otro para IPv6. Y es que es una característica fundamental de Redfox separar completamente los protocolos IPv4 e IPv6, y poder así gestionarlos de forma independiente.


## Características
+ Exclusivo para Linux y nftables.
+ En esencia gestiona un cortafuegos, activándolo o desactivándolo.
+ Delega a otro script la tarea de añadir reglas. Es decir, activar el cortafuegos significa literalmente mandar ejecutar ese script (debe ser un archivo con permisos de lectura y ejecución).
+ A pesar de usar nftables, requiere separar las reglas de IPv4 e IPv6. Así que habrá un script para IPv4 y otro para IPv6.
+ Aunque las reglas se añadan una a una desde un script, se las ingenia para que sea de forma atómica.
+ No le importa cómo ni cuándo se le pedirá activar el cortafuegos, no le incumbe. Eso lo decidirá el usuario.
+ Unas pocas variables son necesarias para afinar su comportamiento.
+ Su código de salida es relevante. Así que este script puede ser usado a su vez por otros.


## La atomicidad es importante
La atomicidad es una característica importante en un cortafuegos. Las reglas que lo componen deben aplicarse todas de golpe, sin pausas entre ellas, o no hacerlo y fallar.

Por muy breve que sea el lapso de tiempo, no es admisible que unas reglas estén ya aplicadas mientras otras aún están por hacerlo. Tampoco lo es que el proceso falle en un punto, y que entonces solo se hayan aplicado un puñado de ellas pero no todas. En ambos casos el cortafuegos estará en una situación inconsistente y no hará bien su trabajo.

Redfox pedirá a otro script que aplique las reglas de nftables. Muy probablemente será un script en Bash que llamará secuencialmente al comando `nft`. Una vez por regla. Esto rompe la atomicidad. Para evitarlo Redfox utiliza un pequeño truco: Ejecutará el script de reglas dentro de un espacio de nombres de red que previamente habrá creado a propósito. Se espera que ahí se ejecute el comando `nft` secuencialmente, añadiendo las reglas una a una. En cuanto termine el proceso, se captura el conjunto de reglas de ese espacio de nombres y se aplica de forma atómica fuera de ahí. El único cometido del espacio de nombres es capturar y almacenar de forma temporal el conjunto de reglas, así que se crea y elimina cuando es necesario.

Redfox exige a este script que termine con un código de salida igual a `0` para indicar que las reglas se han aplicado correctamente. En caso contrario debe terminar con un código de salida diferente. Así se interpretará que el proceso ha fallado.


## Pilas no incluidas
Redfox viene sin pilas incluidas, es decir, no proporciona los scripts de reglas para IPv4 y IPv6. Se espera que el usuario sea capaz de crear sus propios scripts.

Hay en marcha el desarrollo de tales scripts que serían siempre opcionales. Ni tan siquiera necesariamente recomendados, solo sugeridos. De hecho, la intención es que incluso puedan ser usados de forma independiente.

Aún así, Redfox sí incluye dos conjuntos predefinidos de reglas para situaciones muy particulares: «bastion» y «lockdown». El primero bastiona la máquina de forma muy elemental, permitiendo el tráfico de salida y de reenvío pero no así el de entrada, el cual se filtra. El segundo directamente impide cualquier tráfico, el que sea, dejando la máquina totalmente aislada al instante.


## Parámetros
Puede usarse según dos parámetros. El primero (obligatorio) será la acción y controlará qué hacer. El segundo será el protocolo y determinará al cual de los dos aplicar la acción: _`ipv4`_ o _`ipv6`_. Como protocolo también se puede pasar los valores _`both`_ (valor por omisión) y _`none`_.

Acciones disponibles:
+ **`bastion`**: Activa el cortafuegos, pero lo hace con el conjunto predefinido de reglas de bastionado.
+ **`lockdown`**: Activa el cortafuegos, pero lo hace con el conjunto predefinido de reglas de aislamiento.
+ **`off`**: Desactiva el cortafuegos. Implica el borrado del conjunto de reglas actualmente en uso por nftables.
+ **`on`**: Activa el cortafuegos. Implica previamente el borrado del conjunto de reglas actualmente en uso por nftables.
+ **`reset`**: Reinicia el cortafuegos. En realidad es un alias encubierto de la acción _`on`_.
+ **`status`**: Muestra el estado del cortafuegos.


## Ejemplos de uso
Activamos el cortafuegos en ambos protocolos:
```
# redfox on
IPv4 on... OK
IPv6 on... FAIL
```

¡Oh! El script para IPv6 ha fallado, así que consultamos si realmente está desactivado para ese protocolo:
```
# redfox status
IPv4 status... ON
IPv6 status... OFF
```

Efectivamente, en IPv6 no tenemos el cortafuegos en marcha. Bueno, pues como mínimo lo bastionamos:

```
# redfox bastion ipv6
IPv6 bastion... OK
# redfox status ipv6
IPv6 status... BASTION
```

Ahora editamos el script de IPv6, vemos qué puede estar fallando y lo arreglamos:

```
# $EDITOR /foo/bar/quux/ipv6.sh
# redfox on ipv6
IPv6 on... OK
```

Ya funciona. Bueno, espera... ¿seguro?
```
# redfox status
IPv4 status... ON
IPv6 status... ON
```


## Variables de configuración
Las variables _`$DEBUG_STDERR_PATHF`_ y _`$DEBUG_STDOUT_PATHF`_ son rutas hacia archivos de log en donde se volcará la salida de error y la salida estándar. Solo tienen sentido en la depuración de este script, así que es preferible dejarlas vacías, lo que en la práctica implicará redirigir la salida hacia _`/dev/null`_.

Las variables _`$IPV4`_ e _`$IPV6`_ apuntan hacia los archivos de script que se ejecutarán para IPv4 e IPv6. Es imprescindible que las rutas sean absolutas. Se aceptan otros valores como _`bastion`_ o _`lockdown`_ para aplicar estas acciones en su lugar.  
Estas variables también se pueden dejar vacías para deshabilitaras, y así no ejecutar la acción _`on`_ al iniciar el cortafuegos.

La variable _`$NAMESPACE`_ determina el nombre del espacio de nombres en donde se aplicarán provisionalmente las reglas. Si se deja vacía, entonces el nombre se generará automáticamente según un patrón aleatorio (valor predeterminado y recomendable). Si la variable se omite, entonces no se usará el espació de nombres, y las reglas se aplicarán secuencialmente sin las ventajas de la atomicidad.


## Código de salida
El código de salida de este script puede variar dependiendo de si ha fallado o no y qué ha provocado el fallo:  
- **`0`** El script se ha ejecutado correctamente.
- **`1`** Ha ocurrido algún error indeterminado.
- **`4`** Ha ocurrido algún error relacionado con IPv4.
- **`6`** Ha ocurrido algún error relacionado con IPv6.
- **`10`** Ha ocurrido algún error relacionado tanto con IPv4 como con IPv6.

La acción _`status`_ también puede retornar estos códigos. Pero además de un error también podría indicar que el cortafuegos está desactivado para IPv4, IPv6 o ambos.