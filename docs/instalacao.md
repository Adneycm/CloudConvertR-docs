# Instalação

Nesta página, você encontrará instruções detalhadas sobre como realizar a instalação da infraestrutura e entender o funcionamento do código.

## Pré-Requisitos

Antes de prosseguir, certifique-se de que você tenha os seguintes pré-requisitos:

* Uma conta na AWS

* Possuir o terraform instalado na sua máquina

* Ter o python instalado na sua máquina

Para verificar se o terraform está corretamente instalado basta rodar o seguinte comando no seu terminal (Caso esteja utilizando o Windows, rode o comando no powershell):

```bash
$ terraform
```

Se o Terraform estiver corretamente instalado, você verá um menu de opções como resposta, conforme mostrado abaixo:

```bash
$ terraform

Usage: terraform [global options] <subcommand> [args]

The available commands for execution are listed below.
The primary workflow commands are given first, followed by
less common or more advanced commands.

Main commands:
  init          Prepare your working directory for other commands
  validate      Check whether the configuration is valid
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure
...
```

Se a resposta não for igual à mostrada acima, verifique o erro e corrija-o antes de prosseguir.

## Download do projeto

Clone o repositório usando o comando abaixo no seu terminal:

```bash
$ git clone https://github.com/Adneycm/CloudConvertR.git
```

A estrutura do projeto está organizada da seguinte forma:

```
CloudConvertR/
├── terraform/
│   ├── python/
│   │   ├── python.zip
│   │   └── lambda-CloudConvertR.py
│   └── main.tf
├─ README.md
└─ .gitignore  
```

