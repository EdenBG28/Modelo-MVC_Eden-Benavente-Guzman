
# Modelo vista controlador (MVC)

Modelo Vista Controlador (MVC) es un estilo de arquitectura de software que separa los datos de una aplicación, la interfaz de usuario, y la lógica de control en tres componentes distintos.

Se trata de un modelo muy maduro y que ha demostrado su validez a lo largo de los años en todo tipo de aplicaciones, y sobre multitud de lenguajes y plataformas de desarrollo.

El modelo que contiene una representación de los datos que maneja el sistema, su lógica de negocio, y sus mecanismos de persistencia.

La vista, o interfaz de usuario, que compone la información que se envía al cliente y los mecanismos interacción con éste.

El controlador, que actúa como intermediario entre el Modelo y la Vista, gestionando el flujo de información entre ellos y las transformaciones para adaptar los datos a las necesidades de cada uno.

### El modelo  es responsable de:
- Acceder a la capa de almacenamiento de datos. Lo ideal es que el modelo sea independiente del sistema de almacenamiento.
- Define las reglas de negocio (la funcionalidad del sistema). Un ejemplo de regla puede ser: "Si la mercancía pedida no está en el almacén, consultar el tiempo de entrega estándar del proveedor".
- Lleva un registro de las vistas y controladores del sistema.
- Si estamos ante un modelo activo, notificará a las vistas los cambios que en los datos pueda producir un agente externo (por ejemplo, un fichero por lotes  que actualiza los datos, un temporizador que desencadena una inserción, etc.).

## El controlador es responsable de:
- Recibir los eventos de entrada (un clic, un cambio en un campo de texto, etc.).
- Contiene reglas de gestión de eventos, del tipo "SI Evento Z, entonces Acción W". Estas acciones pueden suponer peticiones al modelo o a las vistas.

Una de estas peticiones a las vistas puede ser una llamada al método:
```javascript
Actualizar();
```

Una petición al modelo puede ser
```javascript
ObtenerTiempo(OrdenVenta);
```

## La vista es responsable de:
- Recibir datos del modelo y los muestra al usuario.
- Tienen un registro de su controlador asociado (normalmente porque además lo instancia).
- Pueden dar el servicio de "Actualizar()", para que sea invocado por el controlador o por el modelo (cuando es un modelo activo que informa de los cambios en los datos producidos por otros agentes).

#### El flujo que sigue el control generalmente es el siguiente:
1. El usuario interactúa con la interfaz de usuario de alguna forma (por ejemplo, el usuario pulsa un botón, enlace, etc.)

2. El controlador recibe (por parte de los objetos de la interfaz-vista) la notificación de la acción solicitada por el usuario. El controlador gestiona el evento que llega, frecuentemente a través de un gestor de eventos (handler) o callback.

3. El controlador accede al modelo, actualizándolo, posiblemente modificándolo de forma adecuada a la acción solicitada por el usuario (por ejemplo, el controlador actualiza el carro de la compra del usuario). Los controladores complejos están a menudo estructurados usando un patrón de comando que encapsula las acciones y simplifica su extensión.
4. El controlador delega a los objetos de la vista la tarea de desplegar la interfaz de usuario.
La vista obtiene sus datos del modelo para generar la interfaz apropiada para el usuario donde se refleja los cambios en el modelo (por ejemplo, produce un listado del contenido del carro de la compra).

El modelo no debe tener conocimiento directo sobre la vista. Sin embargo, se podría utilizar el patrón Observador para proveer cierta indirección entre el modelo y la vista, permitiendo al modelo notificar a los interesados de cualquier cambio. Un objeto vista puede registrarse con el modelo y esperar a los cambios, pero aun así el modelo en sí mismo sigue sin saber nada de la vista.

El controlador no pasa objetos de dominio (el modelo) a la vista aunque puede dar la orden a la vista para que se actualice. Nota: En algunas implementaciones la vista no tiene acceso directo al modelo, dejando que el controlador envíe los datos del modelo a la vista.

5. La interfaz de usuario espera nuevas interacciones del usuario, comenzando el ciclo nuevamente.


