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

$conn = new mysqli("localhost", "pau", "Hola1234!", "projecte");
if ($conn->connect_error) {
    die("Error de connexió: " . $conn->connect_error);
}

$stmt = $conn->prepare("SELECT plataforma FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$result = $stmt->get_result();
if ($result->num_rows === 0) {
    die("El servei no existeix.");
}
$servei = $result->fetch_assoc();
$stmt->close();

$base_path = "/var/www/containers/usuari_$usuari_id/$servei_id";
shell_exec("cd $base_path && docker compose down");
shell_exec("sudo rm -rf " . escapeshellarg($base_path));

$stmt = $conn->prepare("DELETE FROM serveis WHERE servei_id = ? AND usuari_id = ?");
$stmt->bind_param("si", $servei_id, $usuari_id);
$stmt->execute();
$stmt->close();

$conn->close();
header("Location: gestionar.php");
exit;
?>
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

// (Aquí van totes les variables recollides del POST...)

if ($estat_actual !== 'aturat') {
    die("Primer has d’aturar el servei abans de modificar-lo. <br><br><a href='gestionar.php'>Tornar</a>");
}

// (Lògica per preparar directoris, càlculs, validacions...)

file_put_contents("$base_path/docker-compose.yml", $compose);

header("Location: gestionar.php?msg=canvis_guardats");
exit;
?>
---
