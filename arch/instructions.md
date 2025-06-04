## Guia de Instalação Arch Linux Personalizado 🐧

Este guia tem como objetivo fornecer um passo a passo para a instalação do Arch Linux, incluindo dual boot com Windows, kernel LTS, drivers NVIDIA, ambiente gráfico com Hyprland e suas ferramentas, além de um setup completo para desenvolvimento de software.

---

### I. Considerações Iniciais e Preparação

1.  **Backup:** Faça backup de todos os dados importantes do seu sistema Windows e de qualquer outro lugar. A instalação de um novo sistema operacional sempre carrega riscos.
2.  **Download da ISO do Arch Linux:**
    * Baixe a imagem ISO mais recente do Arch Linux em [https://archlinux.org/download/](https://archlinux.org/download/).
    * Verifique a integridade da ISO baixada (opcional, mas recomendado).
3.  **Criação de Mídia Inicializável:**
    * Use uma ferramenta como Rufus, Etcher ou `dd` para criar um pendrive inicializável com a ISO do Arch Linux.
    * Exemplo com `dd` (substitua `/dev/sdX` pelo seu pendrive):
        ```bash
        sudo dd bs=4M if=/caminho/para/archlinux-*.iso of=/dev/sdX status=progress oflag=sync
        ```
4.  **Preparação do Windows para Dual Boot:**
    * **Desabilitar Fast Startup (Inicialização Rápida) no Windows:**
        * Painel de Controle > Opções de Energia > Escolher a função dos botões de energia > Alterar configurações não disponíveis no momento > Desmarque "Ligar inicialização rápida".
    * **Redimensionar Partição do Windows:**
        * Abra o "Gerenciamento de Disco" no Windows (diskmgmt.msc).
        * Clique com o botão direito na partição do Windows (geralmente C:) e selecione "Diminuir Volume".
        * Libere espaço suficiente para a instalação do Arch Linux (pelo menos 50-100GB para o sistema e mais para seus arquivos `/home`). Como seu NVMe é de 1TB, você tem bastante espaço.
5.  **Configurações da BIOS/UEFI:**
    * Reinicie o computador e acesse a BIOS/UEFI (geralmente pressionando Del, F2, F10, F12 ou Esc durante a inicialização).
    * **Desabilitar Secure Boot:** Esta é uma etapa crucial para instalar o Arch Linux.
    * **Definir Boot Order:** Certifique-se de que o pendrive USB seja a primeira opção de boot.
    * **SATA Mode:** Verifique se está configurado para AHCI (geralmente é o padrão).
    * Salve as alterações e saia.

---

### II. Boot e Conexão com a Internet

1.  **Iniciar pela Mídia de Instalação:** O sistema deve iniciar a partir do pendrive do Arch Linux. Você verá um menu, selecione "Arch Linux install medium".
2.  **Conectar à Internet (Wi-Fi):**
    * Assim que o prompt de comando estiver disponível, conecte-se à sua rede Wi-Fi usando `iwctl`. Seu nome de Wi-Fi é `CHIDAS_ROOM_5G`.
    * Identifique sua interface Wi-Fi:
        ```bash
        iwctl device list
        ```
        (Anote o nome do dispositivo, por exemplo, `wlan0`)
    * Inicie o `iwctl` no modo interativo:
        ```bash
        iwctl
        ```
    * Dentro do `iwctl`:
        ```
        station <SEU_DISPOSITIVO_WIFI> scan
        station <SEU_DISPOSITIVO_WIFI> get-networks
        station <SEU_DISPOSITIVO_WIFI> connect "{WIFI_NAME}"
        ```
        (Você será solicitado a inserir a senha do Wi-Fi)
    * Verifique a conexão:
        ```
        station <SEU_DISPOSITIVO_WIFI> show
        exit
        ```
    * Teste a conexão:
        ```bash
        ping archlinux.org
        ```
        (Pressione Ctrl+C para parar)

---

### III. Preparação do Disco e Sistema Base

#### A. Atualizar o Relógio do Sistema
```bash
timedatectl set-ntp true
````

#### B. Definir Layout do Teclado

Seu teclado é Inglês Internacional.

```bash
loadkeys us-acentos
```

#### C. Particionamento do Disco (UEFI com NVMe)

Seu disco é um NVMe de 1TB (`/dev/nvme0n1`). Criaremos partições EFI, Raiz (`/`) e Home (`/home`). Não usaremos partição swap devido aos seus 64GB de RAM.

  * **Partições Sugeridas para 1TB NVMe:**

      * EFI System Partition (ESP): 1GB (para `/boot`)
      * Raiz (`/`): 150GB
      * Home (`/home`): Restante do disco (aproximadamente 849GB para um disco de 1000GB, ou \~873GB para um disco de 1024GB. O sistema mostrará o tamanho exato).

  * Use `cfdisk` para particionar (mais amigável que `fdisk`):

    ```bash
    cfdisk /dev/nvme0n1
    ```

    1.  Selecione `gpt` como tipo de label.
    2.  Crie as seguintes partições:
          * **Partição 1 (EFI):**
              * `New` -\> Size: `1G` -\> `Type`: `EFI System`
          * **Partição 2 (Raiz):**
              * `New` -\> Size: `150G` -\> `Type`: `Linux filesystem` (padrão)
          * **Partição 3 (Home):**
              * `New` -\> Size: (use o restante do espaço) -\> `Type`: `Linux filesystem` (padrão)
    3.  Selecione `Write` para salvar as alterações e depois `Quit`.

  * **Identificadores das Partições (Exemplo):**

      * EFI: `/dev/nvme0n1p1`
      * Raiz: `/dev/nvme0n1p2`
      * Home: `/dev/nvme0n1p3`

#### D. Formatar as Partições

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

#### E. Montar as Partições

```bash
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot # -p para criar diretórios pais se não existirem
mount /dev/nvme0n1p1 /mnt/boot
mkdir -p /mnt/home
mount /dev/nvme0n1p3 /mnt/home
```

#### F. Selecionar Espelhos (Mirrors) para o Pacman

É uma boa prática editar `/etc/pacman.d/mirrorlist` para selecionar os espelhos mais rápidos e próximos. Você pode usar o `reflector` para isso (se não estiver na ISO, pode pular esta etapa por agora ou instalar depois do chroot). Se for editar manualmente:

```bash
nvim /etc/pacman.d/mirrorlist
ou
reflector --country "Brazil" --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
pacman -Syy
```

(Descomente os servidores desejados, por exemplo, os do Brasil. Mova os mais rápidos para o topo).

#### G. Instalar Pacotes Essenciais (Sistema Base e Kernel LTS)

Vamos instalar o kernel LTS (`linux-lts`) em vez do kernel padrão para maior estabilidade.
Também incluiremos `neovim` aqui para que esteja disponível imediatamente.

```bash
pacstrap /mnt base base-devel linux-lts linux-lts-headers linux-firmware neovim git sudo networkmanager bluez bluez-utils ntfs-3g man-db man-pages texinfo
```

  * `base-devel`: Contém ferramentas importantes para compilação.
  * `linux-lts` e `linux-lts-headers`: Kernel LTS e seus headers.
  * `networkmanager`: Para gerenciamento de rede (incluindo Wi-Fi após a instalação).
  * `bluez`, `bluez-utils`: Para Bluetooth.
  * `ntfs-3g`: Para suporte a partições NTFS (Windows).

-----

### IV. Configurar o Sistema

#### A. Gerar o Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Verifique o arquivo gerado para garantir que está correto:

```bash
nvim /mnt/etc/fstab
```

#### B. Chroot para o Novo Sistema

```bash
arch-chroot /mnt
```

Agora todos os comandos serão executados dentro do seu novo sistema Arch.

#### C. Configurar Fuso Horário

```bash
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
hwclock --systohc
```

(Ajuste `America/Sao_Paulo` se sua localização for diferente).

#### D. Configurar Localização (Locale)

1.  Edite `/etc/locale.gen` e descomente as localizações desejadas (ex: `en_US.UTF-8 UTF-8` e `pt_BR.UTF-8 UTF-8`):
    ```bash
    nvim /etc/locale.gen
    ```
2.  Gere os locales:
    ```bash
    locale-gen
    ```
3.  Crie o arquivo `/etc/locale.conf`:
    ```bash
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    ```
    (Ou `pt_BR.UTF-8` se preferir o sistema em português).
4.  Defina o layout do teclado persistente (opcional aqui, pois Hyprland terá sua própria configuração, mas bom para o console):
    ```bash
    echo "KEYMAP=us-acentos" > /etc/vconsole.conf
    ```

#### E. Configurar Nome do Host (Hostname)

```bash
echo "SEU_HOSTNAME_AQUI" > /etc/hostname
```

Adicione entradas correspondentes em `/etc/hosts`:

```bash
nvim /etc/hosts
```

Conteúdo:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   SEU_HOSTNAME_AQUI.localdomain   SEU_HOSTNAME_AQUI
```

#### F. Definir Senha do Root

```bash
passwd
```

#### G. Criar Usuário Pessoal

1.  Crie o usuário:
    ```bash
    useradd -m -G wheel SEU_USUARIO
    ```
    (Substitua `SEU_USUARIO` pelo seu nome de usuário. O grupo `wheel` permite usar `sudo`).
2.  Defina a senha do usuário:
    ```bash
    passwd SEU_USUARIO
    ```
3.  Configure `sudo` para permitir que membros do grupo `wheel` executem comandos como root:
    ```bash
    EDITOR=nvim visudo
    ```
    Descomente a linha: `%wheel ALL=(ALL:ALL) ALL`

#### H. Configurar o Pacman para Pacotes Estáveis

O Arch Linux por padrão já usa repositórios estáveis (`core`, `extra`, `multilib`). Certifique-se de que nenhum repositório `testing` esteja habilitado em `/etc/pacman.conf` (a menos que você saiba o que está fazendo).
Habilite o repositório `multilib` para compatibilidade com aplicações 32-bit (necessário para Steam, Wine, alguns drivers):

```bash
nvim /etc/pacman.conf
```

Descomente as seguintes linhas:

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Depois, atualize o sistema e os repositórios:

```bash
pacman -Syu
```

-----

### V. Instalar Bootloader (GRUB para Dual Boot)

1.  Instale os pacotes necessários para o GRUB e detecção de outros SOs(obs: `amd-ucode` somente se seu processador é um AMD):
    ```bash
    pacman -S grub efibootmgr os-prober amd-ucode
    ```
2.  Monte a partição Windows:
    ```bash
    mkdir -p /mnt/windows_efi
    mount /dev/nvme0n0pX /mnt/windows_efi # Substitua 'X' pelo número correto da partição EFI do Windows
    ```
3.  Instale o GRUB no diretório EFI:
    ```bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH --removable
    ```
    (O `--bootloader-id=ARCH` define o nome da entrada de boot na UEFI).
4.  **Importante para Dual Boot:** Edite `/etc/default/grub` para que o `os-prober` detecte o Windows:
    ```bash
    nvim /etc/default/grub
    ```
    Adicione ou modifique a linha para habilitar o os-prober:
    ```
    GRUB_DISABLE_OS_PROBER=false
    ```
    Para melhorar a experiência com o kernel LTS e o dual boot (opcional, mas recomendado):
    ```
    GRUB_DEFAULT=saved
    GRUB_SAVEDEFAULT=true
    # GRUB_DISABLE_SUBMENU=y # Descomente se não quiser submenus
    ```
5.  Gere o arquivo de configuração do GRUB:
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    Você deverá ver o Arch Linux (com kernel LTS) e o Windows Boot Manager listados.

-----

### VI. Instalar Drivers Gráficos (NVIDIA RTX 4070 Super)

Sua placa é uma RTX 4070 Super, que requer drivers recentes. Usaremos os drivers proprietários da NVIDIA.

1.  **Instalar os Drivers NVIDIA e Utilitários:**

      * Usaremos `nvidia-dkms` que é geralmente mais flexível.

    <!-- end list -->

    ```bash
    pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings
    ```

      * `nvidia-dkms`: Driver principal (versão DKMS).
      * `nvidia-utils`: Utilitários como `nvidia-smi`.
      * `lib32-nvidia-utils`: Versões 32-bit para multilib.
      * `nvidia-settings`: Ferramenta de configuração da NVIDIA.

2.  **Configurar `mkinitcpio`:**

      * Adicione os módulos NVIDIA ao initramfs para garantir que sejam carregados no início do boot.
      * Edite `/etc/mkinitcpio.conf`:
        ```bash
        nvim /etc/mkinitcpio.conf
        ```
      * Localize a linha `MODULES=(...)` e adicione os módulos da NVIDIA:
        ```
        MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
        ```
        (Se `kms` estiver na linha `HOOKS`, remova-o, pois `nvidia_drm.modeset=1` cuidará disso. Se não houver `kms` em `HOOKS`, não se preocupe).
      * Reconstrua o initramfs:
        ```bash
        mkinitcpio -P
        ```

3.  **Configurar Parâmetros do Kernel (GRUB):**

      * Para Wayland e bom funcionamento, o early KMS da NVIDIA é essencial.
      * Edite `/etc/default/grub`:
        ```bash
        nvim /etc/default/grub
        ```
      * Adicione `nvidia-drm.modeset=1` à linha `GRUB_CMDLINE_LINUX_DEFAULT`:
        ```
        GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet nvidia-drm.modeset=1"
        ```
      * Regenere a configuração do GRUB:
        ```bash
        grub-mkconfig -o /boot/grub/grub.cfg
        ```

4.  **(Opcional mas Recomendado) Pacman Hook para NVIDIA DKMS:**

      * Para garantir que o módulo DKMS da NVIDIA seja recompilado automaticamente após atualizações do kernel.
      * Crie o arquivo `/etc/pacman.d/hooks/nvidia-dkms.hook`:
        ```bash
        nvim /etc/pacman.d/hooks/nvidia-dkms.hook
        ```
        Conteúdo:
        ```
        [Trigger]
        Operation=Install
        Operation=Upgrade
        Type=Package
        Target=nvidia-dkms
        Target=linux-lts # Ou o nome do seu pacote de kernel principal

        [Action]
        Description=Rebuild NVIDIA DKMS modules
        When=PostTransaction
        Exec=/usr/bin/dkms autoinstall --no-depmod -k $(cat /usr/lib/modules/$(uname -r)/version) # Adapte se necessário ou use um script mais genérico
        # Uma abordagem mais simples pode ser apenas executar mkinitcpio -P, mas o dkms autoinstall é mais direto para o módulo.
        # Alternativa mais robusta (requer mkinitcpio):
        # Exec=/bin/sh -c 'mkinitcpio -P'
        ```
      * Nota: A criação de hooks pode ser complexa. A principal ação é que após uma atualização do kernel, o `dkms` deve ser acionado. Muitas vezes, o próprio `dkms` já se integra bem. Se você enfrentar problemas com módulos após atualizações do kernel, revisite esta etapa. O hook do `mkinitcpio` já é um bom começo.

-----

### VII. Instalar Ambiente Gráfico (Hyprland e Ferramentas)

Primeiro, vamos instalar algumas dependências e o servidor de exibição.
Obs: o comando `makepkg` deve ser executado em user level, para fazer isso saia da sessão de root com `exit` e depois execute `arch-chroot -u username /mnt`

1.  **Servidor de Áudio (PipeWire):**

    ```bash
    pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber gst-plugin-pipewire
    ```

      * `wireplumber`: Gerenciador de sessão para PipeWire.
      * `gst-plugin-pipewire`: Para aplicações GStreamer.

2.  **Instalar AUR Helper (yay):**
    Muitas das ferramentas de personalização e algumas de desenvolvimento são mais fáceis de instalar via Arch User Repository (AUR). `yay` é um popular AUR helper.

    ```bash
    cd /tmp
    git clone [https://aur.archlinux.org/yay-bin.git](https://aur.archlinux.org/yay-bin.git)
    # Se você não tiver 'base-devel' completo, pode precisar de 'fakeroot' etc.
    # pacman -S --needed base-devel # Garanta que base-devel está completo
    cd yay-bin
    makepkg -si # Como seu usuário normal, não root
    cd ..
    rm -rf yay-bin
    ```

3.  **Instalar Hyprland e Componentes Principais:**

      * **Hyprland (Window Manager Wayland):**
        ```bash
        pacman -S hyprland xdg-desktop-portal-hyprland polkit-kde-agent qt5-wayland qt6-wayland
        ```
          * `xdg-desktop-portal-hyprland`: Para funcionalidades como compartilhamento de tela.
          * `polkit-kde-agent`: Agente Polkit para permissões. (Ou `polkit-gnome` se preferir)
      * **Waybar (Barra de Ferramentas):**
        ```bash
        pacman -S waybar otf-font-awesome ttf-font-awesome # Fontes para ícones
        ```
      * **Ghostty (Terminal):**
        ```bash
        pacman -S ghostty # Já deve estar no repositório [extra]
        # Se não, yay -S ghostty-git
        ```
      * **Shells (Zsh e Fish):**
        ```bash
        pacman -S zsh zsh-completions fish fisher
        ```
          * Para Zsh (mudar shell padrão do seu usuário): `chsh -s /usr/bin/zsh SEU_USUARIO`
          * Para Fish: `chsh -s /usr/bin/fish SEU_USUARIO`
          * Plugin manager para Zsh (Oh My Zsh) - Instale como seu usuário:
            ```bash
            sh -c "$(curl -fsSL [https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh](https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh))" "" --unattended
            ```
            Ou via AUR: `yay -S oh-my-zsh-git`
          * Fisher é um plugin manager para Fish (já instalado).
      * **Hyprlock (Lockscreen):**
        ```bash
        pacman -S hyprlock # Já deve estar no repositório [extra]
        # Se não, yay -S hyprlock-git
        ```
      * **Hyprpaper (Backgrounds):**
        ```bash
        pacman -S hyprpaper
        ```
      * **Swaync (Notificações):**
        ```bash
        pacman -S swaync
        ```
      * **Wofi (App Launcher):**
        ```bash
        pacman -S wofi
        ```
      * **File Explorer (Thunar):**
        ```bash
        pacman -S thunar thunar-volman gvfs # gvfs para funcionalidades como lixeira
        ```
        (Alternativa: Dolphin (KDE) `pacman -S dolphin dolphin-plugins kio-extras`)
      * **Outras Ferramentas e Utilitários:**
          * **Neovim:** Já instalado com `pacstrap`.
          * **Wi-Fi Manager (GUI):** `network-manager-applet` (para bandeja do sistema).
            ```bash
            pacman -S network-manager-applet
            ```
            Habilitar o serviço NetworkManager para iniciar no boot (se ainda não estiver):
            ```bash
            systemctl enable NetworkManager
            ```
          * **Bluetooth Manager (GUI):** `blueman`
            ```bash
            pacman -S blueman
            ```
            Habilitar o serviço Bluetooth:
            ```bash
            systemctl enable bluetooth
            ```
          * **Nerd Fonts (Exemplo: FiraCode Nerd Font):**
            ```bash
            pacman -S ttf-firacode-nerd
            ```
            Para um conjunto mais completo (pode ser grande): `yay -S nerd-fonts-complete`
          * **Utilitários de Fontes:**
            ```bash
            pacman -S fontconfig
            ```
            Atualize o cache de fontes (como usuário normal, após instalar as fontes): `fc-cache -fv`

-----

### VIII. Configuração Inicial do Hyprland e Ferramentas (Pós-Reboot)

Após reiniciar no novo sistema, você fará login no TTY. Para iniciar o Hyprland, você pode usar um Display Manager (como SDDM) ou iniciar manualmente.

**Para iniciar Hyprland manualmente a partir do TTY:**
Digite `Hyprland` no prompt do TTY após o login.
Para que isso funcione, algumas variáveis de ambiente podem ser necessárias. Normalmente, `dbus-launch --exit-with-session Hyprland` é usado se você não estiver usando um gerenciador de login que configure a sessão D-Bus.

Adicione ao seu `~/.bash_profile` (se usar Bash e logar via console), `~/.zprofile` (se usar Zsh), ou `~/.config/fish/config.fish` (se usar Fish) para iniciar Hyprland automaticamente ao logar no TTY1:

```bash
# Para .bash_profile ou .zprofile
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
  exec dbus-launch --exit-with-session Hyprland
fi

# Para config.fish
if status is-login; and test -z "$DISPLAY"; and test (tty) = "/dev/tty1"
  exec dbus-launch --exit-with-session Hyprland
end
```

#### A. Configuração do Hyprland (`~/.config/hypr/hyprland.conf`)

O Hyprland não cria um arquivo de configuração padrão. Você precisará criá-lo.
Copie o exemplo de configuração (como seu usuário normal):

```bash
mkdir -p ~/.config/hypr
cp /usr/share/hyprland/hyprland.conf ~/.config/hypr/hyprland.conf
```

Edite `~/.config/hypr/hyprland.conf` e adicione/modifique:

  * **Troque as variáveis padrão (importante\!):**
    ```ini
    ...
    $terminal = ghostty
    $fileManager = thunar
    ...
    ```

  * **Variáveis de Ambiente NVIDIA (importante\!):**

    ```ini
    # NVIDIA specific environment variables
    env = LIBVA_DRIVER_NAME,nvidia
    env = __GLX_VENDOR_LIBRARY_NAME,nvidia
    env = GBM_BACKEND,nvidia-drm
    env = __VK_LAYER_NV_optimus,NVIDIA_only
    # env = WLR_NO_HARDWARE_CURSORS,1 # Teste se necessário para seu cursor
    # env = NVD_BACKEND,direct # Para libva-nvidia-driver, se usado
    # env = ELECTRON_OZONE_PLATFORM_HINT,auto # Para apps Electron
    ```

  * **Autostart para Ferramentas Essenciais:**

    ```ini
    # Execute applications at launch
    exec-once = waybar
    exec-once = hyprpaper
    exec-once = swaync
    exec-once = /usr/lib/polkit-kde-authentication-agent-1 # Agente Polkit
    exec-once = blueman-applet # Se quiser o applet do Bluetooth
    exec-once = nm-applet --indicator # Applet do NetworkManager
    ```

  * **Configurações Básicas (Teclado, Monitor, etc.):**
    Consulte a [Wiki do Hyprland](https://wiki.hyprland.org/) para configurar monitores, teclado, mouse, binds, etc. Exemplo para teclado (dentro da seção `input`):

    ```ini
    input {
        kb_layout = us
        kb_variant = intl
        # ... outras configurações de input
        follow_mouse = 1 # Foco segue o mouse
        touchpad {
            natural_scroll = no # ou yes
        }
        sensitivity = 0 # -1.0 - 1.0, 0 means no modification.
    }
    ```

#### B. Configuração das Ferramentas do Ecossistema

1.  **Waybar (`~/.config/waybar/config` e `style.css`):**
    Copie a configuração padrão (como seu usuário normal):

    ```bash
    mkdir -p ~/.config/waybar
    cp /etc/xdg/waybar/config ~/.config/waybar/config
    cp /etc/xdg/waybar/style.css ~/.config/waybar/style.css
    ```

    Ajuste o `~/.config/waybar/config` e `~/.config/waybar/style.css` conforme necessário.

2.  **Hyprlock (`~/.config/hypr/hyprlock.conf`):**
    Copie o exemplo de configuração (como seu usuário normal):

    ```bash
    mkdir -p ~/.config/hypr
    cp /usr/share/hyprland/hyprlock.conf.example ~/.config/hypr/hyprlock.conf # O nome do arquivo exemplo pode variar, verifique o pacote
    # Ou se o pacote hyprlock tiver seu próprio exemplo:
    # cp /usr/share/hyprlock/hyprlock.conf.example ~/.config/hypr/hyprlock.conf
    ```

    Se o arquivo de exemplo não for encontrado nesses locais, consulte a documentação do Hyprlock. A configuração é simples.
    Edite `~/.config/hypr/hyprlock.conf` para personalizar.

3.  **Hyprpaper (`~/.config/hypr/hyprpaper.conf`):**
    Crie o arquivo `~/.config/hypr/hyprpaper.conf` (como seu usuário normal).
    Adicione conteúdo como:

    ```ini
    preload = /caminho/para/seu/wallpaper.png
    # preload = /caminho/para/outro/wallpaper.jpg

    wallpaper = eDP-1,/caminho/para/seu/wallpaper.png
    # Para múltiplos monitores, descubra os nomes com `hyprctl monitors`
    # wallpaper = DP-1,/caminho/para/outro/wallpaper.jpg
    # Para preencher em todos os monitores com o mesmo wallpaper:
    # wallpaper = ,/caminho/para/seu/wallpaper.png

    # Para permitir IPC (opcional, para mudar wallpapers dinamicamente)
    ipc = on
    ```

    Substitua `/caminho/para/seu/wallpaper.png` e os nomes dos monitores.

4.  **Swaync (`~/.config/swaync/config.json` e `style.css`):**
    Copie a configuração padrão (como seu usuário normal):

    ```bash
    mkdir -p ~/.config/swaync
    cp /etc/xdg/swaync/config.json ~/.config/swaync/config.json
    cp /etc/xdg/swaync/style.css ~/.config/swaync/style.css
    ```

    Personalize os arquivos copiados.

5.  **Wofi (`~/.config/wofi/config` e `style.css`):**
    Wofi geralmente funciona sem config, mas para personalizar (como seu usuário normal):

    ```bash
    mkdir -p ~/.config/wofi
    # Wofi pode não vir com um arquivo de config padrão em /etc/xdg.
    # Consulte /usr/share/doc/wofi/examples/ ou a documentação online para exemplos.
    # Exemplo de config básico: crie ~/.config/wofi/config
    # show=drun # Para mostrar aplicativos
    # width=40%
    # Crie ~/.config/wofi/style.css para estilizar
    ```

-----

### IX. Sair do Chroot e Reiniciar

1.  Saia do ambiente chroot:
    ```bash
    exit
    ```
2.  Desmonte todas as partições:
    ```bash
    umount -R /mnt
    ```
3.  Reinicie o sistema:
    ```bash
    reboot
    ```
    Remova o pendrive de instalação. Se tudo correu bem, você verá o menu do GRUB com opções para Arch Linux e Windows.

-----

### X. Instalação de Stacks de Desenvolvimento

Após reiniciar e logar no seu novo sistema Arch com Hyprland (como seu usuário normal):

#### A. Git

Já instalado como parte do `base-devel` ou `pacstrap`. Verifique com `git --version`.

#### B. Python

O Python base já está instalado. Para gerenciamento de múltiplas versões e ambientes virtuais:

```bash
pacman -S python-pip python-virtualenv
yay -S pyenv
```

Configure `pyenv` no seu shell (adicione ao `~/.zshrc`, `~/.bashrc` ou `~/.config/fish/config.fish`):

  * Para `~/.zshrc` ou `~/.bashrc`:
    ```bash
    export PYENV_ROOT="$HOME/.pyenv"
    command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
    eval "$(pyenv init -)"
    eval "$(pyenv virtualenv-init -)" # Para pyenv-virtualenv
    ```
  * Para `~/.config/fish/config.fish`:
    ```fish
    set -x PYENV_ROOT "$HOME/.pyenv"
    if not string match -q -- "$PYENV_ROOT/bin" $PATH
        set -x PATH "$PYENV_ROOT/bin" $PATH
    end
    pyenv init - | source
    pyenv virtualenv-init - | source
    ```

Recarregue a configuração do shell (`source ~/.zshrc`, etc.) ou abra um novo terminal.
Instale versões do Python: `pyenv install 3.11.4`, `pyenv global 3.11.4`.

#### C. Node.js com NVM

1.  Instale NVM (Node Version Manager):
    ```bash
    curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh) | bash
    ```
    (Verifique no [repositório NVM](https://github.com/nvm-sh/nvm) a versão mais recente do script).
2.  O script de instalação deve adicionar as linhas necessárias ao seu arquivo de configuração do shell (`~/.bashrc`, `~/.zshrc`). Se não, adicione manualmente:
    ```bash
    export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
    ```
    Para Fish, use um plugin como `jorgebucaran/nvm.fish` via Fisher: `fisher install jorgebucaran/nvm.fish`.
3.  Recarregue a configuração do shell ou abra um novo terminal.
4.  Instale Node.js (ex: LTS):
    ```bash
    nvm install --lts
    nvm use --lts
    nvm alias default lts/ Erbium # ou o nome da LTS atual
    ```

#### D. Vue.js, Next.js, Angular (via npm)

Requer Node.js e npm (instalado com Node.js via NVM).

  * **Vue CLI:**
    ```bash
    npm install -g @vue/cli
    ```
  * **Next.js (para criar projetos):**
    ```bash
    npx create-next-app@latest
    ```
  * **Angular CLI:**
    ```bash
    npm install -g @angular/cli
    ```

#### E. .NET SDK (usando `dotnet-install.sh` da Microsoft)

1.  Baixe o script de instalação:

    ```bash
    mkdir -p ~/dotnet_install_scripts && cd ~/dotnet_install_scripts
    curl -LO [https://dot.net/v1/dotnet-install.sh](https://dot.net/v1/dotnet-install.sh)
    chmod +x dotnet-install.sh
    cd ~ # Voltar para home ou diretório de sua preferência
    ```

2.  Instale as versões desejadas do .NET SDK (exemplo, instalando em `~/.dotnet-sdks/<version>`):

      * **.NET 6 (LTS):**
        ```bash
        mkdir -p ~/.dotnet-sdks/6.0
        ~/dotnet_install_scripts/dotnet-install.sh --channel 6.0 --install-dir ~/.dotnet-sdks/6.0
        ```
      * **.NET 8 (LTS):**
        ```bash
        mkdir -p ~/.dotnet-sdks/8.0
        ~/dotnet_install_scripts/dotnet-install.sh --channel 8.0 --install-dir ~/.dotnet-sdks/8.0
        ```
      * **.NET 9 (Quando disponível, exemplo para STS ou Preview):**
        ```bash
        mkdir -p ~/.dotnet-sdks/9.0
        ~/dotnet_install_scripts/dotnet-install.sh --channel 9.0 --install-dir ~/.dotnet-sdks/9.0
        # Se o canal 9.0 ainda não for final, pode ser:
        # ~/dotnet_install_scripts/dotnet-install.sh --version <versao_preview_especifica> --install-dir ~/.dotnet-sdks/9.0
        ```
        (Verifique os canais e versões exatas disponíveis no site da Microsoft).

3.  Adicione os SDKs ao PATH. Uma forma de gerenciar é ter um diretório principal para o `dotnet` no PATH e usar symlinks para alternar as versões ativas.

      * Crie um diretório para o executável `dotnet` que estará no PATH: `mkdir -p ~/.dotnet/bin`
      * Adicione ao seu `~/.zshrc` ou `~/.bashrc`:
        ```bash
        export PATH="$HOME/.dotnet/bin:$PATH"
        ```
      * Para Fish (`~/.config/fish/config.fish`):
        ```fish
        fish_add_path $HOME/.dotnet/bin
        ```
      * Crie um symlink do executável `dotnet` da versão desejada para `~/.dotnet/bin/dotnet`.
        Exemplo para ativar .NET 8:
        ```bash
        ln -sfn ~/.dotnet-sdks/8.0/dotnet ~/.dotnet/bin/dotnet
        ```
        Para mudar para .NET 6:
        ```bash
        ln -sfn ~/.dotnet-sdks/6.0/dotnet ~/.dotnet/bin/dotnet
        ```

    Recarregue o shell. Verifique com `dotnet --version`.

#### F. Java (Amazon Corretto 17 e 21 via AUR)

```bash
yay -S amazon-corretto-17 amazon-corretto-21
```

Verifique as versões instaladas e defina a padrão:

```bash
archlinux-java status
sudo archlinux-java set java-17-amazon-corretto # O nome pode variar, use o que `archlinux-java status` mostrar
# Ou sudo archlinux-java set java-21-amazon-corretto
```

#### G. Rust (via rustup)

```bash
pacman -S rustup
```

Configure o toolchain padrão (stable) e adicione componentes úteis:

```bash
rustup default stable
rustup component add rust-src rust-analyzer # rust-analyzer é o LSP preferido
```

O `rustup` deve configurar o PATH automaticamente. Se `~/.cargo/bin` não estiver no seu PATH, adicione-o:

  * Para `~/.zshrc` ou `~/.bashrc`: `export PATH="$HOME/.cargo/bin:$PATH"`
  * Para `~/.config/fish/config.fish`: `fish_add_path $HOME/.cargo/bin`

#### H. Go

```bash
pacman -S go
```

Configure seu ambiente Go (adicione ao `~/.zshrc`, `~/.bashrc` ou `~/.config/fish/config.fish`):

  * Para `~/.zshrc` ou `~/.bashrc`:
    ```bash
    export GOPATH="$HOME/go" # Diretório para seus projetos Go (legado, mas alguns ainda usam)
    export GOBIN="$HOME/go/bin" # Onde 'go install' coloca binários
    export PATH="$GOBIN:$PATH"
    export PATH="$PATH:/usr/local/go/bin" # Se go foi instalado em /usr/local/go
    ```
  * Para `~/.config/fish/config.fish`:
    ```fish
    set -x GOPATH "$HOME/go"
    set -x GOBIN "$GOPATH/bin"
    fish_add_path $GOBIN
    fish_add_path /usr/local/go/bin # Se necessário
    ```

Crie os diretórios: `mkdir -p ~/go/bin`

#### I. Swift

Use o AUR para instalar o Swift. `swift-bin` usa binários pré-compilados.

```bash
yay -S swift-bin # Recomendado para uma instalação mais rápida
yay -S sourcekit-lsp # Para autocompletion e IDE features
```

O Swift geralmente é instalado em `/opt/swift/usr/bin` ou similar. Adicione ao PATH:

  * Para `~/.zshrc` ou `~/.bashrc`: `export PATH="/opt/swift/usr/bin:$PATH"` (verifique o caminho exato)
  * Para `~/.config/fish/config.fish`: `fish_add_path /opt/swift/usr/bin`

-----

### XI. Pós-Instalação e Dicas Adicionais

  * **AUR Helper (Yay):** Use `yay -S <nome_do_pacote>` para instalar pacotes do AUR e `yay -Syu` para atualizar tudo (repositórios oficiais e AUR).
  * **Personalização:** Explore os arquivos de configuração (`dotfiles`) para Hyprland, Waybar, Wofi, Ghostty, Zsh/Fish, Neovim para personalizar seu ambiente.
  * **Firewall:** Considere configurar um firewall (ex: `ufw`):
    ```bash
    pacman -S ufw
    sudo ufw enable
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    # Permita serviços específicos se necessário, ex: sudo ufw allow ssh
    sudo systemctl enable ufw
    sudo systemctl start ufw
    ```
  * **Documentação:** A [Arch Wiki](https://wiki.archlinux.org/) é sua melhor amiga. Consulte-a sempre\!

-----
