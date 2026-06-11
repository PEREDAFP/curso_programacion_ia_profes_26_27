# Sesión 1

Vamos a crear una aplicación de recogida de datos meteorológicos ubicada en un servidor web no local.


### 📚Prompt para utilizar github.com como mi servidor web
Quiero crear una página en github.com para alojar una página web html con javascript ¿Cómo lo hago? Ya tengo cuenta en github.com ahórrate ese paso
[Respuesta](pages_github.md)

### 📚Prompt para crear la aplicación Estación Meteorológica
Crea una aplicación html+javascript que contenga, en la parte superior, un formulario en el que introducir datos: localidad, fecha, situación climatológica y temperatura. 

En la parte inferior, aparecerá un listado con los registros añadidos previamente y que podrán eliminarse o modificarse (esto en el mismo formulario de la parte superior) 

La idea es guardar los datos en una base de datos SQLite y subir todo el proyecto a una página github

[Respuesta](inicio.md)

### 📚Prompt Queremos cambiar de LocalStorage, que no nos permite mantener los datos entre sesiones y navegadores,a  una base de datos

Buen intento, pero estás utilizando el LocalStorage. Quiero que los datos persistan entre sesiones y que puedan ser accedidos desde diferentes máquinas y navegadores. Por eso la base de datos ¿Es posible hacerlo en github? Si es que no. Dame alternativas. Ah, y los datos de los desplegables también deben obtenerse de la base de datos
[Respuesta](cambio_localStorage_base_datos.md)

### 📚Prompt Solicitamos ayuda para crear la base de datos en Supabase

Dime cómo hacer que sea SupaBase el motor de base de datos
[Respuesta](supabase.md)

### 📚Prompt Aparece un error en la consola del navegador y solicitamos solución

Aparece el siguiente error: Uncaught SyntaxError: redeclaration of non-configurable global property supabase
    <anonymous> http://127.0.0.1:5500/index2.html:226
[Respuesta](solucion_error_objeto_supabase.md)

### 📚Prompt solicitando modificar la aplicación para que cambios en la base de datos se obtengan en el listado en tiempo real


Perfecto. Pero se puede hacer mejor ¿Cómo podemos hacer que cuando se modifique o elimine un registro por parte de otra conexión se refleje en nuestra pantalla?
[Respuesta](refresco_cambios_registro_tiemporeal.md)


### 📚Prompt de error tras solicitar que los cambios en la base de datos desde otros navegadores se refresquen


Aparece este mensaje en la consola La cookie “__cf_bm” ha sido rechazada por un dominio no válido. y la lista de registros no se actualiza hasta que se refresca la pantalla
[Respuesta](Solucion_error_sincronizacion.md)


### 📚Prompt para obligar a utilizar usuario y contraseña
Solucionado. Ahora me gustaría habilitar control de acceso solo usuarios autorizados deben acceder a la aplicación
[Respuesta](Creando_login.md)

### 📚Prompt para que los usuarios no se den de alta ellos solos, sino que podamos administrarlo

Pero cualquiera puede autenticarse. ¿Puede haber un paso previo para que un usuario administrador valide a ese usuario?
[Respuesta](Administrando_la_seguridad.md)

### Un posible resultado:
[Estacion_meteorológica](https://peredafp.github.io/estacion_meteorologica/)

### Actividades:
  - Utiliza Firebase como repositorio. Recuerda que es la IA la que debe darte todas las respuestas de código.
  - Habrás observado que el código html es "infinito". Pregunta por CSS y haz que el código html, el código JavaScript y el CSS se encuentren en ficheros separados.
  - Personaliza, mediante CSS, tu aplicación.
  - Añade un campo PROVINCIA que muestre un desplegable con las provincias que creas oportuno. Estos datos deben encontrarse en una tabla aparte, similar a la tabla **situaciones**