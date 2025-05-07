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

## `packs.php`

```php
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Packs Predefinits</title>
  <style>
    body {
      font-family: 'Segoe UI', sans-serif;
      background-color: #f4f8fb;
      padding: 40px;
      text-align: center;
    }
    h1 {
      margin-bottom: 30px;
    }
    .packs-container {
      display: flex;
      justify-content: center;
      gap: 30px;
      flex-wrap: wrap;
    }
    .pack {
      background-color: white;
      padding: 20px;
      border-radius: 12px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.1);
      width: 250px;
    }
    .pack h3 {
      margin-top: 0;
    }
    .basic { border-top: 5px solid #22c55e; }
    .mitja { border-top: 5px solid #eab308; }
    .pro { border-top: 5px solid #ef4444; }
    .pack ul {
      text-align: left;
      padding-left: 20px;
    }
    .pack a {
      display: inline-block;
      margin-top: 15px;
      padding: 10px 20px;
      background-color: #3b82f6;
      color: white;
      text-decoration: none;
      border-radius: 8px;
    }
    .pack a:hover {
      background-color: #2563eb;
    }
  </style>
</head>
<body>
  <h1>Tria un Pack Predefinit</h1>
  <div class="packs-container">
    <div class="pack basic">
      <h3>Pack Bàsic</h3>
      <ul>
        <li>1 vCPU</li>
        <li>1 GiB RAM</li>
        <li>20 GiB SSD</li>
        <li>100 Mbps</li>
        <li>Sense Backup</li>
      </ul>
      <a href="crear_servei_pack.php?pack=basic">Seleccionar</a>
    </div>

    <div class="pack mitja">
      <h3>Pack Mitjà</h3>
      <ul>
        <li>2 vCPU</li>
        <li>2 GiB RAM</li>
        <li>50 GiB SSD</li>
        <li>500 Mbps</li>
        <li>Amb Backup</li>
      </ul>
      <a href="crear_servei_pack.php?pack=mitja">Seleccionar</a>
    </div>

    <div class="pack pro">
      <h3>Pack Pro</h3>
      <ul>
        <li>4 vCPU</li>
        <li>4 GiB RAM</li>
        <li>100 GiB SSD</li>
        <li>1 Gbps</li>
        <li>Amb Backup</li>
      </ul>
      <a href="crear_servei_pack.php?pack=pro">Seleccionar</a>
    </div>
  </div>
</body>
</html>

```


## `crear_servei_pack.php`

```php
<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
  header("Location: login.php");
  exit;
}

$pack = $_GET['pack'] ?? 'basic';

$configuracions = [
  'basic' => ['vcpu' => 1, 'ram' => 1, 'ssd' => 20, 'xarxa' => 100, 'backup_diari' => 0],
  'mitja' => ['vcpu' => 2, 'ram' => 2, 'ssd' => 50, 'xarxa' => 500, 'backup_diari' => 1],
  'pro' => ['vcpu' => 4, 'ram' => 4, 'ssd' => 100, 'xarxa' => 1000, 'backup_diari' => 1]
];

$conf = $configuracions[$pack] ?? $configuracions['basic'];
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Crear Servei - Pack <?= ucfirst($pack) ?></title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #f4f8fb;
      padding: 40px;
    }
    .container {
      background-color: #fff;
      padding: 30px;
      border-radius: 12px;
      max-width: 600px;
      margin: auto;
      box-shadow: 0 10px 25px rgba(0,0,0,0.1);
    }
    h2 {
      text-align: center;
      color: #2c3e50;
    }
    label {
      font-weight: bold;
      display: block;
      margin-top: 15px;
    }
    input, select {
      width: 100%;
      padding: 10px;
      margin-top: 5px;
      border: 1px solid #ccc;
      border-radius: 6px;
    }
    .submit-btn {
      background-color: #3b82f6;
      color: white;
      padding: 12px;
      border: none;
      border-radius: 8px;
      width: 100%;
      font-size: 16px;
      margin-top: 25px;
      cursor: pointer;
    }
    .submit-btn:hover {
      background-color: #2563eb;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Crear Servei - Pack <?= ucfirst($pack) ?></h2>
    <form action="guardar_servei.php" method="POST">
      <input type="hidden" name="vcpu" value="<?= $conf['vcpu'] ?>">
      <input type="hidden" name="ram" value="<?= $conf['ram'] ?>">
      <input type="hidden" name="ssd" value="<?= $conf['ssd'] ?>">
      <input type="hidden" name="xarxa" value="<?= $conf['xarxa'] ?>">
      <input type="hidden" name="backup_diari" value="<?= $conf['backup_diari'] ?>">

      <label>Nom del servei:</label>
      <input type="text" name="nom_servei" required>

      <label>Plataforma:</label>
      <select name="plataforma">
        <option value="wordpress">WordPress</option>
        <option value="nextcloud">Nextcloud</option>
      </select>

      <label>Versió MySQL:</label>
      <select name="versio_mysql">
        <option value="5.7">MySQL 5.7</option>
        <option value="8.0">MySQL 8.0</option>
      </select>

      <button type="submit" class="submit-btn">Crear amb el Pack</button>
    </form>
  </div>
</body>
</html>

```

