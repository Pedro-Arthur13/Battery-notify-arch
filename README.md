
# **README: Notificações de Bateria no Linux**

<img src="https://i.pinimg.com/originals/71/05/ea/7105eaad76922b7e0f5fe4f5b0ba71de.gif" alt="Yokuso" width="1050" height="auto"/>

Este projeto configura notificações de bateria em um sistema Linux utilizando `cron`, `acpi`, e `notify-send`. O objetivo é fornecer notificações quando a bateria estiver baixa, carregada ou desconectada.

### **Dependências**
1. **`cronie`** - O cron job, responsável por executar comandos periodicamente.
2. **`dunst`** - Gerenciador de notificações leve para exibir as notificações.
3. **`acpi`** - Ferramenta para obter informações sobre a bateria do laptop.

### **Instalação das Dependências**

Para que tudo funcione corretamente, você precisará instalar algumas dependências. Siga os passos abaixo:

#### 1. **Instalar o `cronie` (Cron Jobs)**

O `cronie` é responsável por executar os comandos periodicamente.

```bash
sudo pacman -S cronie
```

#### 2. **Instalar o `dunst` (Gerenciador de Notificações)**

O `dunst` gerencia as notificações na sua área de trabalho.

```bash
sudo pacman -S dunst
```

#### 3. **Instalar o `acpi` (Informações sobre Bateria)**

A ferramenta `acpi` é usada para obter informações sobre o status da bateria.

```bash
sudo pacman -S acpi
```

### **Configuração do `cronie`**

#### 1. **Iniciar e Habilitar o `cronie`**

Para que os cron jobs sejam executados corretamente, é necessário ativar o serviço do `cronie`.

```bash
sudo systemctl start cronie
sudo systemctl enable cronie
```

#### 2. **Verificar se o `cronie` está funcionando**

Verifique o status do `cronie`:

```bash
systemctl status cronie
```

### **Arquivos de Comando**

Os arquivos de comando `batterynotify` e `chargingnotify` são usados para monitorar o status da bateria e enviar notificações.

#### 1. **`batterynotify`**

O script `batterynotify` é responsável por enviar notificações quando a bateria está baixa ou totalmente carregada.

```bash
#!/bin/sh

# Send a notification if the laptop battery is either low or is fully charged.
# Set on a systemd timer (~/.config/systemd/user/battery-alert.timer).

export DISPLAY=:0
export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"

# Battery percentage at which to notify
WARNING_LEVEL=25
# CRITICAL_LEVEL=5
BATTERY_DISCHARGING=`acpi -b | grep "Battery 0" | grep -c "Discharging"`
BATTERY_LEVEL=`acpi -b | grep "Battery 0" | grep -P -o '[0-9]+(?=%)'`

# Use files to store whether we've shown a notification or not (to prevent multiple notifications)
FULL_FILE=/tmp/batteryfull
EMPTY_FILE=/tmp/batteryempty
# CRITICAL_FILE=/tmp/battery-alert

# Reset notifications if the computer is charging/discharging
if [ "$BATTERY_DISCHARGING" -eq 1 ] && [ -f $FULL_FILE ]; then
	rm $FULL_FILE
elif [ "$BATTERY_DISCHARGING" -eq 0 ] && [ -f $EMPTY_FILE ]; then
	rm $EMPTY_FILE
fi

# If the battery is charging and is full (and has not shown notification yet)
if [ "$BATTERY_LEVEL" -gt 95 ] && [ "$BATTERY_DISCHARGING" -eq 0 ] && [ ! -f $FULL_FILE ]; then
	/usr/bin/notify-send "Battery Charged" "Battery is fully charged." -i ~/.local/bin/battery-resized.png -r 9991
	touch $FULL_FILE
	# If the battery is low and is not charging (and has not shown notification yet)
elif [ "$BATTERY_LEVEL" -le $WARNING_LEVEL ] && [ "$BATTERY_DISCHARGING" -eq 1 ] && [ ! -f $EMPTY_FILE ]; then
	/usr/bin/notify-send "Low Battery" "${BATTERY_LEVEL}% of battery remaining." -u critical -i ~/.local/bin/battery-alert-resized.png -r 9991
	touch $EMPTY_FILE
fi
```

