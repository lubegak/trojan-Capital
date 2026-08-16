# trojan-Capital
# Trojan Capital Website

A professional PHP/MySQL starter website for Trojan Capital.

## Included
- Premium responsive homepage using the supplied Trojan Capital logo.
- Capital-market sections: FX, Rates & Bonds, Equities, Commodities.
- Research/forecasting section ready for your macro modules.
- Insights/blog system backed by MySQL.
- Member registration and secure password hashing.
- Member sign-in.
- Private member messaging.
- Dashboard showing sent messages.
- Admin-ready database field for future post publishing.
- CSRF protection, prepared SQL statements and secure sessions.

## Hostinger deployment
1. Create a MySQL database in Hostinger.
2. Import `schema.sql` using phpMyAdmin.
3. Edit `config.php` with the database credentials and your real email address.
4. Upload all files/folders to `public_html`.
5. Visit the domain.
6. Create your account. To make yourself an admin for the future publishing dashboard, set `is_admin = 1` for your user in the `users` table.

## Important production upgrade
The included `mail()` call is a simple notification fallback. For reliable delivery, connect SMTP (for example through your domain email provider) before launch. Do not put database passwords or SMTP passwords into public GitHub repositories.

## Next development
- Admin dashboard to publish/edit/delete market posts.
- Rich text editor and image uploads for posts.
- Market data APIs and live charts.
- Email verification and password reset.
- Rate limiting / CAPTCHA.
- Professional SMTP service.
- Privacy policy, terms and cookie controls.
- Analytics and SEO/structured data.

<?php
// Copy this file to your live server and replace the values below.
define('DB_HOST', 'localhost');
define('DB_NAME', 'trojan_capital');
define('DB_USER', 'YOUR_DATABASE_USER');
define('DB_PASS', 'YOUR_DATABASE_PASSWORD');

define('ADMIN_EMAIL', 'YOUR_EMAIL@example.com');
define('SITE_NAME', 'Trojan Capital');

ini_set('session.cookie_httponly', '1');
ini_set('session.cookie_samesite', 'Lax');
if (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') {
    ini_set('session.cookie_secure', '1');
}

<?php
require_once __DIR__ . '/config.php';

try {
    $pdo = new PDO(
        "mysql:host=".DB_HOST.";dbname=".DB_NAME.";charset=utf8mb4",
        DB_USER,
        DB_PASS,
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES => false
        ]
    );
} catch (Throwable $e) {
    $pdo = null;
}
function csrf_token(): string {
    if (empty($_SESSION['csrf'])) $_SESSION['csrf'] = bin2hex(random_bytes(32));
    return $_SESSION['csrf'];
}
function verify_csrf(): void {
    if (!isset($_POST['csrf']) || !hash_equals($_SESSION['csrf'] ?? '', $_POST['csrf'])) {
        http_response_code(403); exit('Invalid security token.');
    }
}

<?php
session_start();
require_once __DIR__.'/db.php';
if (!$pdo) exit('Database is not configured yet.');
verify_csrf();

$action = $_POST['action'] ?? '';
$email = strtolower(trim($_POST['email'] ?? ''));
$password = $_POST['password'] ?? '';

if (!filter_var($email, FILTER_VALIDATE_EMAIL) || strlen($password) < 8) exit('Please provide valid account details.');

if ($action === 'register') {
    $name = trim($_POST['name'] ?? '');
    if (strlen($name) < 2) exit('Please enter your full name.');
    $stmt = $pdo->prepare("SELECT id FROM users WHERE email=?");
    $stmt->execute([$email]);
    if ($stmt->fetch()) exit('An account with this email already exists.');
    $hash = password_hash($password, PASSWORD_DEFAULT);
    $stmt = $pdo->prepare("INSERT INTO users (name,email,password_hash) VALUES (?,?,?)");
    $stmt->execute([$name,$email,$hash]);
    $_SESSION['user'] = ['id'=>$pdo->lastInsertId(),'name'=>$name,'email'=>$email,'is_admin'=>0];
    header('Location: index.php#contact'); exit;
}

if ($action === 'login') {
    $stmt = $pdo->prepare("SELECT id,name,email,password_hash,is_admin FROM users WHERE email=?");
    $stmt->execute([$email]);
    $u = $stmt->fetch();
    if (!$u || !password_verify($password, $u['password_hash'])) exit('Incorrect email or password.');
    session_regenerate_id(true);
    $_SESSION['user'] = ['id'=>$u['id'],'name'=>$u['name'],'email'=>$u['email'],'is_admin'=>(int)$u['is_admin']];
    header('Location: index.php'); exit;
}
exit('Invalid action.');