## `gestionar.php`

```php
<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
  header("Location: login.php");
  exit;
}

$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
$usuari_id = $_SESSION['usuari_id'];

// Obtenir tots els serveis de l’usuari amb preu
$stmt = $conn->prepare("SELECT servei_id, nom_servei, vcpu, ram, ssd, estat, port_http, preu_total FROM serveis WHERE usuari_id = ?");
$stmt->bind_param("i", $usuari_id);
$stmt->execute();
$result = $stmt->get_result();
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Gestió de Servidors</title>
  <style>
    body {
      background-color: #f4f8fb;
      font-family: 'Segoe UI', sans-serif;
      padding: 40px;
      display: flex;
      justify-content: center;
    }

    .container {
      background-color: #ffffff;
      padding: 30px;
      border-radius: 16px;
      box-shadow: 0 10px 25px rgba(0,0,0,0.1);
      width: 100%;
      max-width: 1000px;
    }

    h2 {
      text-align: center;
      color: #2c3e50;
      margin-bottom: 25px;
    }

    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 20px;
    }

    th, td {
      padding: 15px;
      text-align: center;
      border-bottom: 1px solid #e0e6ed;
    }

    th {
      background-color: #f5faff;
      color: #2c3e50;
    }

    .badge {
      padding: 5px 10px;
      border-radius: 10px;
      font-weight: bold;
      font-size: 0.85rem;
    }

    .actiu {
      background-color: #c6f6d5;
      color: #276749;
    }

    .aturat {
      background-color: #fed7d7;
      color: #c53030;
    }

    .btn {
      padding: 8px 12px;
      border-radius: 6px;
      font-weight: bold;
      border: none;
      cursor: pointer;
      font-size: 0.85rem;
      margin: 2px;
    }

    .btn-mod {
      background-color: #4299e1;
      color: white;
    }

    .btn-stop {
      background-color: #feb2b2;
      color: #742a2a;
    }

    .btn-start {
      background-color: #c6f6d5;
      color: #22543d;
    }

    .btn-del {
      background-color: #faf089;
      color: #744210;
    }

    .btn-web {
      background-color: #edf2f7;
      color: #2b6cb0;
      border: 1px solid #cbd5e0;
    }

    .top-btn {
      float: right;
      margin-bottom: 20px;
      background-color: #3b82f6;
      color: white;
      padding: 10px 16px;
      border-radius: 8px;
      text-decoration: none;
      font-weight: bold;
    }

    .top-btn:hover {
      background-color: #2563eb;
    }
  </style>
</head>
<body>
  <div class="container">
    <a href="panell.php" class="top-btn">⟵ Tornar al panell</a>
    <h2>Els meus servidors</h2>
    <table>
      <tr>
        <th>Nom del servidor</th>
        <th>vCPU</th>
        <th>RAM</th>
        <th>SSD</th>
        <th>Preu</th>
        <th>Estat</th>
        <th>Accions</th>
      </tr>

      <?php while ($row = $result->fetch_assoc()): ?>
        <tr>
          <td><?= htmlspecialchars($row['nom_servei']) ?></td>
          <td><?= htmlspecialchars($row['vcpu']) ?> vCPU</td>
          <td><?= htmlspecialchars($row['ram']) ?> GiB</td>
          <td><?= htmlspecialchars($row['ssd']) ?> GiB</td>
          <td><?= number_format($row['preu_total'], 2) ?> €</td>
          <td>
            <span class="badge <?= $row['estat'] === 'actiu' ? 'actiu' : 'aturat' ?>">
              <?= ucfirst($row['estat']) ?>
            </span>
          </td>
          <td>
            <a href="modificar.php?servei_id=<?= $row['servei_id'] ?>"><button class="btn btn-mod">Modificar</button></a>
            <?php if ($row['estat'] === 'actiu'): ?>
              <a href="aturar.php?servei_id=<?= $row['servei_id'] ?>"><button class="btn btn-stop">Aturar</button></a>
            <?php else: ?>
              <a href="iniciar.php?servei_id=<?= $row['servei_id'] ?>"><button class="btn btn-start">Iniciar</button></a>
            <?php endif; ?>
            <a href="eliminar.php?servei_id=<?= $row['servei_id'] ?>"><button class="btn btn-del">Eliminar</button></a>
            <a href="http://<?= $_SERVER['SERVER_ADDR'] . ':' . $row['port_http'] ?>" target="_blank">
              <button class="btn btn-web">Obrir Web</button>
            </a>
          </td>
        </tr>
      <?php endwhile; ?>

    </table>
  </div>
</body>
</html>

<?php
$stmt->close();
$conn->close();
?>

```