## Ejemplo practico
Para este ejemplo vamos a programar en PHP una aplicación muy simple que nos permita listar, editar y eliminar notas de texto.
### Base de datos
Las notas que creemos se guardarán en una base de datos, que podremos crear con este SQL:
```sql
CREATE DATABASE mvc_example; 

CREATE TABLE `note` (
  `id` int(11) NOT NULL,
  `title` varchar(75) NOT NULL,
  `content` text NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

ALTER TABLE `note`
  ADD PRIMARY KEY (`id`);
```

### Estructura de ficheros

```
[re-directory]mvc
    [re-directory]config
        [re-file]config.php
    [re-directory]controller
        [re-file]note.php
    [re-directory]model
        [re-file]db.php
        [re-file]note.php
    [re-directory]view
        [re-directory]template
            [re-file]footer.php
            [re-file]header.php
           [re-file]confirm_delete_note.php
        [re-file]delete_note.php
        [re-file]edit_note.php
        [re-file]list_note.php
    [re-file]index.php
```
Este ejemplo es muy sencillo, ya que sólo utilizaremos un controlador y dos modelos. El único controlador se encargará de gestionar las notas, y los dos modelos serán uno para notas y otro para la base de datos.

### config/config.php
En este fichero guardaremos información básica que atañe tanto a la conexión con base de datos como otros parámetros. Por ejemplo, el controlador y la acción por defecto.

Si esta aplicación creciera, seguramente el uso de este fichero estaría más justificado.
```php
<?php

/* Database connection values */
define("DB_HOST", "localhost");
define("DB", "mvc_example");
define("DB_USER", "root");
define("DB_PASS", "");

/* Default options */
define("DEFAULT_CONTROLLER", "note");
define("DEFAULT_ACTION", "list");

?>
```

### controller/note.php
Será el único controlador de nuestra aplicación. Se encarga de recibir las peticiones desde la vista, solicitar información y/u ordenar cambios al modelo.
```php
<?php 

require_once 'model/note.php';

class noteController{
	public $page_title;
	public $view;

	public function __construct() {
		$this->view = 'list_note';
		$this->page_title = '';
		$this->noteObj = new Note();
	}

	/* List all notes */
	public function list(){
		$this->page_title = 'Listado de notas';
		return $this->noteObj->getNotes();
	}

	/* Load note for edit */
	public function edit($id = null){
		$this->page_title = 'Editar nota';
		$this->view = 'edit_note';
		/* Id can from get param or method param */
		if(isset($_GET["id"])) $id = $_GET["id"];
		return $this->noteObj->getNoteById($id);
	}

	/* Create or update note */
	public function save(){
		$this->view = 'edit_note';
		$this->page_title = 'Editar nota';
		$id = $this->noteObj->save($_POST);
		$result = $this->noteObj->getNoteById($id);
		$_GET["response"] = true;
		return $result;
	}

	/* Confirm to delete */
	public function confirmDelete(){
		$this->page_title = 'Eliminar nota';
		$this->view = 'confirm_delete_note';
		return $this->noteObj->getNoteById($_GET["id"]);
	}

	/* Delete */
	public function delete(){
		$this->page_title = 'Listado de notas';
		$this->view = 'delete_note';
		return $this->noteObj->deleteNoteById($_POST["id"]);
	}

}

?>
```

### model/db.php
Creamos este modelo para facilitar las operaciones contra base de datos.
```php
<?php 

require_once 'config/config.php';

class Db {

	private $host;
	private $db;
	private $user;
	private $pass;
	public $conection;

	public function __construct() {		

		$this->host = constant('DB_HOST');
		$this->db = constant('DB');
		$this->user = constant('DB_USER');
		$this->pass = constant('DB_PASS');

		try {
           $this->conection = new PDO('mysql:host='.$this->host.'; dbname='.$this->db, $this->user, $this->pass);
        } catch (PDOException $e) {
            echo $e->getMessage();
            exit();
        }

	}

}

?>
```

### model/note.php
Dentro de nuestro ejemplo, es el modelo principal. Contiene métodos que operan la información en la base de datos.

