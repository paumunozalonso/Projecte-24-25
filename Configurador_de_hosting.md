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
