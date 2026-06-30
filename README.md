# NetSimon Device Check — Sistema de Bloqueio de Compartilhamento de Credenciais

Sistema de anti-compartilhamento para o app NetSimon que bloqueia automaticamente acessos em múltiplos dispositivos além do limite definido, com log de comportamento e notificação ao usuário.

---

## 🚀 Instalação Rápida

Execute em qualquer servidor Ubuntu/Debian como root:

```bash
sudo bash <(curl -s https://raw.githubusercontent.com/miau4/script-de-instala-o-do-bloqueio-de-usuarios-no-app/refs/heads/main/install)
```

O script instalará e configurará automaticamente todos os componentes necessários.

---


Se for deletar apenas o script:

```bash
bashrm -f /var/www/html/device_check.php
```

Se for remover a instalação completa do script de bloqueio:

```bash
bashrm -rf /var/www/html/netsimon_devices.db /var/www/html/netsimon_device_log.txt /var/www/h
```

## 📋 O que é instalado

- **Nginx** — Servidor web na porta 81
- **PHP 8.1 + PHP-FPM** — Processamento de requisições
- **SQLite3** — Banco de dados de dispositivos
- **cURL** — Requisições HTTP para o painel DragonCore
- **Script `device_check.php`** — Endpoint de verificação
- **Banco de dados SQLite** com tabelas de controle e log
- **Arquivo de log** para análise de comportamento

---

## 🔧 Como funciona

### Fluxo de verificação

```
Usuário toca LIGAR no app
    ↓
App envia: username + device_hash + phone_number
    ↓
Servidor verifica no painel DragonCore via API
    ↓
Servidor compara com dispositivos registrados no SQLite
    ↓
Se dispositivo novo e dentro do limite → registra e AUTORIZA
Se dispositivo novo e limite atingido → BLOQUEIA
Se dispositivo já registrado → AUTORIZA
    ↓
App recebe resposta:
  ✓ allowed: continua conexão normalmente
  ✗ blocked: exibe popup vermelho com botão de suporte WhatsApp
```

### Identificação de dispositivo

O sistema usa um hash único por dispositivo gerado a partir de:
- **ANDROID_ID** — identificador único do Android
- **Modelo do aparelho** — ex: Samsung Galaxy S21
- **Versão do Android** — ex: 13

Isso gera um hash SHA-256 que não muda mesmo com reinstalações do app ou roaming entre operadoras.

---

## 📱 Integração no App

### 1. Kotlin (WebUiActivity.kt)

Adicionar na `inner class AndroidBridge`:

```kotlin
@SuppressLint("MissingPermission")
@JavascriptInterface
fun getDeviceHash(): String {
    return try {
        val androidId = Settings.Secure.getString(
            contentResolver,
            Settings.Secure.ANDROID_ID
        ) ?: "unknown"
        val model   = android.os.Build.MODEL   ?: "unknown"
        val version = android.os.Build.VERSION.SDK_INT.toString()
        val raw     = "$androidId|$model|$version"
        java.security.MessageDigest.getInstance("SHA-256")
            .digest(raw.toByteArray())
            .joinToString("") { "%02x".format(it) }
            .take(32)
    } catch (e: Exception) {
        Log.e("WebUiActivity", "getDeviceHash: ${e.message}")
        "unknown_${System.currentTimeMillis()}"
    }
}

@SuppressLint("MissingPermission")
@JavascriptInterface
fun getPhoneNumber(): String {
    return try {
        if (checkSelfPermission(android.Manifest.permission.READ_PHONE_STATE)
            == android.content.pm.PackageManager.PERMISSION_GRANTED
        ) {
            val tm = getSystemService(Context.TELEPHONY_SERVICE) as android.telephony.TelephonyManager
            tm.line1Number?.takeIf { it.isNotBlank() } ?: ""
        } else ""
    } catch (_: Exception) { "" }
}
```

### 2. JavaScript (index.html)

Adicionar antes do botão LIGAR/PARAR:

```javascript
var deviceCheckUrl = 'http://2.netsimon.fun:81/device_check.php';

function getDeviceIdentifier() {
    if (window.Android && Android.getDeviceHash) {
        return Android.getDeviceHash();
    }
    var stored = storageGet('_device_id');
    if (!stored) {
        stored = Math.random().toString(36).substring(2) + Date.now().toString(36);
        storageSave('_device_id', stored);
    }
    return stored;
}

function checkDeviceAccess(username, onAllowed) {
    var deviceHash = getDeviceIdentifier();
    var phone = (window.Android && Android.getPhoneNumber) ? Android.getPhoneNumber() : '';

    var xhr = new XMLHttpRequest();
    xhr.open('POST', deviceCheckUrl, true);
    xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    xhr.timeout = 10000;

    xhr.onload = function () {
        try {
            var data = JSON.parse(xhr.responseText);
            if (data.status === 'blocked') {
                showDeviceBlocked(data.message || 'Acesso bloqueado.', data.limit || 1);
            } else {
                onAllowed();
            }
        } catch (e) {
            onAllowed();
        }
    };

    xhr.onerror = function () { onAllowed(); };
    xhr.ontimeout = function () { onAllowed(); };

    xhr.send(
        'username=' + encodeURIComponent(username) +
        '&device_hash=' + encodeURIComponent(deviceHash) +
        '&phone=' + encodeURIComponent(phone)
    );
}

function showDeviceBlocked(message, limit) {
    var old = document.getElementById('device-blocked-overlay');
    if (old) old.remove();

    var overlay = document.createElement('div');
    overlay.id = 'device-blocked-overlay';
    overlay.style.cssText = 'display:flex;position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.88);z-index:99999;align-items:center;justify-content:center;';

    overlay.innerHTML =
        '<div style="background:#1a0000;border:2px solid #ff2222;border-radius:16px;padding:32px 24px;max-width:320px;width:90%;text-align:center;box-shadow:0 0 40px rgba(255,0,0,0.4);">' +
            '<div style="font-size:48px;margin-bottom:12px;">🚫</div>' +
            '<div style="color:#ff3333;font-size:20px;font-weight:bold;margin-bottom:12px;">ACESSO BLOQUEADO</div>' +
            '<div style="color:#ffaaaa;font-size:14px;line-height:1.5;margin-bottom:8px;">' + (message || 'Este dispositivo não está autorizado.') + '</div>' +
            '<div style="color:#ff6666;font-size:12px;margin-bottom:16px;">Limite de ' + (limit || 1) + ' dispositivo(s) por conta.</div>' +
            '<div style="color:#aaa;font-size:12px;line-height:1.6;margin-bottom:20px;">' +
                'Acredita que isso é um erro ou deseja contratar um novo acesso?<br>' +
                'Entre em contato com o suporte:' +
            '</div>' +
            '<a href="https://wa.me/5511997675068" target="_blank" ' +
                'style="display:flex;align-items:center;justify-content:center;gap:8px;background:#25D366;color:#fff;text-decoration:none;border-radius:8px;padding:12px 20px;font-size:14px;font-weight:bold;margin-bottom:12px;">' +
                '<span style="font-size:20px;">💬</span> FALAR COM SUPORTE' +
            '</a>' +
            '<button onclick="document.getElementById(\'device-blocked-overlay\').remove()" ' +
                'style="background:transparent;color:#888;border:1px solid #444;border-radius:8px;padding:10px 32px;font-size:13px;cursor:pointer;width:100%;">FECHAR</button>' +
        '</div>';

    document.body.appendChild(overlay);

    var btn = document.getElementById('btn-connection');
    var sl  = document.getElementById('status-label');
    if (btn) { btn.innerText = 'LIGAR'; btn.disabled = false; btn.classList.remove('connecting','active'); }
    if (sl)  { sl.innerText = 'BLOQUEADO'; sl.classList.add('error-text'); sl.classList.remove('connected','warn-text'); }
    isSshConnecting = false;
}
```

Depois, no botão LIGAR/PARAR, envolver as chamadas de conexão com:

```javascript
// Para SSH
checkDeviceAccess(sshUser, function () {
    // código de conexão aqui
    Android.startSshWithCredentials(selectedOpName, sshUser, sshPass);
});

// Para V2Ray (UUID)
checkDeviceAccess(uuid, function () {
    // código de conexão aqui
    Android.startTunnelWithCreds(uuid, sshUser, sshPass);
});
```

---

## 🛠️ Gerenciamento

### Ver logs em tempo real

```bash
tail -f /var/www/html/netsimon_device_log.txt
```

### Resetar dispositivos de um usuário

```bash
sqlite3 /var/www/html/netsimon_devices.db "DELETE FROM devices WHERE username='USUARIO';"
```

### Ver todos os dispositivos registrados

```bash
sqlite3 /var/www/html/netsimon_devices.db "SELECT username, device_hash, phone, first_seen, last_seen FROM devices;"
```

### Ver log de bloqueios

```bash
sqlite3 /var/www/html/netsimon_devices.db "SELECT * FROM device_log WHERE action='BLOCKED';"
```

---

## 📊 Estrutura do banco de dados

### Tabela `devices`
```sql
id              — ID único
username        — Login do usuário
device_hash     — Hash SHA-256 do dispositivo
phone           — Número do SIM (quando disponível)
ip              — IP de origem
first_seen      — Data/hora do primeiro acesso
last_seen       — Data/hora do último acesso
```

### Tabela `device_log`
```sql
id              — ID único
username        — Login do usuário
device_hash     — Hash do dispositivo
phone           — Número do SIM
ip              — IP de origem
action          — ALLOWED | BLOCKED | NEW_DEVICE
reason          — Motivo da ação
created_at      — Data/hora do evento
```

---

## 🔐 Segurança

- ✅ Verificação de permissão em tempo de execução para `READ_PHONE_STATE`
- ✅ Falha aberta (fail-open) — problemas de rede não prejudicam usuários legítimos
- ✅ CORS habilitado para requisições do app
- ✅ Log completo de todas as tentativas para análise de comportamento
- ✅ Limite configurável por usuário no painel DragonCore

---

## 🚦 Próximos passos

### Quando conseguir credenciais SQL do painel DragonCore

Reescrever `device_check.php` para usar os campos nativos `deviceid` e `deviceativo` da tabela de usuários, eliminando o SQLite separado e integrando nativamente com o botão "Limpar Device ID" do painel.

### Adicionar painel de admin

Criar interface em `2.netsimon.fun:81/admin` para:
- Visualizar dispositivos por usuário
- Resetar dispositivos manualmente
- Ver histórico de bloqueios
- Exportar relatórios de comportamento

---

## 📝 Changelog

### v1.0 (Inicial)
- Sistema completo de bloqueio por dispositivo
- Logging em SQLite e arquivo de texto
- Popup de bloqueio com contato de suporte
- Identificação confiável por hash ANDROID_ID + modelo + versão

---

## 💬 Suporte

Para dúvidas sobre a instalação ou integração:  
📱 **WhatsApp**: [wa.me/5511997675068](https://wa.me/5511997675068)

---

**NetSimon © 2026**