## `guardar_dades_usuari.php`

```php
<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
    header("Location: login.php");
    exit;
}

$usuari_id = $_SESSION['usuari_id'];

$nom     = $_POST['nom'] ?? '';
$email   = $_POST['email'] ?? '';
$telefon = $_POST['telefon'] ?? '';
$dni     = $_POST['dni'] ?? '';
$nova_contrasenya = $_POST['nova_contrasenya'] ?? '';

// Validació bàsica
if (empty($nom) || empty($email) || empty($telefon) || empty($dni)) {
    die("Falten dades obligatòries");
}

// Connexió a la base de dades
$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
if ($conn->connect_error) {
    die("Error de connexió: " . $conn->connect_error);
}

// Actualitzar dades bàsiques
$stmt = $conn->prepare("UPDATE usuaris SET nom = ?, email = ?, telefon = ?, dni = ? WHERE id = ?");
$stmt->bind_param("ssssi", $nom, $email, $telefon, $dni, $usuari_id);

if (!$stmt->execute()) {
    echo "Error al guardar dades: " . $stmt->error;
    $stmt->close();
    $conn->close();
    exit;
}
$stmt->close();

// Si s'ha introduït nova contrasenya, atualitza
if (!empty($nova_contrasenya)) {
    $hash = password_hash($nova_contrasenya, PASSWORD_DEFAULT);

    $stmt_pass = $conn->prepare("UPDATE usuaris SET contrasenya = ? WHERE id = ?");
    $stmt_pass->bind_param("si", $hash, $usuari_id);

    if (!$stmt_pass->execute()) {
        echo "Error al actualitzar la contrasenya: " . $stmt_pass->error;
        $stmt_pass->close();
        $conn->close();
        exit;
    }

    $stmt_pass->close();
}

$conn->close();
header("Location: panell.php?msg=canvis_guardats");
exit;
?>

```

