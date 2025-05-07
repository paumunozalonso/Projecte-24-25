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

## `guardar_servei.php`

```php

<?php
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

$usuari_id      = $_SESSION['usuari_id'];
$nom_servei     = $_POST['nom_servei'];
$plataforma     = $_POST['plataforma'];
$versio_mysql   = $_POST['versio_mysql'];
$vcpu           = $_POST['vcpu'];
$ram            = $_POST['ram'];
$ssd            = $_POST['ssd'];
$xarxa          = $_POST['xarxa'];
$estat          = "aturat";
$backup_diari   = isset($_POST['backup_diari']) ? 1 : 0;

$servei_id      = uniqid("servei_");
$port_http      = rand(8000, 8999);

// Càlcul de preu
$preu_cpu     = $vcpu * 2;
$preu_ram     = $ram * 1.5;
$preu_ssd     = $ssd * 0.1;

switch ($xarxa) {
  case "100":  $preu_xarxa = 1; break;
  case "500":  $preu_xarxa = 3; break;
  case "1000": $preu_xarxa = 5; break;
  default:     $preu_xarxa = 0;
}

$preu_backup = $backup_diari ? 1.5 : 0;
$preu_total = $preu_cpu + $preu_ram + $preu_ssd + $preu_xarxa + $preu_backup;

//Inserir a la base de dades amb el preu
$stmt = $conn->prepare("INSERT INTO serveis
  (servei_id, usuari_id, nom_servei, plataforma, vcpu, ram, ssd, versio_mysql, xarxa, estat, port_http, backup_diari, preu_total)
  VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
$stmt->bind_param("sissiiissssii", $servei_id, $usuari_id, $nom_servei, $plataforma, $vcpu, $ram, $ssd, $versio_mysql, $xarxa, $estat, $port_http, $backup_diari, $preu_total);
$stmt->execute();
$stmt->close();

// Crear carpetes
$base_path = "/var/www/containers/usuari_$usuari_id/$servei_id";
mkdir("$base_path/app_data", 0777, true);
mkdir("$base_path/db_data", 0777, true);

// Generar docker-compose.yml
$compose = <<<YML
version: '3.8'
services:
  app:
    image: $plataforma
    container_name: ${servei_id}_app
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
      - $base_path/app_data:/var/www/html

  db:
    image: mysql:$versio_mysql
    restart: always
    environment:
      MYSQL_DATABASE: db_$servei_id
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - $base_path/db_data:/var/lib/mysql
YML;

file_put_contents("$base_path/docker-compose.yml", $compose);

// Tornar al panell
header("Location: panell.php");
exit;
?>
```

## `iniciar.php`

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
$compose_path = "$base_path/docker-compose.yml";
echo "Ruta que s'està comprovant: $base_path/docker-compose.yml<br>";
if (file_exists($compose_path)) {
    shell_exec("cd $base_path && docker compose up -d");

    $conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");

    if ($conn->connect_error) {
        die("Error de connexió: " . $conn->connect_error);
    }

    $stmt = $conn->prepare("UPDATE serveis SET estat = 'actiu' WHERE servei_id = ? AND usuari_id = ?");
    $stmt->bind_param("si", $servei_id, $usuari_id);
    $stmt->execute();
    $stmt->close();
    $conn->close();

    header("Location: gestionar.php");
    exit;

} else {
    die("El servei no existeix o falta el docker-compose.yml");
}
?>
```

## `modificar.php`

```php

<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
    header("Location: login.php");
    exit;
}

$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
if ($conn->connect_error) {
    die("Error de connexió: " . $conn->connect_error);
}

$usuari_id = $_SESSION['usuari_id'];
$servei_id = $_GET['servei_id'] ?? '';