<?php
session_start();
session_unset();
session_destroy();
header('Location: index.php');
exit;

<?php
session_start();
require_once __DIR__.'/db.php';
if (empty($_SESSION['user'])) { header('Location: index.php'); exit; }
$user = $_SESSION['user'];
$messages = [];
if ($pdo) {
    $stmt = $pdo->prepare("SELECT subject,message,status,created_at FROM messages WHERE user_id=? ORDER BY created_at DESC");
    $stmt->execute([$user['id']]);
    $messages = $stmt->fetchAll();
}
?>
<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Member Dashboard — Trojan Capital</title><link rel="stylesheet" href="assets/styles.css"></head>
<body class="dashboard-body">
<header class="site-header"><div class="container nav-wrap"><a class="brand" href="index.php"><img src="assets/trojan-capital-logo.png"><span>Trojan Capital</span></a><nav class="nav"><a href="index.php#posts">Insights</a><a href="index.php#contact">Contact</a><a class="btn btn-gold btn-small" href="logout.php">Sign out</a></nav></div></header>
<main class="dashboard"><div class="container">
<div class="dashboard-head"><div><div class="eyebrow"><span></span> MEMBER AREA</div><h1>Welcome, <?= htmlspecialchars($user['name']) ?>.</h1><p>Your private space for communicating with Trojan Capital.</p></div><div class="member-badge">VERIFIED MEMBER<br><strong><?= htmlspecialchars($user['email']) ?></strong></div></div>
<?php if (isset($_GET['sent'])): ?><div class="notice">Your message has been sent successfully.</div><?php endif; ?>
<div class="dashboard-grid">
<div class="dash-card"><h2>Send a message</h2><form action="send_message.php" method="post"><input type="hidden" name="csrf" value="<?= htmlspecialchars(csrf_token()) ?>"><label>Subject<input name="subject" required maxlength="160"></label><label>Message<textarea name="message" rows="8" required maxlength="5000"></textarea></label><button class="btn btn-gold">Send message</button></form></div>
<div class="dash-card"><h2>Your messages</h2><?php if (!$messages): ?><p class="muted">No messages yet.</p><?php else: foreach ($messages as $m): ?><div class="message-row"><span class="message-status"><?= htmlspecialchars($m['status']) ?></span><strong><?= htmlspecialchars($m['subject']) ?></strong><small><?= date('M j, Y H:i', strtotime($m['created_at'])) ?></small><p><?= nl2br(htmlspecialchars($m['message'])) ?></p></div><?php endforeach; endif; ?></div>
</div></div></main></body></html>

<?php
require_once __DIR__.'/db.php';
$id = (int)($_GET['id'] ?? 0);
$post = null;
if ($pdo) {
    $stmt = $pdo->prepare("SELECT * FROM posts WHERE id=?");
    $stmt->execute([$id]); $post = $stmt->fetch();
}
if (!$post) { http_response_code(404); exit('Post not found.'); }
?>
<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title><?= htmlspecialchars($post['title']) ?> — Trojan Capital</title><link rel="stylesheet" href="assets/styles.css"></head>
<body><header class="site-header"><div class="container nav-wrap"><a class="brand" href="index.php"><img src="assets/trojan-capital-logo.png"><span>Trojan Capital</span></a><nav class="nav"><a href="index.php#posts">← All insights</a><a href="index.php#contact">Contact</a></nav></div></header>
<main class="article"><div class="container article-wrap"><div class="eyebrow"><span></span> <?= htmlspecialchars($post['category']) ?></div><h1><?= htmlspecialchars($post['title']) ?></h1><div class="article-meta">By <?= htmlspecialchars($post['author']) ?> · <?= date('F j, Y', strtotime($post['created_at'])) ?></div><div class="article-body"><?= nl2br(htmlspecialchars($post['body'])) ?></div></div></main></body></html>

<?php
session_start();
require_once __DIR__.'/db.php';
if (!$pdo || empty($_SESSION['user'])) exit('Please sign in first.');
verify_csrf();

$subject = trim($_POST['subject'] ?? '');
$message = trim($_POST['message'] ?? '');
if ($subject === '' || $message === '') exit('Please complete the message.');
if (mb_strlen($subject) > 160 || mb_strlen($message) > 5000) exit('Message is too long.');

$stmt = $pdo->prepare("INSERT INTO messages (user_id,subject,message) VALUES (?,?,?)");
$stmt->execute([$_SESSION['user']['id'],$subject,$message]);