## `index.php`
```php
root@docker:/var/www/html# cat index.php
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Hosting Personalitzat</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: 'Segoe UI', sans-serif;
      background-color: #f1f5f9;
      display: flex;
      flex-direction: column;
      min-height: 100vh;
    }

    header {
      background-color: #4a90e2;
      color: white;
      padding: 20px;
      text-align: center;
      font-size: 1.5rem;
      font-weight: bold;
    }

    .main {
      flex: 1;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 60px 20px;
      background: linear-gradient(to bottom right, #eaf4fd, #d0e7fb);
      text-align: center;
    }

    .main h1 {
      font-size: 2rem;
      color: #1a202c;
      margin-bottom: 15px;
    }

    .main p {
      font-size: 1rem;
      color: #4a5568;
      max-width: 500px;
      margin: 0 auto 25px;
    }

    .buttons a {
      display: inline-block;
      margin: 10px;
      padding: 12px 24px;
      border: 2px solid #4a90e2;
      border-radius: 8px;
      text-decoration: none;
      font-weight: bold;
      color: #ffffff;
      background-color: #4a90e2;
      transition: 0.3s;
    }

    .buttons a:nth-child(2) {
      background-color: transparent;
      color: #4a90e2;
    }

    .buttons a:hover {
      background-color: #3270c5;
      color: #fff;
    }

    footer {
      background-color: #e2e8f0;
      color: #718096;
      text-align: center;
      padding: 15px;
      font-size: 0.9rem;
    }
  </style>
</head>
<body>
  <header>
    Hosting Personalitzat
  </header>

  <div class="main">
    <h1>Benvingut a la teva plataforma de gestió</h1>
    <p>
      Accedeix al teu espai privat on podràs gestionar i controlar fàcilment els teus servidors.
      Inicia sessió o crea un compte nou per començar.
    </p>
    <div class="buttons">
      <a href="login.php">Inicia sessió</a>
      <a href="registre.php">Registra’t</a>
    </div>
  </div>

  <footer>
    © 2025 – Plataforma de Hosting Personalitzat
  </footer>
</body>
</html>

```

## `login.php`
```php
<?php
session_start();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $email = $_POST['email'] ?? '';
    $contrasenya = $_POST['contrasenya'] ?? '';

    if (empty($email) || empty($contrasenya)) {
        die("Cal omplir tots els camps.");
    }

    $conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
    if ($conn->connect_error) {
        die("Error de connexió: " . $conn->connect_error);
    }

    $stmt = $conn->prepare("SELECT id, contrasenya FROM usuaris WHERE email = ?");
    $stmt->bind_param("s", $email);
    $stmt->execute();
    $stmt->bind_result($usuari_id, $hash);
    $stmt->fetch();
    $stmt->close();
    $conn->close();

    if (isset($usuari_id) && password_verify($contrasenya, $hash)) {
        $_SESSION['usuari_id'] = $usuari_id;
        header("Location: panell.php");
        exit;
    } else {
        die("Correu o contrasenya incorrectes.");
    }
}
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Iniciar sessió</title>
  <style>
    body {
      background-color: #e6f0fa;
      font-family: 'Segoe UI', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
    }
    .container {
      background-color: #ffffff;
      padding: 40px;
      border-radius: 20px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.1);
      max-width: 400px;
      width: 100%;
      text-align: center;
    }
    h2 {
      color: #2c3e50;
      margin-bottom: 30px;
    }
    input {
      width: 100%;
      padding: 12px;
      margin: 10px 0;
      border: none;
      border-radius: 10px;
      background-color: #f0f4f8;
    }
    button {
      padding: 12px 20px;
      background-color: #4a90e2;
      color: white;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      width: 100%;
    }
    button:hover {
      background-color: #3270c5;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Iniciar Sessió</h2>
    <form method="POST">
      <input type="email" name="email" placeholder="Correu electrònic" required>
      <input type="password" name="contrasenya" placeholder="Contrasenya" required>
      <button type="submit">Entra</button>
    </form>
    <br>
    <a href="index.php">
      <button style="background-color: #ccc; color: #2c3e50;">← Tornar</button>
    </a>
  </div>
</body>
</html>

```