Por ejemplo, tenemos un método para cargar las notas, y otros para cargar una nota, actualizar etc.
```php
<?php 

class Note {

	private $table = 'note';
	private $conection;

	public function __construct() {
		
	}

	/* Set conection */
	public function getConection(){
		$dbObj = new Db();
		$this->conection = $dbObj->conection;
	}

	/* Get all notes */
	public function getNotes(){
		$this->getConection();
		$sql = "SELECT * FROM ".$this->table;
		$stmt = $this->conection->prepare($sql);
		$stmt->execute();

		return $stmt->fetchAll();
	}

	/* Get note by id */
	public function getNoteById($id){
		if(is_null($id)) return false;
		$this->getConection();
		$sql = "SELECT * FROM ".$this->table. " WHERE id = ?";
		$stmt = $this->conection->prepare($sql);
		$stmt->execute([$id]);

		return $stmt->fetch();
	}

	/* Save note */
	public function save($param){
		$this->getConection();

		/* Set default values */
		$title = $content = "";

		/* Check if exists */
		$exists = false;
		if(isset($param["id"]) and $param["id"] !=''){
			$actualNote = $this->getNoteById($param["id"]);
			if(isset($actualNote["id"])){
				$exists = true;	
				/* Actual values */
				$id = $param["id"];
				$title = $actualNote["title"];
				$content = $actualNote["content"];
			}
		}

		/* Received values */
		if(isset($param["title"])) $title = $param["title"];
		if(isset($param["content"])) $content = $param["content"];

		/* Database operations */
		if($exists){
			$sql = "UPDATE ".$this->table. " SET title=?, content=? WHERE id=?";
			$stmt = $this->conection->prepare($sql);
			$res = $stmt->execute([$title, $content, $id]);
		}else{
			$sql = "INSERT INTO ".$this->table. " (title, content) values(?, ?)";
			$stmt = $this->conection->prepare($sql);
			$stmt->execute([$title, $content]);
			$id = $this->conection->lastInsertId();
		}	

		return $id;	

	}

	/* Delete note by id */
	public function deleteNoteById($id){
		$this->getConection();
		$sql = "DELETE FROM ".$this->table. " WHERE id = ?";
		$stmt = $this->conection->prepare($sql);
		return $stmt->execute([$id]);
	}

}

?>
```

### view/template/footer.php
En el directorio view tendremos las vistas de nuestra aplicación. Como tanto el header como el footer serán los mismos en todas las vistas, en lugar de repetir el código, los crearemos en el directorio template y los llamaremos en las vistas.

El footer simplemente cerrará las etiquetas body y html, además del div que tiene la clase container y que abrimos en el header.
```html
</div>
</body>
</html>
```

### view/template/header.php
En esta parte simplementes abriremos las etiquetas necesarias de html, así como incrustaremos las llamadas a las librerías que vamos a utilizar. En nuestro caso, para no tener que trabajar demasiado los estilos, utilizaremos Bootstrap.

Además, mostraremos en el encabezado un título de página, que se cargará de forma dinámica desde nuestro controlador.
```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title></title>
	<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" integrity="sha384-1BmE4kWBq78iYhFldvKuhfTAU6auU8tT94WrHftjDbrCEXSU1oBoqyl2QvZ6jIW3" crossorigin="anonymous">

	<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.min.js" integrity="sha384-QJHtvGhmr9XOIpI6YVutG+2QOK9T+ZnN4kzFN1RtK3zEFEIsxhlmWl5/YESvpZ13" crossorigin="anonymous"></script>
</head>
<body>
	<div class="container">
		<header class="mb-5">
			<div class="p-5 text-center bg-light" style="margin-top: 58px;">
				<h1 class="mb-3"><?php echo $controller->page_title; ?></h1>
				<h4 class="mb-3">-</h4>
			</div>
		</header>
```

### view/confirm_delete_note.php
Esta vista se mostrará cuando el usuario haga click en eliminar una nota. Simplemente pide confirmación, por si el usuario se arrepiente y no desea eliminar la nota realmente.

Para que sea más claro, mostraremos el título de la nota que se dispone a eliminar.
```html
<div class="row">
	<form class="form" action="index.php?controller=note&action=delete" method="POST">
		<input type="hidden" name="id" value="<?php echo $dataToView["data"]["id"]; ?>" />
		<div class="alert alert-warning">
			<b>¿Confirma que desea eliminar esta nota?:</b>
			<i><?php echo $dataToView["data"]["title"]; ?></i>
		</div>
		<input type="submit" value="Eliminar" class="btn btn-danger"/>
		<a class="btn btn-outline-success" href="index.php?controller=note&action=list">Cancelar</a>
	</form>
</div>
```