$stmt = $conn->prepare("SELECT * FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$result = $stmt->get_result();
$servei = $result->fetch_assoc();
$stmt->close();

if (!$servei) {
    die("El servei no existeix.");
}
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Modificar Servei</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #f3f7fb;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
    }
    .container {
      background-color: #ffffff;
      padding: 30px;
      border-radius: 12px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.1);
      width: 400px;
    }
    h2 {
      text-align: center;
      color: #2c3e50;
    }
    label {
      font-weight: bold;
      margin-top: 15px;
      display: block;
    }
    select, input[type="text"] {
      width: 100%;
      padding: 8px;
      border-radius: 6px;
      border: 1px solid #ccc;
      margin-top: 5px;
    }
    .checkbox {
      margin-top: 15px;
    }
    button {
      margin-top: 20px;
      width: 100%;
      background-color: #3b82f6;
      color: white;
      padding: 10px;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }
    button:hover {
      background-color: #2563eb;
    }
    .back {
      display: flex;
      justify-content: center;
      margin-top: 15px;
    }
    .back a {
      text-decoration: none;
      font-size: 14px;
      color: #555;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Modificar Servei: <?= htmlspecialchars($servei['nom_servei']) ?></h2>
    <form action="guardar_canvis.php" method="POST">
      <input type="hidden" name="servei_id" value="<?= htmlspecialchars($servei_id) ?>">
      <input type="hidden" name="nom_servei" value="<?= htmlspecialchars($servei['nom_servei']) ?>">
      <label>Plataforma:</label>
      <select name="plataforma" required>
        <option value="wordpress" <?= $servei['plataforma'] === 'wordpress' ? 'selected' : '' ?>>WordPress</option>
        <option value="nextcloud" <?= $servei['plataforma'] === 'nextcloud' ? 'selected' : '' ?>>Nextcloud</option>
      </select>
      <label>Versió MySQL:</label>
      <select name="versio_mysql" required>
        <option value="5.7" <?= $servei['versio_mysql'] === '5.7' ? 'selected' : '' ?>>MySQL 5.7</option>
        <option value="8.0" <?= $servei['versio_mysql'] === '8.0' ? 'selected' : '' ?>>MySQL 8.0</option>
      </select>
      <label>vCPU:</label>
      <select name="vcpu" required>
        <option value="1" <?= $servei['vcpu'] == 1 ? 'selected' : '' ?>>1 vCPU</option>
        <option value="2" <?= $servei['vcpu'] == 2 ? 'selected' : '' ?>>2 vCPU</option>
        <option value="4" <?= $servei['vcpu'] == 4 ? 'selected' : '' ?>>4 vCPU</option>
      </select>
      <label>RAM (GiB):</label>
      <select name="ram" required>
        <option value="1" <?= $servei['ram'] == 1 ? 'selected' : '' ?>>1 GiB</option>
        <option value="2" <?= $servei['ram'] == 2 ? 'selected' : '' ?>>2 GiB</option>
        <option value="4" <?= $servei['ram'] == 4 ? 'selected' : '' ?>>4 GiB</option>
      </select>
      <label>SSD (GiB):</label>
      <select name="ssd" required>
        <option value="20" <?= $servei['ssd'] == 20 ? 'selected' : '' ?>>20 GiB</option>
        <option value="50" <?= $servei['ssd'] == 50 ? 'selected' : '' ?>>50 GiB</option>
        <option value="100" <?= $servei['ssd'] == 100 ? 'selected' : '' ?>>100 GiB</option>
      </select>
      <label>Amplada de banda:</label>
      <select name="xarxa" required>
        <option value="100" <?= $servei['xarxa'] == 100 ? 'selected' : '' ?>>100 Mbps</option>
        <option value="500" <?= $servei['xarxa'] == 500 ? 'selected' : '' ?>>500 Mbps</option>
        <option value="1000" <?= $servei['xarxa'] == 1000 ? 'selected' : '' ?>>1 Gbps</option>
      </select>
      <div class="checkbox">
        <label>
          <input type="checkbox" name="backup_diari" value="1" <?= $servei['backup_diari'] ? 'checked' : '' ?>>
          Backup diari
        </label>
      </div>
      <button type="submit">Desar Canvis</button>
    </form>
    <div class="back">
      <a href="panell.php">← Tornar al panell</a>
    </div>
  </div>
</body>
</html>

```