## `nou_servei.php`
```php
<?php
session_start();
if (!isset($_SESSION['usuari_id'])) {
  header("Location: login.php");
  exit;
}
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Nou Servei</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #f4f8fb;
      padding: 40px;
    }

    .container {
      background-color: #fff;
      padding: 30px;
      border-radius: 12px;
      max-width: 700px;
      margin: auto;
      box-shadow: 0 10px 25px rgba(0,0,0,0.1);
    }

    h2 {
      text-align: center;
      margin-bottom: 30px;
      color: #2c3e50;
    }

    .back-btn {
      text-align: right;
      margin-bottom: 20px;
    }

    .back-btn a {
      text-decoration: none;
      background-color: #3b82f6;
      color: white;
      padding: 10px 16px;
      border-radius: 6px;
      font-weight: bold;
    }

    label {
      font-weight: bold;
      display: block;
      margin-top: 15px;
    }

    input[type="text"],
    select {
      width: 100%;
      padding: 10px;
      margin-top: 6px;
      border: 1px solid #ccc;
      border-radius: 6px;
    }

    .checkbox-group {
      margin-top: 20px;
    }

    .checkbox-group label {
      display: block;
      margin-bottom: 6px;
    }

    .submit-btn {
      background-color: #3b82f6;
      color: white;
      padding: 12px;
      border: none;
      border-radius: 8px;
      width: 100%;
      font-size: 16px;
      margin-top: 25px;
      cursor: pointer;
    }

    .submit-btn:hover {
      background-color: #2563eb;
    }
    .pack-card {
      margin-top: 30px;
      padding: 20px;
      background-color: #f0f9ff;
      border: 2px dashed #3b82f6;
      border-radius: 12px;
      text-align: center;
    }
    .pack-card h3 {
      color: #2563eb;
      margin-bottom: 10px;
    }
    .pack-card p {
      margin-bottom: 15px;
      font-size: 15px;
    }
    .pack-link {
      display: inline-block;
      padding: 10px 20px;
      background-color: #2563eb;
      color: white;
      text-decoration: none;
      font-weight: bold;
      border-radius: 8px;
      transition: background-color 0.3s ease;
    }
    .pack-link:hover {
      background-color: #1e40af;
    }

  </style>
</head>
<body>
  <div class="container">
    <div class="back-btn">
      <a href="panell.php">⬅ Tornar al panell</a>
    </div>

    <h2>Configura un Nou Servei</h2>
    <form action="guardar_servei.php" method="POST">

      <label>Nom del servei:</label>
      <input type="text" name="nom_servei" required>

      <label>Plataforma:</label>
      <select name="plataforma" required>
        <option value="wordpress">WordPress</option>
        <option value="nextcloud">Nextcloud</option>
      </select>

      <label>Versió MySQL:</label>
      <select name="versio_mysql">
        <option value="5.7">MySQL 5.7</option>
        <option value="8.0">MySQL 8.0</option>
      </select>

      <label>vCPU:</label>
      <select name="vcpu">
        <option value="1">1 vCPU</option>
        <option value="2">2 vCPU</option>
        <option value="4">4 vCPU</option>
      </select>

      <label>RAM (GiB):</label>
      <select name="ram">
        <option value="1">1 GiB</option>
        <option value="2">2 GiB</option>
        <option value="4">4 GiB</option>
      </select>

      <label>Emmagatzematge SSD:</label>
      <select name="ssd">
        <option value="20">20 GiB</option>
        <option value="50">50 GiB</option>
        <option value="100">100 GiB</option>
      </select>

      <label>Amplada de banda:</label>
      <select name="xarxa">
        <option value="100">100 Mbps</option>
        <option value="500">500 Mbps</option>
        <option value="1000">1 Gbps</option>
      </select>

      <div class="checkbox-group">
        <label><input type="checkbox" name="backup_diari" value="1"> Backup diari (+1,50 €)</label>
      </div>

      <button type="submit" class="submit-btn">Crear Servei</button>
      </form>
       <div class="pack-card">
        <h3>Packs predefinits disponibles</h3>
        <p>Vols una configuració ràpida i equilibrada? Tria un dels nostres packs!</p>
        <a href="packs.php" class="pack-link">Veure Packs</a>
       </div>
  </div>
</body>
</html>

```

## `panell.php`
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

// Obtenir dades usuari
$stmt = $conn->prepare("SELECT nom, email, telefon, dni FROM usuaris WHERE id = ?");
$stmt->bind_param("i", $usuari_id);
$stmt->execute();
$result = $stmt->get_result();
$usuari = $result->fetch_assoc();
$stmt->close();