### view/delete_note.php
Es la vista que se carga a continuación de haber eliminado una nota. Nos muestra un simple mensaje informativo de que hemos eliminado la nota, y un link para volver al listado.
```html
<div class="row">
	<div class="alert alert-success">
		Nota eliminada correctamente. <a href="index.php?controller=note&action=list">Volver al listado</a>
	</div>
</div>
```

### view/edit_note.php
Esta vista nos mostrará un formulario en el que podremos tanto crear una nota, como edtar una existente.
```php
<?php
$id = $title = $content = "";

if(isset($dataToView["data"]["id"])) $id = $dataToView["data"]["id"];
if(isset($dataToView["data"]["title"])) $title = $dataToView["data"]["title"];
if(isset($dataToView["data"]["content"])) $content = $dataToView["data"]["content"];

?>
<div class="row">
	<?php
	if(isset($_GET["response"]) and $_GET["response"] === true){
		?>
		<div class="alert alert-success">
			Operación realizada correctamente. <a href="index.php?controller=note&action=list">Volver al listado</a>
		</div>
		<?php
	}
	?>
	<form class="form" action="index.php?controller=note&action=save" method="POST">
		<input type="hidden" name="id" value="<?php echo $id; ?>" />
		<div class="form-group">
			<label>Título</label>
			<input class="form-control" type="text" name="title" value="<?php echo $title; ?>" />
		</div>
		<div class="form-group mb-2">
			<label>Contenido</label>
			<textarea class="form-control" style="white-space: pre-wrap;" name="content"><?php echo $content; ?></textarea>
		</div>
		<input type="submit" value="Guardar" class="btn btn-primary"/>
		<a class="btn btn-outline-danger" href="index.php?controller=note&action=list">Cancelar</a>
	</form>
</div>
```

### view/list_note.php
Es la vista que se encarga de mostrarnos las notas ya creadas, si las hay. Además tiene un botón para crear una nota nueva.
```php
<div class="row">
	<div class="col-md-12 text-right">
		<a href="index.php?controller=note&action=edit" class="btn btn-outline-primary">Crear nota</a>
		<hr/>
	</div>
	<?php
	if(count($dataToView["data"])>0){
		foreach($dataToView["data"] as $note){
			?>
			<div class="col-md-3">
				<div class="card-body border border-secondary rounded">
					<h5 class="card-title"><?php echo $note['title']; ?></h5>
					<div class="card-text"><?php echo nl2br($note['content']); ?></div>
					<hr class="mt-1"/>
					<a href="index.php?controller=note&action=edit&id=<?php echo $note['id']; ?>" class="btn btn-primary">Editar</a>
					<a href="index.php?controller=note&action=confirmDelete&id=<?php echo $note['id']; ?>" class="btn btn-danger">Eliminar</a>
				</div>
			</div>
			<?php
		}
	}else{
		?>
		<div class="alert alert-info">
			Actualmente no existen notas.
		</div>
		<?php
	}
	?>
</div>
```

### index.php
En este fichero recibiremos todas las peticiones, tanto para el controlador de notas como para otros si los hubiere.
```php
<?php 

require_once 'config/config.php';
require_once 'model/db.php';

if(!isset($_GET["controller"])) $_GET["controller"] = constant("DEFAULT_CONTROLLER");
if(!isset($_GET["action"])) $_GET["action"] = constant("DEFAULT_ACTION");

$controller_path = 'controller/'.$_GET["controller"].'.php';

/* Check if controller exists */
if(!file_exists($controller_path)) $controller_path = 'controller/'.constant("DEFAULT_CONTROLLER").'.php';

/* Load controller */
require_once $controller_path;
$controllerName = $_GET["controller"].'Controller';
$controller = new $controllerName();

/* Check if method is defined */
$dataToView["data"] = array();
if(method_exists($controller,$_GET["action"])) $dataToView["data"] = $controller->{$_GET["action"]}();


/* Load views */
require_once 'view/template/header.php';
require_once 'view/'.$controller->view.'.php';
require_once 'view/template/footer.php';

?>
```

La forma de acceder a nuestra aplicación, si por ejemplo la hemos creado en nuestro propio pc, sería algo así:

http://localhost/mvc/index.php?controller=note&action=list

Llamaremos al archivo index.php, pasándole como parámetros el nombre del controlador y la acción que necesitamos.

Será el propio controlador el que decida que vista se va a cargar, según las que tengamos definidas.