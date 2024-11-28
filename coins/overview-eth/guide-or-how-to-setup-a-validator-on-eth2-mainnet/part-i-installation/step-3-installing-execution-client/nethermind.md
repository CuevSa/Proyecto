# Nethermind

## Descipción

{% hint style="info" %}
**Nethermind** es un cliente insignia de Ethereum que se centra en el rendimiento y la flexibilidad. Creado sobre **.NET** core, una plataforma generalizada y amigable para las empresas, Nethermind simplifica la integración con las infraestructuras existentes, sin perder de vista la estabilidad, confiabilidad, integridad de los datos y seguridad.
{% endhint %}

#### Enlaces Oficiales

| El Tema       | Enlace                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| Lanzamientos  | [https://github.com/NethermindEth/nethermind/releases](https://github.com/NethermindEth/nethermind/releases) |
| Documentación | [https://docs.nethermind.io](https://docs.nethermind.io/)                                                    |
| Sitio Web     | [https://nethermind.io/nethermind-client](https://nethermind.io/nethermind-client/)                          |

### 1. Configuración inicial

Cree un usuario de servicio para el servicio de ejecución, cree un directorio de datos y asigne la propiedad.
```bash
sudo adduser --system --no-create-home --group execution
sudo mkdir -p /var/lib/nethermind
sudo chown -R execution:execution /var/lib/nethermind
```

Instalar dependencias.

```bash
sudo apt update
sudo apt install ccze curl libsnappy-dev libc6-dev jq libc6 unzip -y
```

### 2. Instalar binarios

* La descarga de archivos binarios suele ser más rápida y cómoda.&#x20;
* Construir a partir de código fuente puede ofrecer una mejor compatibilidad y está más alineado con el espíritu de FOSS (software gratuito de código abierto).

<details>

<summary>Option 1 - Download binaries</summary>

Ejecute lo siguiente para descargar automáticamente la última versión de Linux, descomprimirla y limpiarla.

```bash
RELEASE_URL="https://api.github.com/repos/NethermindEth/nethermind/releases/latest"
BINARIES_URL="$(curl -s $RELEASE_URL | jq -r ".assets[] | select(.name) | .browser_download_url" | grep linux-x64)"

echo Descargando URL: $BINARIES_URL

cd $HOME
wget -O nethermind.zip $BINARIES_URL
unzip -o nethermind.zip -d $HOME/nethermind
rm nethermind.zip
```

Instale los binarios.

<pre class="language-bash"><code class="lang-bash"><strong>sudo mv $HOME/nethermind /usr/local/bin/nethermind
</strong></code></pre>

</details>

<details>

<summary>Option 2 - Build from source code</summary>

Instale las dependencias de compilación del SDK de .NET.

```bash
# Get Ubuntu version
declare repo_version=$(if command -v lsb_release &> /dev/null; then lsb_release -r -s; else grep -oP '(?<=^VERSION_ID=).+' /etc/os-release | tr -d '"'; fi)

# Descargue la clave de firma y el repositorio de Microsoft
wget https://packages.microsoft.com/config/ubuntu/$repo_version/packages-microsoft-prod.deb -O packages-microsoft-prod.deb

# Instalar la clave de firma y el repositorio de Microsoft
sudo dpkg -i packages-microsoft-prod.deb

# Limpiar
rm packages-microsoft-prod.deb

# Paquetes de actualización
sudo apt-get update && sudo apt-get install -y dotnet-sdk-8.0
```

Construir los binarios.

```bash
mkdir -p ~/git
cd ~/git
# Clone the repo
git clone https://github.com/NethermindEth/nethermind.git
cd nethermind
# Get new tags
git fetch --tags
# Get latest tag name
latestTag=$(git describe --tags `git rev-list --tags --max-count=1`)
# Checkout latest tag
git checkout $latestTag
# Build
dotnet publish src/Nethermind/Nethermind.Runner -c release -o nethermind
```

Verifique que Nethermind se haya creado correctamente comprobando la versión.

```shell
./nethermind/nethermind --version
```

Salida de muestra de una versión compatible.

```
Version: 1.25.2+78c7bf5f
Commit: 78c7bf5f2c0819f23e248ee6d108c17cd053ffd3
Build Date: 2024-01-23 06:34:53Z
OS: Linux x64
Runtime: .NET 8.0.1
```

Instale los binarios.

<pre class="language-shell"><code class="lang-shell"><strong>sudo mv $HOME/git/nethermind/nethermind /usr/local/bin
</strong></code></pre>

</details>

### **3. Instalar y configurar systemd**

Cree un **archivo de unidad systemd** para definir su configuración `execution.service`.

```bash
sudo nano /etc/systemd/system/execution.service
```

Pegue la siguiente configuración en el archivo.

```shell
[Unit]
Description=Nethermind Execution Layer Client service for Mainnet
Wants=network-online.target
After=network-online.target
Documentation=https://www.coincashew.com

[Service]
Type=simple
User=execution
Group=execution
Restart=always
RestartSec=3
KillSignal=SIGINT
TimeoutStopSec=900
WorkingDirectory=/var/lib/nethermind
Environment="DOTNET_BUNDLE_EXTRACT_BASE_DIR=/var/lib/nethermind"
ExecStart=/usr/local/bin/nethermind/nethermind \
  --config mainnet \
  --datadir="/var/lib/nethermind" \
  --Network.DiscoveryPort 30303 \
  --Network.P2PPort 30303 \
  --Network.MaxActivePeers 50 \
  --JsonRpc.Port 8545 \
  --JsonRpc.EnginePort 8551 \
  --Metrics.Enabled true \
  --Metrics.ExposePort 6060 \
  --JsonRpc.JwtSecretFile /secrets/jwtsecret \
  --Pruning.Mode=Hybrid \
  --Pruning.FullPruningTrigger=VolumeFreeSpace \
  --Pruning.FullPruningThresholdMb=300000
  
[Install]
WantedBy=multi-user.target
```

{% hint style="info" %}
Nethermind podará la base de datos cuando el espacio en disco sea bajo (por debajo de 300 GB)
{% endhint %}

Para salir y guardar, presione `Ctrl` + `X`, luego `Y`, luego `Enter`.

Run the following to enable auto-start at boot time.

Ejecute lo siguiente para habilitar el inicio automático en el momento del arranque.

```bash
sudo systemctl daemon-reload
sudo systemctl enable execution
```

Finalmente, inicie su cliente de capa de ejecución y verifique su estado.

```bash
sudo systemctl start execution
sudo systemctl status execution
```

Presione `Ctrl` + `C` para salir del estado.

### 4. Comandos útiles del cliente de ejecución

{% tabs %}
{% tab title="View Logs" %}
```bash
sudo journalctl -fu execution | ccze
```

Un cliente de ejecución **Nethermind** que funcione correctamente indicará "Nuevo bloque recibido". Por ejemplo,

```
Nethermind.Runner[2]: 29 Sep 03:00:00 | Received new block:  8372 (0x425ab9...854f4)
Nethermind.Runner[2]: 29 Sep 03:00:00 | Processed                8372     |      0.17 ms  |  slot     13,001 ms |
Nethermind.Runner[2]: 29 Sep 03:00:00 | - Block               0.00 MGas   |      0    txs |  calls      0 (  0) | sload       0 | sstore      0 | create   0
Nethermind.Runner[2]: 29 Sep 03:00:00 | - Block throughput    0.00 MGas/s |      0.00 t/s |       7217.16 Blk/s | recv        0 | proc        0
Nethermind.Runner[2]: 29 Sep 03:00:00 | Received ForkChoice: Head: 8372 (0x425ab9...854f4), Safe: 8350 (0xfd781...c2e19f), Finalized: 8332 (0x9ccf...88684c)
Nethermind.Runner[2]: 29 Sep 03:00:00 | Synced chain Head to 8372 (0x425ab9...2881a5)
```
{% endtab %}

{% tab title="Stop" %}
```bash
sudo systemctl stop execution
```
{% endtab %}

{% tab title="Start" %}
```bash
sudo systemctl start execution
```
{% endtab %}

{% tab title="View Status" %}
```bash
sudo systemctl status execution
```
{% endtab %}

{% tab title="Reset Database" %}
Las razones comunes para restablecer la base de datos pueden incluir:

* Recuperación de una base de datos dañada debido a un corte de energía o una falla de hardware
* Resincronización para reducir el uso de espacio en disco
* Actualización a un nuevo formato de almacenamiento

```bash
sudo systemctl stop execution
sudo rm -rf /var/lib/nethermind/*
sudo systemctl restart execution
```

El tiempo para volver a sincronizar el cliente de ejecución puede tardar desde algunas horas hasta un día.
{% endtab %}
{% endtabs %}

Ahora que su cliente de ejecución está configurado e iniciado, continúe con el siguiente paso para configurar su cliente de consenso.

{% hint style="warning" %}
Si está revisando los registros y ve alguna advertencia o error, tenga paciencia, ya que normalmente se resolverán una vez que tanto su cliente de ejecución como el de consenso estén completamente sincronizados con la red Ethereum.
{% endhint %}