// Obtenir preu total dels serveis
$stmt2 = $conn->prepare("SELECT SUM(preu_total) AS total FROM serveis WHERE usuari_id = ?");
$stmt2->bind_param("i", $usuari_id);
$stmt2->execute();
$result2 = $stmt2->get_result();
$total = $result2->fetch_assoc()['total'] ?? 0;
$stmt2->close();

$conn->close();
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Panell d'usuari</title>
  <style>
    body {
      font-family: 'Segoe UI', sans-serif;
      background-color: #f4f8fc;
      padding: 40px;
      display: flex;
      justify-content: center;
    }
    .container {
      background: #fff;
      padding: 30px;
      border-radius: 15px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.1);
      max-width: 600px;
      width: 100%;
      text-align: center;
    }
    h2 {
      color: #2c3e50;
      margin-bottom: 10px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    h2 img {
      width: 32px;
      margin-right: 10px;
    }
    p {
      color: #555;
      margin-bottom: 15px;
    }
    label {
      display: block;
      text-align: left;
      margin-top: 10px;
      font-weight: bold;
    }
    input {
      width: 100%;
      padding: 10px;
      margin-top: 5px;
      border-radius: 8px;
      border: 1px solid #ccc;
      font-size: 1rem;
    }
    button {
      margin-top: 20px;
      padding: 12px 20px;
      font-size: 1rem;
      background-color: #3498db;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: background 0.3s ease;
    }
    button:hover {
      background-color: #2980b9;
    }
    .buttons {
      margin-top: 30px;
      display: flex;
      justify-content: space-between;
    }
    .buttons a button {
      background-color: #ecf0f1;
      color: #2c3e50;
      font-weight: bold;
    }
    .back-button {
      margin-top: 20px;
      display: inline-block;
      background-color: #d1d1d1;
      color: #333;
      padding: 10px 15px;
      border-radius: 6px;
      text-decoration: none;
      font-weight: bold;
    }
    .back-button:hover {
      background-color: #bbb;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2><img src="https://cdn-icons-png.flaticon.com/512/149/149071.png" alt="Usuari">Benvingut, <?= htmlspecialchars($usuari['nom']) ?>!</h2>
    <p>Aquest és el teu espai personal. Aquí podràs modificar les teves dades, consultar els serveis que tens actius i accedir a la configuració de nous servidors.</p>
    <p><strong>Preu total dels teus serveis:</strong> <?= number_format($total, 2) ?> €</p>

    <form action="guardar_dades_usuari.php" method="POST">
      <label for="nom">Nom i cognoms</label>
      <input type="text" id="nom" name="nom" value="<?= htmlspecialchars($usuari['nom']) ?>" required>

      <label for="email">Correu electrònic</label>
      <input type="email" id="email" name="email" value="<?= htmlspecialchars($usuari['email']) ?>" required>

      <label for="telefon">Telèfon</label>
      <input type="tel" id="telefon" name="telefon" value="<?= htmlspecialchars($usuari['telefon']) ?>" required>

      <label for="dni">DNI</label>
      <input type="text" id="dni" name="dni" value="<?= htmlspecialchars($usuari['dni']) ?>" required>

      <label for="nova_contrasenya">Nova contrasenya (opcional)</label>
      <input type="password" id="nova_contrasenya" name="nova_contrasenya" placeholder="Deixa en blanc si no vols canviar-la">

      <button type="submit">Desar canvis</button>
    </form>

    <div class="buttons">
      <a href="gestionar.php"><button type="button">Gestionar servidors</button></a>
      <a href="nou_servei.php"><button type="button">Contractar nou servei</button></a>
    </div>

    <a href="index.php" class="back-button">⬅️ Tancar sessió</a>
  </div>
</body>
</html>

```

## `registre.php`
```php
<?php
session_start();

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $nom = $_POST['nom'] ?? '';
    $email = $_POST['email'] ?? '';
    $telefon = $_POST['telefon'] ?? '';
    $dni = $_POST['dni'] ?? '';
    $contrasenya = $_POST['contrasenya'] ?? '';

    if (empty($nom) || empty($email) || empty($telefon) || empty($dni) || empty($contrasenya)) {
        die("Cal omplir tots els camps.");
    }

    // Encriptar contrasenya
    $hash = password_hash($contrasenya, PASSWORD_DEFAULT);

    // Connexió
    $conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
    if ($conn->connect_error) {
        die("Error de connexió: " . $conn->connect_error);
    }

    // Evitar duplicats
    $check = $conn->prepare("SELECT id FROM usuaris WHERE email = ?");
    $check->bind_param("s", $email);
    $check->execute();
    $check->store_result();

    if ($check->num_rows > 0) {
        $check->close();
        die("a existeix un compte amb aquest correu.");
    }
    $check->close();

    // Inserir nou usuari
    $stmt = $conn->prepare("INSERT INTO usuaris (nom, email, telefon, dni, contrasenya) VALUES (?, ?, ?, ?, ?)");
    $stmt->bind_param("sssss", $nom, $email, $telefon, $dni, $hash);
    $stmt->execute();
    $stmt->close();
    $conn->close();

    // Redirigir a login
    header("Location: login.php?msg=registrat");
    exit;
}
?>

<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="UTF-8">
  <title>Registre d'Usuari</title>
  <style>
    body {
      background-color: #e6f0fa;
      font-family: 'Segoe UI', sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
    }
    .container {
      background-color: #ffffff;
      padding: 40px;
      border-radius: 20px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.1);
      max-width: 400px;
      width: 100%;
      text-align: center;
    }
    h2 {
      color: #2c3e50;
      margin-bottom: 30px;
    }
    input {
      width: 100%;
      padding: 12px;
      margin: 10px 0;
      border: none;
      border-radius: 10px;
      background-color: #f0f4f8;
    }
    button {
      padding: 12px 20px;
      background-color: #4a90e2;
      color: white;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      width: 100%;
    }
    button:hover {
      background-color: #3270c5;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Crear un compte nou</h2>
    <form method="POST">
      <input type="text" name="nom" placeholder="Nom complet" required>
      <input type="email" name="email" placeholder="Correu electrònic" required>
      <input type="text" name="telefon" placeholder="Telèfon" required>
      <input type="text" name="dni" placeholder="DNI / NIE" required>
      <input type="password" name="contrasenya" placeholder="Contrasenya" required>
      <button type="submit">Registra'm</button>
    </form>
    <br>
    <a href="index.php">
      <button style="background-color: #ccc; color: #2c3e50;">← Tornar</button>
    </a>
  </div>
</body>
</html>

```
## `backup_serveis.sh`
```php

#!/bin/bash

BASE_DIR="/var/www/containers"
BACKUP_DIR="/var/backups/serveis"
DATA=$(date +%F_%H-%M)

DB_USER="pau"
DB_PASS="Hola1234!"
DB_NAME="projecte"

mkdir -p $BACKUP_DIR

# Recórrer tots els serveis
for USUARI_DIR in $BASE_DIR/usuari_*; do
  USUARI_ID=$(basename "$USUARI_DIR" | cut -d'_' -f2)

  for SERVEI_DIR in $USUARI_DIR/servei_*; do
    ID_SERVEI=$(basename "$SERVEI_DIR")

    # Comprovem si aquest servei té backup activat
    BACKUP_ACTIU=$(mysql -u"$DB_USER" -p"$DB_PASS" -D "$DB_NAME" -se "SELECT backup_diari FROM serveis WHERE servei_id='$ID_SERVEI' AND usuari_id=$USUARI_ID;" | xargs)

    if [ "$BACKUP_ACTIU" = "1" ]; then
      echo "Fent backup de $ID_SERVEI per a usuari $USUARI_ID..."

      DB_NAME="db_$ID_SERVEI"
      OUTDIR="$BACKUP_DIR/$ID_SERVEI/$DATA"
      mkdir -p "$OUTDIR"

      # Dump de base de dades
      docker exec servei_${ID_SERVEI}_db-1 mysqldump -u user -ppass $DB_NAME > "$OUTDIR/db.sql" 2>/dev/null

      # Còpia de fitxers web
      cp -r "$SERVEI_DIR/app_data" "$OUTDIR/"
      echo "Backup completat: $OUTDIR"
    else
      echo "altant $ID_SERVEI (backup no actiu)"
    fi
  done
done

```