// Optional server-side email notification. Configure SMTP for production.
$headers = "Reply-To: ".$_SESSION['user']['email']."\r\n";
@mail(ADMIN_EMAIL, "Trojan Capital: ".$subject, $message, $headers);

header('Location: dashboard.php?sent=1');
exit;

CREATE DATABASE IF NOT EXISTS trojan_capital CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE trojan_capital;

CREATE TABLE users (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(190) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  is_admin TINYINT(1) NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  category VARCHAR(80) NOT NULL,
  excerpt VARCHAR(500) NOT NULL,
  body TEXT NOT NULL,
  author VARCHAR(120) NOT NULL DEFAULT 'Trojan Capital Research',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE messages (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id INT UNSIGNED NOT NULL,
  subject VARCHAR(160) NOT NULL,
  message TEXT NOT NULL,
  status ENUM('new','read','replied') NOT NULL DEFAULT 'new',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

INSERT INTO posts (title,category,excerpt,body) VALUES
('Building a data-driven view of inflation','MACRO','A framework for organizing inflation data into market intelligence.','This is a starter research post. Replace it with your own analysis, charts, sources and conclusions.'),
('How macro surprises can move currencies','FX','A practical framework for thinking about economic surprises and FX reactions.','This is a starter research post. Replace it with your own analysis and examples.'),
('Reading the yield curve','RATES','Understand the relationship between yields, inflation expectations and policy.','This is a starter research post. Replace it with your own analysis and charts.');

const modal = document.getElementById('authModal');
const openers = document.querySelectorAll('[data-open-auth]');
const closers = document.querySelectorAll('[data-close-auth]');
const tabs = document.querySelectorAll('[data-tab]');
const login = document.getElementById('loginForm');
const register = document.getElementById('registerForm');
const navToggle = document.querySelector('.nav-toggle');
const nav = document.querySelector('.nav');

openers.forEach(b => b.addEventListener('click', () => { modal.classList.add('open'); modal.setAttribute('aria-hidden','false'); }));
closers.forEach(b => b.addEventListener('click', () => { modal.classList.remove('open'); modal.setAttribute('aria-hidden','true'); }));
tabs.forEach(t => t.addEventListener('click', () => {
  tabs.forEach(x => x.classList.remove('active')); t.classList.add('active');
  const reg = t.dataset.tab === 'register';
  login.classList.toggle('hidden', reg); register.classList.toggle('hidden', !reg);
}));
document.addEventListener('keydown', e => { if (e.key === 'Escape') modal?.classList.remove('open'); });
navToggle?.addEventListener('click', () => nav.classList.toggle('mobile-open'));
nav?.querySelectorAll('a').forEach(a => a.addEventListener('click', () => nav.classList.remove('mobile-open')));

<?php
session_start();
require_once __DIR__.'/db.php';
if (!$pdo) exit('Database is not configured yet.');
verify_csrf();

$action = $_POST['action'] ?? '';
$email = strtolower(trim($_POST['email'] ?? ''));
$password = $_POST['password'] ?? '';

if (!filter_var($email, FILTER_VALIDATE_EMAIL) || strlen($password) < 8) exit('Please provide valid account details.');

if ($action === 'register') {
    $name = trim($_POST['name'] ?? '');
    if (strlen($name) < 2) exit('Please enter your full name.');
    $stmt = $pdo->prepare("SELECT id FROM users WHERE email=?");
    $stmt->execute([$email]);
    if ($stmt->fetch()) exit('An account with this email already exists.');
    $hash = password_hash($password, PASSWORD_DEFAULT);
    $stmt = $pdo->prepare("INSERT INTO users (name,email,password_hash) VALUES (?,?,?)");
    $stmt->execute([$name,$email,$hash]);
    $_SESSION['user'] = ['id'=>$pdo->lastInsertId(),'name'=>$name,'email'=>$email,'is_admin'=>0];
    header('Location: index.php#contact'); exit;
}

if ($action === 'login') {
    $stmt = $pdo->prepare("SELECT id,name,email,password_hash,is_admin FROM users WHERE email=?");
    $stmt->execute([$email]);
    $u = $stmt->fetch();
    if (!$u || !password_verify($password, $u['password_hash'])) exit('Incorrect email or password.');
    session_regenerate_id(true);
    $_SESSION['user'] = ['id'=>$u['id'],'name'=>$u['name'],'email'=>$u['email'],'is_admin'=>(int)$u['is_admin']];
    header('Location: index.php'); exit;
}
exit('Invalid action.');

<?php
session_start();
session_unset();
session_destroy();
header('Location: index.php');
exit;