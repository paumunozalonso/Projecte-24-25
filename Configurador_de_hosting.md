# Fitxers PHP del projecte

## `aturar.php`

```php

<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
    header("Location: login.php");
    exit;
}

$servei_id = $_GET['servei_id'] ?? '';
$usuari_id = $_SESSION['usuari_id'];
$base_path = "/var/www/containers/usuari_$usuari_id/$servei_id";
$docker_compose_path = "$base_path/docker-compose.yml";

if (file_exists($docker_compose_path)) {
    shell_exec("cd $base_path && docker compose down");

    $conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
    $stmt = $conn->prepare("UPDATE serveis SET estat = 'aturat' WHERE servei_id = ? AND usuari_id = ?");
    $stmt->bind_param("si", $servei_id, $usuari_id);
    $stmt->execute();
    $stmt->close();
    $conn->close();

    header("Location: gestionar.php");
    exit;
// veure error
} else {
    echo "El servei no existeix o falta el docker-compose.yml";
    echo "<br>Ruta comprovada: $docker_compose_path";
}
?>

```

## `eliminar.php`

```php

<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
    header("Location: login.php");
    exit;
}

$servei_id = $_GET['servei_id'] ?? '';
$usuari_id = $_SESSION['usuari_id'];

if (empty($servei_id)) {
    die("Falta l'ID del servei.");
}

// Connexió a la base de dades
$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
if ($conn->connect_error) {
    die("Error de connexió: " . $conn->connect_error);
}

// Obtenir el port_http
$stmt = $conn->prepare("SELECT plataforma FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$result = $stmt->get_result();
if ($result->num_rows === 0) {
    die("El servei no existeix.");
}
$servei = $result->fetch_assoc();
$stmt->close();

// Ruta del contenidor
$base_path = "/var/www/containers/usuari_$usuari_id/$servei_id";

// Parar i eliminar contenidors
shell_exec("cd $base_path && docker compose down");

// Eliminar la carpeta + evitar liarla
shell_exec("sudo rm -rf " . escapeshellarg($base_path));

// Eliminar de la base de dades
$stmt = $conn->prepare("DELETE FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$stmt->close();

$conn->close();

// Redirigir
header("Location: gestionar.php");
exit;
?>

```

## `guardar_canvis.php`

```php

<?php

// Mostrar errors
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

session_start();
if (!isset($_SESSION['usuari_id'])) {
    header("Location: login.php");
    exit;
}

$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
if ($conn->connect_error) {
    die("Error de connexió: " . $conn->connect_error);
}

// Recollir dades
$usuari_id     = $_SESSION['usuari_id'];
$servei_id     = $_POST['servei_id'] ?? '';
$nom_servei    = $_POST['nom_servei'] ?? '';
$vcpu          = $_POST['vcpu'] ?? '';
$ram           = $_POST['ram'] ?? '';
$ssd           = $_POST['ssd'] ?? '';
$plataforma    = $_POST['plataforma'] ?? '';
$versio_mysql  = $_POST['versio_mysql'] ?? '';
$xarxa         = $_POST['xarxa'] ?? '';
$backup_diari  = isset($_POST['backup_diari']) ? 1 : 0;

if (empty($servei_id) || empty($nom_servei) || empty($vcpu) || empty($ram) || empty($ssd) || empty($plataforma) || empty($versio_mysql) || empty($xarxa)) {
    die("Error: Falta algun camp obligatori.");
}

// Verificar estat i plataforma antiga
$check = $conn->prepare("SELECT estat, plataforma FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$check->bind_param("si", $servei_id, $usuari_id);
$check->execute();
$check->bind_result($estat_actual, $plataforma_antiga);
$check->fetch();
$check->close();

if ($estat_actual !== 'aturat') {
    die("Primer has d’aturar el servei abans de modificar-lo. <br><br><a href='gestionar.php'>Tornar</a>");
}

// Recuperar port
$stmt = $conn->prepare("SELECT port_http FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$stmt->bind_result($port_http);
$stmt->fetch();
$stmt->close();

if (empty($port_http)) {
    $port_http = rand(8000, 8999);
}

// Rutes
$base_path = "/var/www/containers/usuari_$usuari_id/$servei_id";
$app_data_path = "$base_path/app_data";
$db_data_path  = "$base_path/db_data";

// Aturar contenidors
shell_exec("cd $base_path && docker compose down");

// Si la plataforma ha canviat, només borrem app_data
if ($plataforma_antiga !== $plataforma) {
    shell_exec("rm -rf " . escapeshellarg($app_data_path));
    mkdir($app_data_path, 0777, true);
    chmod($app_data_path, 0777);
}

// Calcular preu
$preu_total = ($vcpu * 2.0) + ($ram * 1.5) + ($ssd * 0.1) + ($xarxa * 0.01);
if ($backup_diari) $preu_total += 1.5;
$preu_total = round($preu_total, 2);

// Actualitzar a BBDD
$stmt = $conn->prepare("UPDATE serveis SET nom_servei = ?, vcpu = ?, ram = ?, ssd = ?, plataforma = ?, versio_mysql = ?, xarxa = ?, backup_diari = ?, estat = 'aturat', port_http = ?, preu_total = ? WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("siisssiisdsi", $nom_servei, $vcpu, $ram, $ssd, $plataforma, $versio_mysql, $xarxa, $backup_diari, $port_http, $preu_total, $servei_id, $usuari_id);
$stmt->execute();
$stmt->close();

// Ruta de muntatge segons plataforma
function obtenirRutaApp($plataforma) {
    switch (strtolower($plataforma)) {
        case 'wordpress':
        case 'nextcloud':
            return '/var/www/html';
        default:
            return '/var/www/html';
    }
}
$app_path = obtenirRutaApp($plataforma);

// Generar docker-compose.yml
$compose = <<<YML
version: '3.8'
services:
  app:
    image: $plataforma
    container_name: servei_{$servei_id}_app
    depends_on:
      - db
    ports:
      - "$port_http:80"
    deploy:
      resources:
        limits:
          cpus: "$vcpu"
          memory: "${ram}g"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: db_$servei_id
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
    volumes:
      - $app_data_path:$app_path

  db:
    image: mysql:$versio_mysql
    container_name: servei_{$servei_id}_db
    restart: always
    environment:
      MYSQL_DATABASE: db_$servei_id
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - $db_data_path:/var/lib/mysql
YML;

file_put_contents("$base_path/docker-compose.yml", $compose);

// Redirigir
header("Location: gestionar.php?msg=canvis_guardats");
exit;
?>
```