**Explicação do código `batterynotify`:**

- O script verifica o status da bateria usando o comando `acpi`.
- Envia notificações se a bateria estiver acima de 95% (carregada) ou abaixo de 25% (bateria baixa).
- Utiliza arquivos temporários (`/tmp/batteryfull` e `/tmp/batteryempty`) para garantir que a notificação seja enviada apenas uma vez para cada condição.

#### 2. **`chargingnotify`**

O script `chargingnotify` envia uma notificação sempre que o estado da bateria mudar para "carregando" ou "descarregando". 

```bash
#!/bin/sh

# Send a notification when the laptop is plugged in/unplugged
# Add the following to /etc/udev/rules.d/60-power.rules (replace USERNAME with your user)

# ACTION=="change", SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="0", ENV{DISPLAY}=":0", ENV{XAUTHORITY}="/home/USERNAME/.Xauthority" RUN+="/usr/bin/su USERNAME -c '/home/USERNAME/.local/bin/battery-charging discharging'"
# ACTION=="change", SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="1", ENV{DISPLAY}=":0", ENV{XAUTHORITY}="/home/USERNAME/.Xauthority" RUN+="/usr/bin/su USERNAME -c '/home/USERNAME/.local/bin/battery-charging charging'"

export XAUTHORITY=~/.Xauthority
export DISPLAY=:0
export DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/1000/bus"

BATTERY_STATE=$1
BATTERY_LEVEL=$(acpi -b | grep "Battery 0" | grep -P -o '[0-9]+(?=%)')

if [ "$BATTERY_STATE" -eq 1 ]; then
    /usr/bin/notify-send "Charging" "${BATTERY_LEVEL}% of battery charged." -u low -i ~/.local/bin/battery-charging-resized.png -t 5000 -r 9991
elif [ "$BATTERY_STATE" -eq 0 ]; then
    /usr/bin/notify-send "Discharging" "${BATTERY_LEVEL}% of battery remaining." -u low -i ~/.local/bin/battery-alert-resized.png -t 5000 -r 9991
else
    echo "Invalid BATTERY_STATE: $BATTERY_STATE"
fi
```

**Explicação do código `chargingnotify`:**

- Esse script usa o parâmetro de estado de carga para determinar se a bateria está "carregando" ou "descarregando".
- Envia notificações de acordo com o status da bateria. A notificação é ajustada dependendo de qual estado (carregando ou descarregando) o sistema está.

### **Arquivos no Path `~/.local/bin/`**

Os scripts `batterynotify` e `chargingnotify` devem ser colocados no diretório `~/.local/bin/`, que é um diretório comum para armazenar scripts e binários de usuários. Certifique-se de que os arquivos sejam executáveis:

```bash
chmod +x ~/.local/bin/batterynotify
chmod +x ~/.local/bin/chargingnotify
```

### **Configuração do Udev**

Para que o script `chargingnotify` seja executado quando a bateria for conectada ou desconectada, adicione as regras do Udev:

1. Crie o arquivo de regras do Udev:

```bash
sudo nano /etc/udev/rules.d/60-power.rules
```

2. Adicione o seguinte conteúdo, substituindo `USERNAME` pelo seu nome de usuário:

```bash
ACTION=="change", SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="0", ENV{DISPLAY}=":0", ENV{XAUTHORITY}="/home/USERNAME/.Xauthority" RUN+="/usr/bin/su USERNAME -c '/home/USERNAME/.local/bin/chargingnotify 0'"
ACTION=="change", SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="1", ENV{DISPLAY}=":0", ENV{XAUTHORITY}="/home/USERNAME/.Xauthority" RUN+="/usr/bin/su USERNAME -c '/home/USERNAME/.local/bin/chargingnotify 1'"
```

Isso vai garantir que o script seja executado sempre que a bateria for conectada ou desconectada.

---

Agora você tem um sistema que envia notificações quando a bateria atinge certos níveis ou quando é conectada/desconectada, com a utilização de cron e Udev para automação.

<img src="https://static.wikia.nocookie.net/aa9c3777-68fe-4d68-ad7e-7f9374fee7c6/scale-to-width/755" alt="Yokuso" width="1050" height="auto"/>
