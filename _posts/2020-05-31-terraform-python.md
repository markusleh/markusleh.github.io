---
layout: post
title:  Deploying Python functions in Azure using Terraform
category: howto
picture: /assets/img/tpython.png
description: How to deploy cloud-native python functions using Terraform
excerpt: In this post, I am going to explore using Terraform to deploy a Python 3.8 application running in Linux as Azure Function app.
---

There are several ways to automate deployment of Function apps in Azure. You can:
1. Use Azure cli and run `func azure functionapp publish <name>`
2. [Use](https://docs.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package) `WEBSITE_RUN_FROM_PACKAGE`
3. [Use CI](https://docs.microsoft.com/en-us/azure/azure-functions/functions-how-to-azure-devops?tabs=csharp) [pipelines](https://docs.microsoft.com/en-us/azure/azure-functions/functions-how-to-github-actions?tabs=javascript)
4. Perform a ZIP deployment

In this post, I am going to explore using method number 4 to deploy a Python 3.8 application running in Linux as Azure Function app. [Some](https://markheath.net/post/run-from-package) people had successfully been able to use the `WEBSITE_RUN_FROM_PACKAGE` method in other languages and frameworks, but after several hours of banging my head against the wall as there are no error logs available anywhere when things don't work, I switched to the ZIP deployment method as it seems to be more documented and I can at least try it locally before automating the same work using Terraform.

## 1. Prepare your function project for ZIP deployment
This part assumes that you have already setup a local development environment as explained [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=macos%2Cpython%2Cbash#start).

I am going to refer to `FuncDir` as the main function directory where the functions and configurations are loaded. The folder structure is something like as follows:

```
FuncDir
├── Function1
│   ├── __init__.py
│   ├── __pycache__
│   │   └── __init__.cpython-38.pyc
│   └── function.json
├── AnotherFunction
│   ├── __init__.py
│   ├── __pycache__
│   │   └── __init__.cpython-38.pyc
│   └── function.json
├── .vscode
│   └── extensions.json
├── .python_packages
│   └── lib
│       └── site-packages
│           ├── ...
├── .funcingnore
├── .gitignore
├── host.json
├── local.settings.json
└── requirements.txt
```

You should add `.funcignore` file so that your local settings (and secrets) are not provisioned to the cloud:
```
local.settings.json
```

## 2. Setup Terraform file
Variables
```
variable "zip_path" {
  # Change as needed
  default = "/Users/markus/Projects/func.zip"
}
variable "func_path" {
  # Change as needed
  default = "/Users/markus/Projects/FuncDir"
}
variable "location" {
  # Change as needed
  default = "North Europe"
}
```

`main.tf`

```
provider "azurerm" {
  features {}
  version = "=2.12.0"
}
provider "archive" {}

data "archive_file" "init" {
  type        = "zip"
  source_dir  = var.func_path
  output_path = var.zip_path
}

resource "azurerm_resource_group" "rg1" {
  name     = "rg1"
  location = "North Europe"
}
resource "random_string" "storage_name" {
  length  = 16
  special = false
  upper   = false
}
resource "random_string" "function_name" {
  length  = 16
  special = false
  upper   = false
}
resource "random_string" "app_service_plan_name" {
  length  = 16
  special = false
}
resource "random_string" "app_name" {
  length  = 16
  special = false
}

resource "azurerm_storage_container" "storage_container" {
  name                  = "func"
  storage_account_name  = azurerm_storage_account.storage.name
  container_access_type = "private"
}

resource "azurerm_storage_blob" "storage_blob" {
  name                   = "azure.zip"
  storage_account_name   = azurerm_storage_account.storage.name
  storage_container_name = azurerm_storage_container.storage_container.name
  type                   = "Block"
  source                 = var.zip_path
}

data "azurerm_storage_account_sas" "storage_sas" {
  connection_string = azurerm_storage_account.storage.primary_connection_string
  https_only        = false
  resource_types {
    service   = false
    container = false
    object    = true
  }
  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }
  start  = "20200529"
  expiry = "20300529"
  permissions {
    read    = true
    write   = false
    delete  = false
    list    = false
    add     = false
    create  = false
    update  = false
    process = false
  }
}
resource "azurerm_app_service_plan" "plan" {
  name                = random_string.app_service_plan_name.result
  location            = azurerm_resource_group.rg1.location
  resource_group_name = azurerm_resource_group.rg1.name
  kind                = "functionapp"
  reserved            = true
  sku {
    tier = "Dynamic"
    size = "Y1"
  }
}
resource "azurerm_function_app" "function" {
  name                      = random_string.storage_name.result
  location                  = azurerm_resource_group.rg1.location
  resource_group_name       = azurerm_resource_group.rg1.name
  app_service_plan_id       = azurerm_app_service_plan.plan.id
  storage_connection_string = azurerm_storage_account.storage.primary_connection_string
  os_type                   = "linux"
  version                   = "~3"
  app_settings = {
    "FUNCTIONS_WORKER_RUNTIME" = "python"
    "FUNCTION_APP_EDIT_MODE"   = "readonly"
    "FUNCTIONS_EXTENSION_VERSION" : "~3",
    "https_only"               = true,
  }
  provisioner "local-exec" {
    command = "az webapp deployment source config-zip --resource-group ${azurerm_resource_group.rg1.name} --name ${random_string.storage_name.result} --src ${var.zip_path} --settings SCM_DO_BUILD_DURING_DEPLOYMENT=true"
  }
}
```

Couple of things to note here:

`azurerm_app_service_plan`and `reserved=true`: There is some bug in the current version of Azure provider whereby it does not check that the reserved parameter is set to true when creating a Linux app. Not setting this causes the function to be eventually deployed as a Windows application. I have submitted a [few](https://github.com/terraform-providers/terraform-provider-azurerm/pull/7146) [pull](https://github.com/terraform-providers/terraform-provider-azurerm/commit/502154ad7926ab8fdd01f9adf9f0e261565a5e4e) requests to try to fix this and get this bug at least documented.

`provisioner "local-exec"`: there is no supported method currently to perform the ZIP deployment using Terraform Azure provider. It should be noted that provisioners only run after the resource is **initially** created. This cannot therefore be used to update you application code.


### Future work

- Terraform Azure provider or Azure itself should have a nice native way to deploy a function app directly from a directory.
- Microsoft should more clearly document what are the requirements to run `WEBSITE_RUN_FROM_PACKAGE` and logs when the deployment fails. There is currently only a cryptic error message when navigating to the function --> App files.
- Terraform should fix couple of things that prevent Function app from generating correctly when using Linux app.
