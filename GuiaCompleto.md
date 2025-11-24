# Guia Completo: Integração Rclone + Google Drive com Systemd



Este guia descreve o processo passo a passo para instalar o Rclone, criar credenciais de API personalizadas no Google Cloud para melhor performance e configurar a montagem automática do Google Drive no Linux utilizando o `systemd`.



## Índice

1. [Obtendo Credenciais Google (Client ID e Secret)](#1-obtendo-credenciais-google-client-id-e-secret)
2. [Instalação do Rclone](#2-instalação-do-rclone)
3. [Configuração e Conexão](#3-configuração-e-conexão)
4. [Configuração do Serviço Automático (Systemd)](#4-configuração-do-serviço-automático-systemd)
5. [Comandos de Gerenciamento](#5-comandos-de-gerenciamento)

------

## 1. Obtendo Credenciais Google (Client ID e Secret)

O uso de credenciais próprias evita que você compartilhe limites de tráfego da API pública do Rclone, garantindo maior estabilidade e velocidade.

### Passo 1: Criar um Projeto

1. Acesse o [Google Cloud Console](https://console.cloud.google.com/).
2. Crie um **Novo Projeto** e dê um nome (ex: `Rclone-Drive-Integration`).
3. Selecione o projeto criado.

### Passo 2: Ativar a API

1. No menu lateral, vá em **APIs e serviços > Biblioteca**.
2. Pesquise por `Google Drive API`.
3. Clique em **Ativar**.

### Passo 3: Tela de Consentimento OAuth

1. Vá em **APIs e serviços > Tela de permissão OAuth**.
2. Em "User Type", selecione **Externo** e clique em **Criar**.
3. Preencha:
   - **Nome do app:** `Rclone`
   - **E-mail:** Seu e-mail de suporte e desenvolvedor.
4. Clique em **Salvar e continuar** (pode pular as etapas de "Escopos").
5. Em **Usuários de teste**, adicione o e-mail da conta Google que você irá conectar.

> **IMPORTANTE:** Na tela de resumo ou painel OAuth, clique no botão **Publicar Aplicativo** (Publish App). Isso evita que seu token expire a cada 7 dias. O Google avisará que o app não foi verificado, o que é normal para uso pessoal.

### Passo 4: Criar Credenciais

1. Vá em **Credenciais > + CRIAR CREDENCIAIS > ID do cliente OAuth**.
2. **Tipo de aplicativo:** App para computador (Desktop app).
3. **Nome:** `Rclone Client`.
4. Ao criar, copie e salve o **ID de cliente** e a **Chave secreta**.

------

## 2. Instalação do Rclone

Instale a versão mais recente via script oficial:

Bash

```bash
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```

------

## 3. Configuração e Conexão

### Criar diretório de montagem

Crie a pasta onde seus arquivos aparecerão. Substitua `[NOME_DO_REMOTE]` pelo nome que deseja dar à conexão (ex: `pessoal`, `trabalho`).

Bash

```bash
mkdir -p ~/GoogleDrive/[NOME_DO_REMOTE]
```

### Configurar o Rclone

Inicie o assistente:

Bash

```bash
rclone config
```

**Siga o fluxo interativo:**

1. **n** (New remote).
2. **name>** Digite o nome desejado (ex: `[NOME_DO_REMOTE]`).
3. **Storage>** Digite `drive`.
4. **client_id>** Cole seu `ID de Cliente` obtido no Passo 1.
5. **client_secret>** Cole sua `Chave Secreta` obtida no Passo 1.
6. **scope>** Digite `1` (Acesso total).
7. **service_account_file>** Deixe em branco (Enter).
8. **Edit advanced config?** Digite `n`.
9. **Use web browser...?** Digite `y`. O navegador abrirá para você fazer login e autorizar (se aparecer aviso de segurança, clique em *Avançado > Acessar Rclone*).
10. **Shared Drive?** Digite `n` (exceto se for usar Drives de Equipe).
11. **Confirmar:** Digite `y` e depois `q` para sair.

------

## 4. Configuração do Serviço Automático (Systemd)

Para que o Google Drive monte automaticamente ao ligar o PC, criaremos um serviço de usuário.

### Criar o arquivo de serviço

Substitua `[NOME_DO_REMOTE]` pelo nome exato que você criou no `rclone config`.

Bash

```bash
nano ~/.config/systemd/user/rclone-gdrive-[NOME_DO_REMOTE].service
```

### Conteúdo do Arquivo

Copie e cole o conteúdo abaixo, alterando onde estiver indicado:

Ini, TOML

```ini
[Unit]
Description=Montar Google Drive (rclone - [NOME_DO_REMOTE])
AssertPathIsDirectory=%h/GoogleDrive/[NOME_DO_REMOTE]
After=network-online.target

[Service]
Type=simple
# Ajuste os parâmetros de cache/buffer conforme sua memória RAM disponível
ExecStart=/usr/bin/rclone mount [NOME_DO_REMOTE]: %h/GoogleDrive/[NOME_DO_REMOTE] \
 --vfs-cache-mode full \
 --buffer-size 256M \
 --dir-cache-time 24h \
 --attr-timeout 24h

ExecStop=/bin/fusermount -u %h/GoogleDrive/[NOME_DO_REMOTE]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

> **Nota:** O `%h` no script representa automaticamente o diretório home do usuário (`/home/seu_usuario`).

### Ativar o serviço

Bash

```bash
# Recarregar o daemon do systemd
systemctl --user daemon-reload

# Habilitar para iniciar com o sistema
systemctl --user enable rclone-gdrive-[NOME_DO_REMOTE].service

# Iniciar agora
systemctl --user start rclone-gdrive-[NOME_DO_REMOTE].service
```

------

## 5. Comandos de Gerenciamento

Substitua `[NOME_DO_REMOTE]` pelo nome da sua conexão.

**Verificar status (logs de erro/sucesso):**

Bash

```bash
systemctl --user status rclone-gdrive-[NOME_DO_REMOTE].service
```

**Parar a montagem:**

Bash

```bash
systemctl --user stop rclone-gdrive-[NOME_DO_REMOTE].service
```

**Reiniciar o serviço:**

Bash

```bash
systemctl --user restart rclone-gdrive-[NOME_DO_REMOTE].service
```

Reconectar Token Expirado:

Se o token expirar e o serviço falhar, execute:

Bash

```bash
rclone config reconnect [NOME_DO_REMOTE]:
```

E depois reinicie o serviço.